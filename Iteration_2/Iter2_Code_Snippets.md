# Iterazione 2 Codice rilevante

# 1. Livello dati — Repository Pattern

> Novità architetturale centrale dell'iterazione. In Iter. 1 i componenti chiamavano `supabase-js` direttamente; ora dipendono dalle interfacce in `types.ts`. Implementa Dependency Inversion + Information Hiding.

## `lib/repositories/types.ts`
Contratti "forniti" del livello dati (ball-and-socket del component diagram). Contiene tutte le interfacce; per l'Iter. 2 sono centrali `IGoalRepository` e `IGoalTemplateRepository`.

```ts
import type {
  Profile,
  Goal,
  GoalStatus,
  MatchResultRow,
  GoalTemplate,
  ApprovalStatus,
} from '../database.types';

// Shared types 
export type RepoResult<T> =
  | { data: T; error: null }
  | { data: null; error: RepoError };

export interface RepoError {
  message: string;
  /** Code from the underlying driver (Postgres / PostgREST), if any. */
  code?: string;
}

// Goal 
export interface IGoalRepository {
  /** All goals of a single student, ordered by `sort_order`. */
  listByStudent(studentId: string): Promise<RepoResult<Goal[]>>;

  /** Lightweight count + created_at snapshot used for club-wide stats. */
  listAllForStats(): Promise<RepoResult<Pick<Goal, 'id' | 'created_at'>[]>>;

  create(input: {
    studentId: string;
    createdBy: string;
    data: Partial<Goal>;
  }): Promise<RepoResult<Goal>>;

  update(id: string, patch: Partial<Goal>): Promise<RepoResult<Goal>>;

  /**
   * Transition a goal to a new status. Encapsulates the side effects:
   *   completed → progress = 100, completed_at = now()
   *   planned   → progress = 0,   completed_at = null
   */
  changeStatus(id: string, status: GoalStatus): Promise<RepoResult<Goal>>;

  /** Used by the kanban progress slider — fire-and-forget patch. */
  setProgress(id: string, progress: number): Promise<RepoResult<void>>;

  delete(id: string): Promise<RepoResult<void>>;
}

// Goal Template (catalog)
export interface IGoalTemplateRepository {
  list(): Promise<RepoResult<GoalTemplate[]>>;

  create(input: {
    createdBy: string;
    data: Partial<GoalTemplate>;
  }): Promise<RepoResult<GoalTemplate>>;

  update(id: string, patch: Partial<GoalTemplate>): Promise<RepoResult<GoalTemplate>>;

  delete(id: string): Promise<RepoResult<void>>;
}

// Altre interfacce dello stesso pattern (fuori scope Iter.2)
// export interface IMatchRepository { … }   // agonismo
// export interface IProfileRepository { … } // profili/approval (Iter.1)
```

## `lib/repositories/errors.ts`  
Helper dell'envelope `RepoResult` e mappatura dell'errore PostgREST.

```ts
import type { PostgrestError } from '@supabase/supabase-js';
import type { RepoError, RepoResult } from './types';

export function toRepoError(err: PostgrestError | Error | unknown): RepoError {
  if (err && typeof err === 'object' && 'message' in err) {
    const e = err as PostgrestError;
    return { message: e.message ?? 'Unknown error', code: e.code };
  }
  return { message: 'Unknown error' };
}

export function ok<T>(data: T): RepoResult<T> {
  return { data, error: null };
}

export function fail<T>(err: unknown): RepoResult<T> {
  return { data: null, error: toRepoError(err) };
}
```

## `lib/repositories/index.ts`
Entry point unico: istanzia i repository legati al client browser.

```ts
import { supabase } from '../supabase';
import { SupabaseGoalRepository } from './goal.repository';
import { SupabaseMatchRepository } from './match.repository';
import { SupabaseProfileRepository } from './profile.repository';
import { SupabaseGoalTemplateRepository } from './goal-template.repository';

export type {
  IGoalRepository,
  IMatchRepository,
  IProfileRepository,
  IGoalTemplateRepository,
  RepoResult,
  RepoError,
} from './types';

export { SupabaseGoalRepository } from './goal.repository';
export { SupabaseMatchRepository } from './match.repository';
export { SupabaseProfileRepository } from './profile.repository';
export { SupabaseGoalTemplateRepository } from './goal-template.repository';

//Default browser-bound instances 
export const goalRepo = new SupabaseGoalRepository(supabase);
export const matchRepo = new SupabaseMatchRepository(supabase);
export const profileRepo = new SupabaseProfileRepository(supabase);
export const templateRepo = new SupabaseGoalTemplateRepository(supabase);
```

