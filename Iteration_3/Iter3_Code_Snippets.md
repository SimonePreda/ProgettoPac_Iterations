# Tennis Club Bellusco  Iterazione 3

Tema Iter. 3: **Agonismo · Dashboard & Statistiche · Finalizzazione PWA**.

## 1. Agonismo (risultati partite)

### 1.1 `lib/repositories/types.ts` contratto `IMatchRepository` 

```ts
// Match (agonistic results) 
export interface IMatchRepository {
  listByStudent(studentId: string): Promise<RepoResult<MatchResultRow[]>>;

  /** Tutti i match con `profiles` joinato — tab "Risultati Agonistici". */
  listAllWithStudent(limit?: number): Promise<RepoResult<MatchResultRow[]>>;

  /** Snapshot minimale per win-rate / volume. */
  listAllForStats(): Promise<
    RepoResult<Pick<MatchResultRow, 'id' | 'result' | 'match_date'>[]>
  >;

  create(input: {
    studentId: string;
    data: Partial<MatchResultRow>;
  }): Promise<RepoResult<MatchResultRow>>;

  update(id: string, patch: Partial<MatchResultRow>): Promise<RepoResult<MatchResultRow>>;

  setCoachNotes(id: string, notes: string | null): Promise<RepoResult<void>>;

  delete(id: string): Promise<RepoResult<void>>;
}
```

### 1.2 `lib/repositories/match.repository.ts` 

```ts
import type { SupabaseClient } from '@supabase/supabase-js';
import type { Database, MatchResultRow } from '../database.types';
import type { IMatchRepository, RepoResult } from './types';
import { ok, fail } from './errors';

type Client = SupabaseClient<Database>;

export class SupabaseMatchRepository implements IMatchRepository {
  constructor(private readonly client: Client) {}

  async listByStudent(studentId: string): Promise<RepoResult<MatchResultRow[]>> {
    const { data, error } = await this.client
      .from('match_results')
      .select('*')
      .eq('student_id', studentId)
      .order('match_date', { ascending: false });
    if (error) return fail(error);
    return ok((data ?? []) as MatchResultRow[]);
  }

  async listAllWithStudent(limit = 100): Promise<RepoResult<MatchResultRow[]>> {
    // Join: `profiles!match_results_student_id_fkey` è l'alias FK sulla tabella,
    // così la riga joinata compare sotto `.profiles` di ogni match.
    const { data, error } = await this.client
      .from('match_results')
      .select('*, profiles!match_results_student_id_fkey(full_name)')
      .order('match_date', { ascending: false })
      .limit(limit);
    if (error) return fail(error);
    return ok((data ?? []) as unknown as MatchResultRow[]);
  }

  async listAllForStats() {
    const { data, error } = await this.client
      .from('match_results')
      .select('id, result, match_date');
    if (error) return fail(error);
    return ok(
      (data ?? []) as Pick<MatchResultRow, 'id' | 'result' | 'match_date'>[]
    );
  }

  async create({
    studentId,
    data,
  }: {
    studentId: string;
    data: Partial<MatchResultRow>;
  }): Promise<RepoResult<MatchResultRow>> {
    const row = { ...data, student_id: studentId };
    const { data: inserted, error } = await this.client
      .from('match_results')
      .insert(
        row as unknown as Database['public']['Tables']['match_results']['Insert']
      )
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(inserted as MatchResultRow);
  }

  async update(
    id: string,
    patch: Partial<MatchResultRow>
  ): Promise<RepoResult<MatchResultRow>> {
    const { data, error } = await this.client
      .from('match_results')
      .update(patch)
      .eq('id', id)
      .select('*')
      .single();
    if (error) return fail(error);
    return ok(data as MatchResultRow);
  }

  async setCoachNotes(
    id: string,
    notes: string | null
  ): Promise<RepoResult<void>> {
    const { error } = await this.client
      .from('match_results')
      .update({ coach_notes: notes })
      .eq('id', id);
    if (error) return fail(error);
    return ok(undefined);
  }

  async delete(id: string): Promise<RepoResult<void>> {
    const { error } = await this.client
      .from('match_results')
      .delete()
      .eq('id', id);
    if (error) return fail(error);
    return ok(undefined);
  }
}
```

### 1.3 `lib/repositories/index.ts` wiring istanza match 

```ts
import { SupabaseMatchRepository } from './match.repository';
// …
export { SupabaseMatchRepository } from './match.repository';
//  Default browser-bound instances 
export const matchRepo = new SupabaseMatchRepository(supabase);
```

### 1.4 `components/MatchForm.tsx` validazione e payload 

```tsx
const handleSubmit = async () => {
  if (!tournament.trim()) { setError('Inserisci il nome del torneo'); return; }
  if (!matchDate) { setError('Inserisci la data'); return; }

  setSaving(true);
  try {
    await onSave({
      tournament_name: tournament.trim(),
      match_date: matchDate,
      opponent_name: opponent.trim() || null,
      score: score.trim() || null,
      result,
      round: round || null,
      surface,
      indoor,
      notes: notes.trim() || null,
    });
    onClose();
  } catch {
    setError('Errore nel salvataggio');
  } finally {
    setSaving(false);
  }
};

// Inizializzazione campi in edit vs creazione (useEffect su [match, open]):
//  - se `match` presente → precompila dai valori esistenti
//  - altrimenti → reset, con match_date = oggi (ISO yyyy-mm-dd)
```