## `lib/repositories/goal.repository.ts`
Implementazione Supabase di `IGoalRepository`. Da notare `changeStatus` (side-effect centralizzati) e `setProgress` (patch fire-and-forget).

```ts
import type { SupabaseClient } from '@supabase/supabase-js';
import type { Database, Goal, GoalStatus } from '../database.types';
import type { IGoalRepository, RepoResult } from './types';
import { ok, fail } from './errors';

type Client = SupabaseClient<Database>;

export class SupabaseGoalRepository implements IGoalRepository {
  constructor(private readonly client: Client) {}

  async listByStudent(studentId: string): Promise<RepoResult<Goal[]>> {
    const { data, error } = await this.client
      .from('goals')
      .select('*')
      .eq('student_id', studentId)
      .order('sort_order');
    if (error) return fail(error);
    return ok((data ?? []) as Goal[]);
  }

  async listAllForStats(): Promise<RepoResult<Pick<Goal, 'id' | 'created_at'>[]>> {
    const { data, error } = await this.client.from('goals').select('id, created_at');
    if (error) return fail(error);
    return ok((data ?? []) as Pick<Goal, 'id' | 'created_at'>[]);
  }

  async create({
    studentId,
    createdBy,
    data,
  }: {
    studentId: string;
    createdBy: string;
    data: Partial<Goal>;
  }): Promise<RepoResult<Goal>> {
    const row = {
      ...data,
      student_id: studentId,
      created_by: createdBy,
    } as Database['public']['Tables']['goals']['Insert'];
    const { data: inserted, error } = await this.client
      .from('goals')
      .insert(row)
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(inserted as Goal);
  }

  async update(id: string, patch: Partial<Goal>): Promise<RepoResult<Goal>> {
    const { data, error } = await this.client
      .from('goals')
      .update({ ...patch, updated_at: new Date().toISOString() } as Database['public']['Tables']['goals']['Update'])
      .eq('id', id)
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(data as Goal);
  }

  async changeStatus(id: string, status: GoalStatus): Promise<RepoResult<Goal>> {
    // Status transitions carry implicit side effects, encoded here (single
    // source of truth) instead of being repeated in every page.
    const patch: Partial<Goal> = {
      status,
      updated_at: new Date().toISOString(),
    };
    if (status === 'completed') {
      patch.progress = 100;
      patch.completed_at = new Date().toISOString();
    }
    if (status === 'planned') {
      patch.progress = 0;
      patch.completed_at = null;
    }
    const { data, error } = await this.client
      .from('goals')
      .update(patch as Database['public']['Tables']['goals']['Update'])
      .eq('id', id)
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(data as Goal);
  }

  async setProgress(id: string, progress: number): Promise<RepoResult<void>> {
    const patch = { progress, updated_at: new Date().toISOString() } as Database['public']['Tables']['goals']['Update'];
    const { error } = await this.client.from('goals').update(patch).eq('id', id);
    if (error) return fail(error);
    return ok(undefined);
  }

  async delete(id: string): Promise<RepoResult<void>> {
    const { error } = await this.client.from('goals').delete().eq('id', id);
    if (error) return fail(error);
    return ok(undefined);
  }
}
```

## `lib/repositories/goal-template.repository.ts`
Implementazione Supabase di `IGoalTemplateRepository` (catalogo lato maestro).

```ts
import type { SupabaseClient } from '@supabase/supabase-js';
import type { Database, GoalTemplate } from '../database.types';
import type { IGoalTemplateRepository, RepoResult } from './types';
import { ok, fail } from './errors';

type Client = SupabaseClient<Database>;

export class SupabaseGoalTemplateRepository implements IGoalTemplateRepository {
  constructor(private readonly client: Client) {}

  async list(): Promise<RepoResult<GoalTemplate[]>> {
    const { data, error } = await this.client
      .from('goal_templates')
      .select('*')
      .order('level')
      .order('category')
      .order('sort_order')
      .order('title');
    if (error) return fail(error);
    return ok((data ?? []) as GoalTemplate[]);
  }

  async create({
    createdBy,
    data,
  }: {
    createdBy: string;
    data: Partial<GoalTemplate>;
  }): Promise<RepoResult<GoalTemplate>> {
    const row = { ...data, created_by: createdBy };
    const { data: inserted, error } = await this.client
      .from('goal_templates')
      .insert([row] as Database['public']['Tables']['goal_templates']['Insert'][])
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(inserted as GoalTemplate);
  }

  async update(id: string, patch: Partial<GoalTemplate>): Promise<RepoResult<GoalTemplate>> {
    const { data, error } = await this.client
      .from('goal_templates')
      .update({ ...patch, updated_at: new Date().toISOString() })
      .eq('id', id)
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(data as GoalTemplate);
  }

  async delete(id: string): Promise<RepoResult<void>> {
    const { error } = await this.client.from('goal_templates').delete().eq('id', id);
    if (error) return fail(error);
    return ok(undefined);
  }
}
```

---

# 2. Modello di dominio  `

## `lib/database.types.ts` (frammenti)  
Enum ed entità rilevanti per gli obiettivi e i template, più la forma `Database` per le due tabelle.

```ts
//  Domain enums / literal unions 
export type GoalCategory = 'tecnica' | 'tattica' | 'fisico' | 'mente' | 'agonismo';
export type GoalStatus = 'planned' | 'in_progress' | 'completed';
export type PlayerLevel = 'DELFINO' | 'CERBIATTO' | 'COCCODRILLO';
// … (UserRole, SurfaceType, MatchResult, ApprovalStatus invariati/fuori scope)

//  Row types 
export interface Goal {
  id: string;
  student_id: string;
  category: GoalCategory;
  title: string;
  description: string | null;
  status: GoalStatus;
  progress: number;
  deadline: string | null;
  sort_order: number;
  created_by: string | null;
  coach_notes: string | null;
  created_at: string;
  updated_at: string;
  completed_at: string | null;
}

export interface GoalTemplate {
  id: string;
  category: GoalCategory;
  level: PlayerLevel;
  title: string;
  description: string | null;
  created_by: string | null;
  created_at: string;
  updated_at: string;
  sort_order: number;
}

// Insert helpers (colonne con default DB → opzionali)
type Optionalize<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

type GoalInsert = Optionalize<
  Goal,
  'id' | 'description' | 'progress' | 'deadline' | 'sort_order'
  | 'created_by' | 'coach_notes' | 'created_at' | 'updated_at' | 'completed_at'
>;

type GoalTemplateInsert = Optionalize<
  GoalTemplate,
  'id' | 'description' | 'created_by' | 'created_at' | 'updated_at' | 'sort_order'
>;

// Database<T> (estratto: solo goals + goal_templates) 
export interface Database {
  public: {
    Tables: {
      // …
      goals: {
        Row: Goal;
        Insert: GoalInsert;
        Update: Partial<Goal>;
      };
      goal_templates: {
        Row: GoalTemplate;
        Insert: GoalTemplateInsert;
        Update: Partial<GoalTemplate>;
      };
    };
    Enums: {
      // …
      goal_category: GoalCategory;
      goal_status: GoalStatus;
      player_level: PlayerLevel;
    };
  };
}
```
## `lib/constants.ts` (frammenti)  
Configurazione di categorie e stati usata da kanban, form e catalogo.

```ts
import { GoalCategory, GoalStatus } from './database.types';

export const CATEGORY_CONFIG: Record<GoalCategory, { label: string; icon: string; color: string; bg: string }> = {
  tecnica:  { label: 'Tecnica',        icon: 'racquet',  color: '#C41E3A', bg: '#F8E8EB' },
  tattica:  { label: 'Tattica',        icon: 'brain',    color: '#1B3A5C', bg: '#E8EDF2' },
  fisico:   { label: 'Fisico/Motori',  icon: 'dumbbell', color: '#2E7D32', bg: '#E8F5E9' },
  mente:    { label: 'Mente',          icon: 'sparkles', color: '#7B1FA2', bg: '#F3E5F5' },
  agonismo: { label: 'Agonismo',       icon: 'trophy',   color: '#E65100', bg: '#FFF3E0' },
};

export const STATUS_CONFIG: Record<GoalStatus, { label: string; labelIt: string }> = {
  planned:     { label: 'Planned',     labelIt: 'In Programma' },
  in_progress: { label: 'In Progress', labelIt: 'In Corso' },
  completed:   { label: 'Completed',   labelIt: 'Conclusi' },
};

export const STATUS_COLUMNS: GoalStatus[] = ['planned', 'in_progress', 'completed'];

export const LEVELS = ['DELFINO', 'CERBIATTO', 'COCCODRILLO'] as const;

// … (SURFACE_LABELS, RESULT_LABELS, getDisplayRanking, getAge, getAgeCategory: fuori scope obiettivi)
```