### 1.5 `components/MatchCard.tsx` logica di presentazione 

```tsx
const isWin = match.result === 'win';
const date = new Date(match.match_date).toLocaleDateString('it-IT', {
  day: 'numeric', month: 'short', year: 'numeric',
});

const SURFACE_ICONS: Record<string, string> = {
  terra_rossa: '🟤', erba: '🟢', cemento: '⚪', sintetico: '🔵',
};

/* Rendering: striscia accento win/loss, badge esito (Vittoria/Sconfitta),
   punteggio + avversario, dettagli (superficie/turno/categoria),
   note allievo e blocco "Note del maestro" (match.coach_notes),
   azioni: note maestro (solo coach) / modifica / elimina con ConfirmDialog. */
```

### 1.6 `app/coach/tabs/RisultatiTab.tsx` filtro esito 

```tsx
const filtered = allMatches.filter((m) => {
  if (resultFilter === 'win') return m.result === 'win';
  if (resultFilter === 'loss') return m.result === 'loss';
  return true;
});
const winsCount   = allMatches.filter((m) => m.result === 'win').length;
const lossesCount = allMatches.filter((m) => m.result === 'loss').length;

/* Header "Risultati Agonistici" + FilterPill (Tutti / Vittorie / Sconfitte)
   con i rispettivi count; lista di CompactMatchCard. */
```

### 1.7 `components/CoachNotesForm.tsx` 

```tsx
export function CoachNotesForm({ open, onClose, onSave, currentNotes, title = 'Note del maestro' }: CoachNotesFormProps) {
  const [notes, setNotes] = useState('');
  const [saving, setSaving] = useState(false);

  useEffect(() => { setNotes(currentNotes || ''); }, [currentNotes, open]);

  const handleSave = async () => {
    setSaving(true);
    try {
      await onSave(notes.trim());
      onClose();
    } finally {
      setSaving(false);
    }
  };

  /* Modal con Textarea "Note (visibili all'allievo)" + Salva/Annulla */
}
```

### 1.8 `app/student/PlayerView.tsx` handler match 

```tsx
const handleSaveMatch = async (data: Partial<MatchResultRow>) => {
  if (editingMatch) {
    await matchRepo.update(editingMatch.id, data);
  } else {
    await matchRepo.create({ studentId: player.id, data });
  }
  await triggerRefresh();          // refetch completo (reloadTick++)
  setEditingMatch(null);
};

const handleDeleteMatch = async (id: string) => {
  await matchRepo.delete(id);
  await triggerRefresh();
};

const handleSaveCoachNotes = async (notes: string) => {
  if (coachNotesMatch) {
    await matchRepo.setCoachNotes(coachNotesMatch.id, notes || null);
    await triggerRefresh();
  }
};

// Caricamento iniziale (useEffect su [player.id, reloadTick]):
const [g, m] = await Promise.all([
  goalRepo.listByStudent(player.id),
  matchRepo.listByStudent(player.id),
]);
// Statistiche hero card (derivate, non persistite):
const wins = matches.filter((m) => m.result === 'win').length;
const winRate = matches.length > 0 ? Math.round((wins / matches.length) * 100) : 0;
```

---

## 2. Dashboard & Statistiche

### 2.1 `app/coach/CoachDashboard.tsx` `ClubStats` + aggregazione 
```tsx
interface ClubStats {
  studentsTotal: number;  studentsMonth: number;
  goalsTotal: number;     goalsMonth: number;
  matchesTotal: number;   matchesMonth: number;
  winRate: number;        winRateDelta: number;
}

// useEffect su [reloadTick] — letture parallele + aggregazione single-pass:
const monthStart = new Date(now.getFullYear(), now.getMonth(), 1);
const prevMonthStart = new Date(now.getFullYear(), now.getMonth() - 1, 1);
const monthStartIso = monthStart.toISOString();
const monthStartDay = monthStartIso.slice(0, 10);
const prevMonthStartDay = prevMonthStart.toISOString().slice(0, 10);

const [s, p, goalsRes, matchesRes] = await Promise.all([
  profileRepo.listApprovedStudents(),
  profileRepo.listPendingStudents(),
  goalRepo.listAllForStats(),
  matchRepo.listAllForStats(),
]);

const matches = matchesRes.data ?? [];
const matchesThisMonth = matches.filter((m) => m.match_date >= monthStartDay);
const matchesPrevMonth = matches.filter(
  (m) => m.match_date >= prevMonthStartDay && m.match_date < monthStartDay
);
const totalWins = matches.filter((m) => m.result === 'win').length;
const winRate = matches.length > 0 ? Math.round((totalWins / matches.length) * 100) : 0;
const wrThis = matchesThisMonth.length > 0
  ? Math.round((matchesThisMonth.filter((m) => m.result === 'win').length / matchesThisMonth.length) * 100)
  : winRate;
const wrPrev = matchesPrevMonth.length > 0
  ? Math.round((matchesPrevMonth.filter((m) => m.result === 'win').length / matchesPrevMonth.length) * 100)
  : wrThis;

setClubStats({
  studentsTotal: studentList.length,
  studentsMonth: studentList.filter((st) => st.created_at >= monthStartIso).length,
  goalsTotal: goals.length,
  goalsMonth: goals.filter((g) => g.created_at >= monthStartIso).length,
  matchesTotal: matches.length,
  matchesMonth: matchesThisMonth.length,
  winRate,
  winRateDelta: wrThis - wrPrev,
});
```