---

# 3. Componenti UI — obiettivi / kanban / catalogo 


## `components/KanbanBoard.tsx`  
Board a 3 colonne. Logica chiave: partizionamento per stato, drag&drop (desktop), tab + swipe (mobile), transizioni avanti/indietro tramite `onStatusChange`.

```tsx
'use client';

import { useState, useRef, useEffect } from 'react';
import { ClipboardList, Zap, CircleCheck, type LucideIcon } from 'lucide-react';
import type { Goal, GoalStatus } from '@/lib/database.types';
import { CATEGORY_CONFIG, STATUS_CONFIG, STATUS_COLUMNS } from '@/lib/constants';
import { useIsMobile } from '@/lib/hooks';
import { Badge, ProgressBar, ConfirmDialog, Select } from './UI';
import { CategoryIcon } from './CategoryIcon';

//GoalCard 
interface GoalCardProps {
  goal: Goal;
  isCoach: boolean;
  isMobile: boolean;
  onEdit: (goal: Goal) => void;
  onDelete: (id: string) => void;
  onStatusChange: (id: string, status: GoalStatus) => void;
  onProgressChange?: (id: string, progress: number) => void;
  onDragStart?: (e: React.DragEvent, id: string) => void;
}

function GoalCard({ goal, isCoach, isMobile, onEdit, onDelete, onStatusChange, onProgressChange, onDragStart }: GoalCardProps) {
  const [confirmDelete, setConfirmDelete] = useState(false);
  const cat = CATEGORY_CONFIG[goal.category];
  const nowRef = useRef(Date.now());
  const daysLeft = goal.deadline ? Math.ceil((new Date(goal.deadline).getTime() - nowRef.current) / (1000 * 60 * 60 * 24)) : null;
  const isCompleted = goal.status === 'completed';

  const currentIndex = STATUS_COLUMNS.indexOf(goal.status);
  const nextStatus = currentIndex < STATUS_COLUMNS.length - 1 ? STATUS_COLUMNS[currentIndex + 1] : null;
  const prevStatus = currentIndex > 0 ? STATUS_COLUMNS[currentIndex - 1] : null;

  // … card "completed" compatta (markup) …

  // Normal card (planned / in_progress) 
  return (
    <>
      <div
        draggable={!isMobile}
        onDragStart={(e) => !isMobile && onDragStart?.(e, goal.id)}
        className={`goal-card … ${isMobile ? '' : 'hover:shadow-md … cursor-grab active:cursor-grabbing'}`}
        style={{ '--cat-color': cat.color } as React.CSSProperties}
      >
        {/* Badge categoria + deadline */}
        {/* Titolo / descrizione */}

        {/* Progress bar (solo in_progress) */}
        {goal.status === 'in_progress' && (
          <div className="mb-3 pl-2">
            <ProgressBar value={goal.progress} color={cat.color} />
          </div>
        )}

        {/* Note del maestro (se presenti) */}

        {/* Azioni: transizione stato + modifica/elimina */}
        {isMobile ? (
          <div className="flex items-center gap-2">
            {prevStatus && (
              <button onClick={() => onStatusChange(goal.id, prevStatus)}>◀ {STATUS_CONFIG[prevStatus].labelIt}</button>
            )}
            {nextStatus && (
              <button onClick={() => onStatusChange(goal.id, nextStatus)}>
                {nextStatus === 'completed' ? '✓ Completa' : '▶ Inizia'}
              </button>
            )}
          </div>
        ) : (
          <select value={goal.status} onChange={(e) => onStatusChange(goal.id, e.target.value as GoalStatus)}>
            {STATUS_COLUMNS.map((s) => <option key={s} value={s}>{STATUS_CONFIG[s].labelIt}</option>)}
          </select>
        )}
      </div>

      <ConfirmDialog open={confirmDelete} /* … elimina obiettivo … */ />
    </>
  );
}

// Kanban Column (desktop) drop target 
function KanbanColumn({ status, goals, /* … */ onDragStart, onDrop }: KanbanColumnProps) {
  const [dragOver, setDragOver] = useState(false);
  return (
    <div
      onDragOver={(e) => { e.preventDefault(); setDragOver(true); }}
      onDragLeave={() => setDragOver(false)}
      onDrop={(e) => { setDragOver(false); onDrop(e, status); }}
    >
      {/* header con conteggio goals.length */}
      {goals.map((goal) => <GoalCard key={goal.id} goal={goal} /* … */ />)}
    </div>
  );
}

// Mobile Tab View tab per stato + swipe 
function MobileTabView({ goals, /* … */ }: MobileTabViewProps) {
  const [activeStatus, setActiveStatus] = useState<GoalStatus>('in_progress');
  // gestione touchStart/touchMove/touchEnd → cambio colonna su swipe
  const filteredGoals = goals.filter((g) => g.status === activeStatus);
  // … tab bar + dots + lista card …
}

// KanbanBoard (export principale) 
interface KanbanBoardProps {
  goals: Goal[];
  isCoach: boolean;
  onEdit: (goal: Goal) => void;
  onDelete: (id: string) => void;
  onStatusChange: (id: string, status: GoalStatus) => void;
  onProgressChange: (id: string, progress: number) => void;
}

export function KanbanBoard({ goals, isCoach, onEdit, onDelete, onStatusChange, onProgressChange }: KanbanBoardProps) {
  const isMobile = useIsMobile();
  const [draggedId, setDraggedId] = useState<string | null>(null);
  const [categoryFilter, setCategoryFilter] = useState<string>('');

  const filtered = categoryFilter ? goals.filter((g) => g.category === categoryFilter) : goals;

  const handleDragStart = (e: React.DragEvent, id: string) => {
    setDraggedId(id);
    e.dataTransfer.effectAllowed = 'move';
  };

  const handleDrop = (e: React.DragEvent, status: GoalStatus) => {
    e.preventDefault();
    if (draggedId) {
      onStatusChange(draggedId, status);
      setDraggedId(null);
    }
  };

  return (
    <div className="flex flex-col h-full overflow-hidden">
      {isMobile ? (
        <MobileTabView goals={filtered} /* … */ />
      ) : (
        <div className="flex gap-4 overflow-x-auto pb-4">
          {STATUS_COLUMNS.map((status) => (
            <KanbanColumn
              key={status}
              status={status}
              goals={filtered.filter((g) => g.status === status)}
              onStatusChange={onStatusChange}
              onDragStart={handleDragStart}
              onDrop={handleDrop}
              /* … */
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

> Markup ripetitivo (SVG inline, classi Tailwind di stile) sintetizzato con `// …` per leggibilità. La logica partizionamento per stato, drag&drop, swipe, transizioni è riportata integralmente.

## `components/GoalForm.tsx`
Form a 3 step: `choice` → `catalog` (UC-08) → `form`. `handlePickTemplate` istanzia un obiettivo a partire dal template (copia campi, `status='planned'`, `progress=0`). Validazione: titolo obbligatorio, scadenza non nel passato.

```tsx
'use client';

import { useState, useEffect } from 'react';
import { ClipboardList, PencilLine } from 'lucide-react';
import type { Goal, GoalCategory, GoalStatus, GoalTemplate, PlayerLevel } from '@/lib/database.types';
import { CATEGORY_CONFIG, STATUS_CONFIG, STATUS_COLUMNS } from '@/lib/constants';
import { Button, Input, Textarea, Select, Modal } from './UI';
import { GoalTemplatePicker } from './GoalTemplatePicker';
import { CategoryIcon } from './CategoryIcon';

interface GoalFormProps {
  open: boolean;
  onClose: () => void;
  onSave: (data: Partial<Goal>) => Promise<void>;
  goal?: Goal | null;
  isCoach?: boolean;
  playerLevel?: PlayerLevel;
}

type Step = 'choice' | 'catalog' | 'form';

export function GoalForm({ open, onClose, onSave, goal, isCoach, playerLevel }: GoalFormProps) {
  const [step, setStep] = useState<Step>('form');
  const [fromTemplate, setFromTemplate] = useState(false);

  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [category, setCategory] = useState<GoalCategory>('tecnica');
  const [status, setStatus] = useState<GoalStatus>('planned');
  const [deadline, setDeadline] = useState('');
  const [progress, setProgress] = useState(0);
  const [coachNotes, setCoachNotes] = useState('');
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  useEffect(() => {
    if (!open) return;
    if (goal) {
      setStep('form'); setFromTemplate(false);
      setTitle(goal.title); setDescription(goal.description || '');
      setCategory(goal.category); setStatus(goal.status);
      setDeadline(goal.deadline || ''); setProgress(goal.progress);
      setCoachNotes(goal.coach_notes || '');
    } else {
      setStep('choice'); setFromTemplate(false);
      setTitle(''); setDescription(''); setCategory('tecnica');
      setStatus('planned'); setDeadline(''); setProgress(0); setCoachNotes('');
    }
    setError('');
  }, [goal, open]);

  // Template → istanza obiettivo (UC-08)
  const handlePickTemplate = (tpl: GoalTemplate) => {
    setTitle(tpl.title);
    setDescription(tpl.description || '');
    setCategory(tpl.category);
    setStatus('planned');
    setDeadline('');
    setProgress(0);
    setCoachNotes('');
    setFromTemplate(true);
    setStep('form');
    setError('');
  };

  const handleSubmit = async () => {
    if (!title.trim()) { setError('Inserisci un titolo'); return; }

    if (!goal && deadline) {
      const today = new Date(); today.setHours(0, 0, 0, 0);
      if (new Date(deadline) < today) { setError('La scadenza non può essere nel passato'); return; }
    }

    setSaving(true);
    try {
      await onSave({
        title: title.trim(),
        description: description.trim() || null,
        category,
        status,
        deadline: deadline || null,
        progress: status === 'in_progress' ? progress : status === 'completed' ? 100 : 0,
        coach_notes: coachNotes.trim() || null,
        ...(status === 'completed' && !goal?.completed_at
          ? { completed_at: new Date().toISOString() }
          : {}),
      });
      onClose();
    } catch {
      setError('Errore nel salvataggio');
    } finally {
      setSaving(false);
    }
  };
}
```