### 2.2 `app/coach/CoachDashboard.tsx` `StudentCard` stats per-card (count server-side) 

```tsx
// Conteggi aggregati lato server (head:true → nessun trasferimento di righe)
const [{ count: goalsCount }, { count: completedCount }, { data: matchData }] =
  await Promise.all([
    supabase.from('goals').select('*', { count: 'exact', head: true }).eq('student_id', student.id),
    supabase.from('goals').select('*', { count: 'exact', head: true }).eq('student_id', student.id).eq('status', 'completed'),
    supabase.from('match_results').select('result').eq('student_id', student.id),
  ]);
const wins = (matchData || []).filter((m: { result: string }) => m.result === 'win').length;
```

### 2.3 `app/coach/tabs/HomeTab.tsx` win rate + feed notifiche (mobile) 

```tsx
const winRate = useMemo(() => {
  if (allMatches.length === 0) return 0;
  const wins = allMatches.filter((m) => m.result === 'win').length;
  return Math.round((wins / allMatches.length) * 100);
}, [allMatches]);

const matchesThisMonth = useMemo(() => {
  const now = new Date();
  return allMatches.filter((m) => {
    const d = new Date(m.match_date);
    return d.getMonth() === now.getMonth() && d.getFullYear() === now.getFullYear();
  }).length;
}, [allMatches]);

// Feed unificato goal-completati + match, filtrato, ordinato, troncato a 30:
const notifications = useMemo<Notif[]>(() => {
  const list: Notif[] = [];
  for (const g of recentGoals) {
    if (g.status === 'completed' && g.completed_at) {
      list.push({ id: `goal:${g.id}`, kind: 'goal', /* … */ date: new Date(g.completed_at) });
    }
  }
  for (const m of allMatches) {
    list.push({ id: `match:${m.id}`, kind: 'match', /* … */ date: new Date(m.created_at) });
  }
  return list
    .filter((n) => !dismissed.has(n.id))
    .sort((a, b) => b.date.getTime() - a.date.getTime())
    .slice(0, 30);
}, [recentGoals, allMatches, dismissed]);
```

---

## 3. Finalizzazione PWA

### 3.1 `public/sw.js` 

```js
const CACHE_NAME = 'tc-bellusco-v1';

self.addEventListener('install', (event) => { self.skipWaiting(); });

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((k) => k !== CACHE_NAME).map((k) => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (event.request.method !== 'GET') return;
  if (url.hostname.includes('supabase')) return;                 // sempre rete
  if (url.pathname.startsWith('/_next/webpack-hmr')) return;     // dev HMR

  // Pagine (navigate): network-first con fallback offline
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((c) => c.put(event.request, clone));
          return response;
        })
        .catch(() => caches.match(event.request).then((r) => r || caches.match('/')))
    );
    return;
  }

  // Asset statici: cache-first
  if (
    url.pathname.startsWith('/_next/static/') ||
    url.pathname.startsWith('/icons/') ||
    url.pathname.endsWith('.css') || url.pathname.endsWith('.js') ||
    url.pathname.endsWith('.png') || url.pathname.endsWith('.svg') ||
    url.pathname.endsWith('.woff2')
  ) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        if (cached) return cached;
        return fetch(event.request).then((response) => {
          if (response.ok) {
            const clone = response.clone();
            caches.open(CACHE_NAME).then((c) => c.put(event.request, clone));
          }
          return response;
        });
      })
    );
    return;
  }
});

// Push + click notifica (predisposti per uso futuro)
self.addEventListener('push', (event) => { /* showNotification(...) */ });
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  event.waitUntil(clients.openWindow('/'));
});
```

### 3.2 `app/manifest.ts` 

```ts
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Tennis Club Bellusco',
    short_name: 'TC Bellusco',
    description: 'Gestione allievi e maestri - Tennis Club Bellusco',
    start_url: '/',
    display: 'standalone',
    background_color: '#F7F8FA',
    theme_color: '#C41E3A',          
    orientation: 'portrait-primary',
    icons: [
      { src: '/icons/icon-192x192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icons/icon-512x512.png', sizes: '512x512', type: 'image/png' },
    ],
  };
}
```

### 3.3 `next.config.ts` header del Service Worker

```ts
{
  // Service worker: no caching, content-type corretto, CSP dedicata
  source: '/sw.js',
  headers: [
    { key: 'Content-Type', value: 'application/javascript; charset=utf-8' },
    { key: 'Cache-Control', value: 'no-cache, no-store, must-revalidate' },
    { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self'" },
  ],
}