## `components/GoalTemplatePicker.tsx`
Catalogo lato allievo: filtro **puro** per categoria + livello + ricerca testuale.

```tsx
'use client';

import { useState, useEffect, useMemo } from 'react';
import { templateRepo } from '@/lib/repositories';
import type { GoalCategory, GoalTemplate, PlayerLevel } from '@/lib/database.types';
import { CATEGORY_CONFIG, LEVELS } from '@/lib/constants';
import { useIsMobile } from '@/lib/hooks';
// … UI imports …

interface GoalTemplatePickerProps {
  defaultLevel?: PlayerLevel;
  onSelect: (template: GoalTemplate) => void;
  onBack: () => void;
  onCreateCustom: () => void;
}

type CategoryFilter = '' | GoalCategory;
type LevelFilter = '' | PlayerLevel;

export function GoalTemplatePicker({ defaultLevel, onSelect, onBack, onCreateCustom }: GoalTemplatePickerProps) {
  const [templates, setTemplates] = useState<GoalTemplate[]>([]);
  const [loading, setLoading] = useState(true);
  const [categoryFilter, setCategoryFilter] = useState<CategoryFilter>('');
  const [levelFilter, setLevelFilter] = useState<LevelFilter>(defaultLevel ?? '');
  const [search, setSearch] = useState('');

  useEffect(() => {
    let cancelled = false;
    (async () => {
      setLoading(true);
      const res = await templateRepo.list();
      if (!cancelled) { setTemplates(res.data ?? []); setLoading(false); }
    })();
    return () => { cancelled = true; };
  }, []);

  // Filtro puro: O(n·m) — testabile in isolamento
  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return templates.filter((t) => {
      if (categoryFilter && t.category !== categoryFilter) return false;
      if (levelFilter && t.level !== levelFilter) return false;
      if (q) {
        const hay = `${t.title} ${t.description ?? ''}`.toLowerCase();
        if (!hay.includes(q)) return false;
      }
      return true;
    });
  }, [templates, categoryFilter, levelFilter, search]);

  // render: pills categoria/livello + SearchBar + griglia TemplateCard → onSelect(t)
  // … markup …
}
```

## `components/GoalTemplateManager.tsx`  `NUOVO`
Catalogo lato maestro (UC-15). Hook `useGoalTemplates` con CRUD via `templateRepo` e ricarica tramite `reloadTick`.

```tsx
'use client';

import { useState, useEffect, useMemo } from 'react';
import { templateRepo } from '@/lib/repositories';
import type { GoalCategory, GoalTemplate, PlayerLevel } from '@/lib/database.types';
import { CATEGORY_CONFIG, LEVELS } from '@/lib/constants';
import { useIsMobile } from '@/lib/hooks';
// … UI imports …

type CategoryFilter = '' | GoalCategory;
type LevelFilter = '' | PlayerLevel;

//Hook: stato + handlers CRUD
export function useGoalTemplates(coachId: string) {
  const [templates, setTemplates] = useState<GoalTemplate[]>([]);
  const [loading, setLoading] = useState(true);
  const [categoryFilter, setCategoryFilter] = useState<CategoryFilter>('');
  const [levelFilter, setLevelFilter] = useState<LevelFilter>('');
  const [search, setSearch] = useState('');
  const [formOpen, setFormOpen] = useState(false);
  const [editing, setEditing] = useState<GoalTemplate | null>(null);
  const [confirmDelete, setConfirmDelete] = useState<GoalTemplate | null>(null);
  const [reloadTick, setReloadTick] = useState(0);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      setLoading(true);
      const res = await templateRepo.list();
      if (!cancelled) { setTemplates(res.data ?? []); setLoading(false); }
    })();
    return () => { cancelled = true; };
  }, [reloadTick]);

  const filtered = useMemo(() => {
    const q = search.trim().toLowerCase();
    return templates.filter((t) => {
      if (categoryFilter && t.category !== categoryFilter) return false;
      if (levelFilter && t.level !== levelFilter) return false;
      if (q) {
        const hay = `${t.title} ${t.description ?? ''}`.toLowerCase();
        if (!hay.includes(q)) return false;
      }
      return true;
    });
  }, [templates, categoryFilter, levelFilter, search]);

  const handleSave = async (data: Partial<GoalTemplate>) => {
    if (editing) {
      await templateRepo.update(editing.id, data);
    } else {
      await templateRepo.create({ createdBy: coachId, data });
    }
    setReloadTick((t) => t + 1);
  };

  const handleDelete = async (tpl: GoalTemplate) => {
    await templateRepo.delete(tpl.id);
    setReloadTick((t) => t + 1);
  };

  return { templates, filtered, loading, categoryFilter, setCategoryFilter, levelFilter,
    setLevelFilter, search, setSearch, formOpen, editing, confirmDelete, setConfirmDelete,
    setFormOpen, setEditing, handleSave, handleDelete };
}

export type GoalTemplatesCtx = ReturnType<typeof useGoalTemplates>;

//  Componente contenitore (header + lista + FAB + form + dialog)
export function GoalTemplateManager({ coachId }: { coachId: string }) {
  const isMobile = useIsMobile();
  const ctx = useGoalTemplates(coachId);
  // render: <GoalTemplatesHeader/> + <GoalTemplatesList/> (+ TemplateForm: title/description/category/level)
  // ConfirmDialog elimina: "Gli obiettivi già copiati dagli allievi non verranno toccati."
  // … markup …
}
```
---
# 4. Wiring nei consumer

## `app/student/PlayerView.tsx` (frammenti obiettivi)
Punto in cui la UI consuma `IGoalRepository`. Da notare l'**aggiornamento ottimistico** in `handleGoalProgressChange` (update locale immediato, niente refetch).

```tsx
'use client';

import { useState, useEffect, useCallback } from 'react';
import { goalRepo, matchRepo } from '@/lib/repositories';
import type { Profile, Goal, GoalStatus, PlayerLevel } from '@/lib/database.types';
import { KanbanBoard } from '@/components/KanbanBoard';
import { GoalForm } from '@/components/GoalForm';
// … altri import …

export function PlayerView({ player, mode, writerId, /* … */ }: PlayerViewProps) {
  const [goals, setGoals] = useState<Goal[]>([]);
  const [goalFormOpen, setGoalFormOpen] = useState(false);
  const [editingGoal, setEditingGoal] = useState<Goal | null>(null);
  const [reloadTick, setReloadTick] = useState(0);
  const refetch = useCallback(() => setReloadTick((t) => t + 1), []);

  // Caricamento obiettivi dell'allievo
  useEffect(() => {
    let cancelled = false;
    (async () => {
      const [g, m] = await Promise.all([
        goalRepo.listByStudent(player.id),
        matchRepo.listByStudent(player.id),
      ]);
      if (cancelled) return;
      setGoals(g.data ?? []);
      // … setMatches …
    })();
    return () => { cancelled = true; };
  }, [player.id, reloadTick]);

  const isCoach = mode === 'coach';

  const triggerRefresh = async () => { refetch(); onDataChanged?.(); };

  // UC-07: crea o aggiorna
  const handleSaveGoal = async (data: Partial<Goal>) => {
    if (editingGoal) {
      await goalRepo.update(editingGoal.id, data);
    } else {
      await goalRepo.create({ studentId: player.id, createdBy: writerId, data });
    }
    await triggerRefresh();
    setEditingGoal(null);
  };

  const handleDeleteGoal = async (id: string) => {
    await goalRepo.delete(id);
    await triggerRefresh();
  };

  // UC-09: cambio stato (side-effect gestiti nel repository)
  const handleGoalStatusChange = async (id: string, status: GoalStatus) => {
    await goalRepo.changeStatus(id, status);
    await triggerRefresh();
  };

  // UC-10: aggiornamento OTTIMISTICO del progresso
  const handleGoalProgressChange = async (id: string, progress: number) => {
    await goalRepo.setProgress(id, progress);           // fire-and-forget
    setGoals((prev) => prev.map((g) => (g.id === id ? { ...g, progress } : g))); // update locale immediato
  };

  // render: <KanbanBoard goals={goals} isCoach={isCoach}
  //            onEdit={…} onDelete={handleDeleteGoal}
  //            onStatusChange={handleGoalStatusChange}
  //            onProgressChange={handleGoalProgressChange} />
  //         <GoalForm open={goalFormOpen} onSave={handleSaveGoal} goal={editingGoal}
  //            isCoach={isCoach} playerLevel={normalizedLevel} />
  // … (parte "match"/agonismo: fuori scope obiettivi) …
}
```

## `app/coach/tabs/CatalogoTab.tsx`
Tab del maestro che monta il gestore del catalogo.

```tsx
'use client';

import { GoalTemplateManager } from '@/components/GoalTemplateManager';

interface Props {
  coachId: string;
}

export function CatalogoTab({ coachId }: Props) {
  return (
    <div className="flex flex-col h-full animate-fade-in">
      <GoalTemplateManager coachId={coachId} />
    </div>
  );
}
```

---

# 5. Schema DB 

## `scripts/sql/2026_goal_templates.sql`
Tabella catalogo + trigger `updated_at` + RLS (lettura a tutti gli autenticati, scrittura solo maestro).

```sql
-- Catalogo Obiettivi (Goal Templates)
create table if not exists public.goal_templates (
  id          uuid primary key default gen_random_uuid(),
  category    text not null check (category in ('tecnica','tattica','fisico','mente','agonismo')),
  level       text not null check (level in ('Principiante','Intermedio','Avanzato')),
  title       text not null,
  description text,
  created_by  uuid references public.profiles(id) on delete set null,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now(),
  sort_order  int not null default 0
);

create index if not exists goal_templates_level_idx    on public.goal_templates (level);
create index if not exists goal_templates_category_idx on public.goal_templates (category);

-- Trigger updated_at
create or replace function public.tg_goal_templates_set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;

drop trigger if exists trg_goal_templates_updated_at on public.goal_templates;
create trigger trg_goal_templates_updated_at
  before update on public.goal_templates
  for each row execute function public.tg_goal_templates_set_updated_at();

-- RLS
alter table public.goal_templates enable row level security;

create policy "Tutti possono leggere i template"
  on public.goal_templates for select using (true);

create policy "Solo il maestro può inserire template"
  on public.goal_templates for insert
  with check (exists (select 1 from public.profiles where id = auth.uid() and role = 'maestro'));

create policy "Solo il maestro può aggiornare template"
  on public.goal_templates for update
  using (exists (select 1 from public.profiles where id = auth.uid() and role = 'maestro'));

create policy "Solo il maestro può eliminare template"
  on public.goal_templates for delete
  using (exists (select 1 from public.profiles where id = auth.uid() and role = 'maestro'));

