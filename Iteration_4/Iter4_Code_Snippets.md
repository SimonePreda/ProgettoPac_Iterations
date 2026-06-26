# Tennis Club Bellusco Iterazione 4

## 1. RF-05 Note del maestro sull'obiettivo

Re-implementato come **colonna** `goals.coach_notes`, speculare a `match_results.coach_notes`
(coerente con ADR-3-3). La tabella `goal_notes` era stata rimossa in Iter. 2 (ADR-4-1).

### 1.1 `lib/database.types.ts` colonna `coach_notes` su `goals`

```ts
goals: {
  Row: {
    id: string;
    student_id: string;
    // … campi esistenti …
    coach_notes: string | null;   // RF-05 (ADR-4-1)
    completed_at: string | null;
    created_at: string;
  };
  Insert: { /* … */ coach_notes?: string | null };
  Update: { /* … */ coach_notes?: string | null };
};
```

### 1.2 `lib/repositories/types.ts` contratto `IGoalRepository` (nessun metodo nuovo)

RF-05 **non** introduce un metodo dedicato: la scrittura passa dal già esistente `update`,
coerentemente con il Repository Pattern.

```ts
export interface IGoalRepository {
  listByStudent(studentId: string): Promise<RepoResult<GoalRow[]>>;
  listAllForStats(): Promise<RepoResult<Pick<GoalRow, 'id' | 'status' | 'created_at' | 'completed_at'>[]>>;
  create(input: { studentId: string; data: Partial<GoalRow> }): Promise<RepoResult<GoalRow>>;
  update(id: string, patch: Partial<GoalRow>): Promise<RepoResult<GoalRow>>; // ← usato anche per coach_notes
  delete(id: string): Promise<RepoResult<void>>;
}
```

### 1.3 `components/GoalForm.tsx` campo riservato al maestro

Il campo "Note del maestro" è mostrato **solo** quando `isCoach` è vero e confluisce nel payload
salvato via `goalRepo.update`.

```tsx
// Props: { open, onClose, onSave, goal, isCoach }  // (dedotto: isCoach)
const [coachNotes, setCoachNotes] = useState('');

useEffect(() => {
  if (goal) {
    // … precompila gli altri campi …
    setCoachNotes(goal.coach_notes ?? '');
  } else {
    // … reset …
    setCoachNotes('');
  }
}, [goal, open]);

const handleSubmit = async () => {
  // … validazione titolo/area/scadenza …
  await onSave({
    title: title.trim(),
    area,
    deadline: deadline || null,
    status,
    progress,
    // RF-05: incluso nel payload solo se il maestro sta modificando
    ...(isCoach ? { coach_notes: coachNotes.trim() || null } : {}),
  });
  onClose();
};

// Render: blocco visibile solo al maestro
{isCoach && (
  <Textarea
    label="Note del maestro (visibili all'allievo)"
    value={coachNotes}
    onChange={(e) => setCoachNotes(e.target.value)}
  />
)}
```

### 1.4 `components/KanbanBoard.tsx`  `GoalCard` lettura note (lato allievo)

```tsx
/* Nella GoalCard, sotto i dettagli dell'obiettivo: riquadro "Note del maestro"
   visibile all'allievo quando coach_notes è valorizzato (speculare a MatchCard). */
{goal.coach_notes && (
  <div className="coach-notes-box">
    <span className="coach-notes-label">Note del maestro</span>
    <p>{goal.coach_notes}</p>
  </div>
)}
```

## 2. UC-15  Allievo fittizio (creazione + hard-delete)

Profili senza email/telefono: vivono **solo** in `public.profiles` (nessuna FK verso
`auth.users`, id via `gen_random_uuid()`), marcati `is_fictitious = true` (ADR-4-2).

### 2.1 `lib/database.types.ts` flag su `profiles`

```ts
profiles: {
  Row: {
    id: string;
    full_name: string | null;
    role: 'maestro' | 'allievo';
    approval_status: 'pending' | 'approved' | 'rejected';
    is_fictitious: boolean;        // UC-15 (ADR-4-2)
    birth_date: string | null;     // categoria età FIT (RF-09)
    fit_ranking: string | null;    // classifica FIT
    level: string | null;          // livello tecnico (DELFINO/CERBIATTO/COCCODRILLO)
    photo_url: string | null;      // UC-19 avatar
    created_at: string;
  };
  // Insert/Update aggiornati di conseguenza
};
```

### 2.2 `lib/repositories/types.ts` estensione `IProfileRepository` (ADR-4-3)

```ts
export interface IProfileRepository {
  // … metodi esistenti (listApprovedStudents, listPendingStudents, getById, …) …

  /** UC-18: aggiornamento campi profilo (classifica/livello, data di nascita, photo_url). */
  updateProfile(id: string, patch: Partial<ProfileRow>): Promise<RepoResult<ProfileRow>>;

  /** UC-15: crea un allievo fittizio (solo public.profiles, id generato lato DB). */
  createFictitious(input: {
    fullName: string;
    birthDate: string | null;
    fitRanking: string | null;
    level: string | null;
  }): Promise<RepoResult<ProfileRow>>;

  /** UC-15: cancellazione fisica, ammessa solo se is_fictitious. */
  deleteFictitious(id: string): Promise<RepoResult<void>>;

  /** UC-19: upload avatar su Storage, ritorna l'URL pubblico con cache-buster. */
  uploadAvatar(userId: string, file: File): Promise<RepoResult<string>>;
}
```

### 2.3 `lib/repositories/profile.repository.ts` implementazioni nuove

```ts
async updateProfile(
  id: string,
  patch: Partial<ProfileRow>
): Promise<RepoResult<ProfileRow>> {
  const { data, error } = await this.client
    .from('profiles')
    .update(patch)
    .eq('id', id)
    .select('*')
    .single();
  if (error) return fail(error);
  return ok(data as ProfileRow);
}

async createFictitious(input: {
  fullName: string;
  birthDate: string | null;
  fitRanking: string | null;
  level: string | null;
}): Promise<RepoResult<ProfileRow>> {
  // id NON passato: generato dal default gen_random_uuid() lato DB.
  // RLS coach_insert_fictitious_profile autorizza (is_coach AND is_fictitious).
  const { data, error } = await this.client
    .from('profiles')
    .insert({
      full_name: input.fullName,
      birth_date: input.birthDate,
      fit_ranking: input.fitRanking,
      level: input.level,
      role: 'allievo',
      approval_status: 'approved',
      is_fictitious: true,
    } as unknown as Database['public']['Tables']['profiles']['Insert'])
    .select('*')
    .single();
  if (error) return fail(error);
  return ok(data as ProfileRow);
}

async deleteFictitious(id: string): Promise<RepoResult<void>> {
  // Guard ridondante al filtro RLS: cancella solo righe fittizie.
  const { error } = await this.client
    .from('profiles')
    .delete()
    .eq('id', id)
    .eq('is_fictitious', true);
  if (error) return fail(error);
  return ok(undefined);
}
```

### 2.4 `lib/repositories/index.ts` wiring (invariato per `profileRepo`)

```ts
export { SupabaseProfileRepository } from './profile.repository';
export const profileRepo = new SupabaseProfileRepository(supabase);
```

### 2.5 `components/CreateStudentForm.tsx` creazione (livelli canonici RF-06, ADR-4-4)

```tsx
// Livelli unificati sul dominio canonico (niente più Principiante/Intermedio/Avanzato)
const LEVELS = ['DELFINO', 'CERBIATTO', 'COCCODRILLO'] as const;  // RF-06 (ADR-4-4)

const handleSubmit = async () => {
  if (!firstName.trim() || !lastName.trim()) {
    setError('Inserisci nome e cognome');
    return;
  }
  const fullName = `${firstName.trim()} ${lastName.trim()}`;

  // Coerenza isClassified: una classifica numerica disattiva il livello tecnico
  const classified = isClassified(fitRanking);   // (dedotto: helper condiviso con ProfilePage)

  setSaving(true);
  try {
    await profileRepo.createFictitious({
      fullName,
      birthDate: birthDate || null,
      fitRanking: fitRanking.trim() || null,
      level: classified ? null : level,
    });
    onCreated();   // refetch lista allievi
    onClose();
  } catch {
    setError('Errore nella creazione');
  } finally {
    setSaving(false);
  }
};
```

### 2.6 `components/DeleteFictitiousStudentDialog.tsx` conferma type-to-confirm

```tsx
const normalize = (s: string) => s.trim().replace(/\s+/g, ' ').toLowerCase();

const expected = normalize(`cancella ${student.full_name ?? ''}`);
const canDelete = normalize(input) === expected;

const handleDelete = async () => {
  // Doppio guard lato client: solo profili fittizi
  if (!student.is_fictitious) return;
  setDeleting(true);
  try {
    await profileRepo.deleteFictitious(student.id);
    onDeleted();
    onClose();
  } finally {
    setDeleting(false);
  }
};

/* Render:
   - testo "Per confermare digita: cancella <nome completo>"
   - <Input value={input} onChange={…} />
   - <Button disabled={!canDelete || deleting} onClick={handleDelete}>Elimina</Button>
   Il pulsante si abilita SOLO a corrispondenza esatta. */
```

## 3. UC-18 · Modifica profilo

Modali distinte per **classifica FIT/livello**, **data di nascita** e **password**.
Attore: allievo (per sé) e maestro (per i fittizi). L'email non è modificabile.

### 3.1 `app/student/ProfilePage.tsx` modale classifica/livello (coerenza `isClassified`)

```tsx
// (dedotto path) Una classifica numerica disattiva il livello tecnico
const isClassified = (ranking: string | null) =>
  !!ranking && /\d/.test(ranking);   // (dedotto: presenza di cifre = classificato)

const handleSaveRanking = async () => {
  const classified = isClassified(fitRanking);
  await profileRepo.updateProfile(player.id, {
    fit_ranking: fitRanking.trim() || null,
    level: classified ? null : level,   // mutua esclusione
  });
  await triggerRefresh();
  setRankingModalOpen(false);
};
```

### 3.2 `app/student/ProfilePage.tsx` modale data di nascita

```tsx
const handleSaveBirthDate = async () => {
  await profileRepo.updateProfile(player.id, { birth_date: birthDate || null });
  await triggerRefresh();
  setBirthModalOpen(false);
};
```

### 3.3 `app/student/ProfilePage.tsx` modale password (verifica + update)

Il cambio password verifica **prima** la password attuale con `signInWithPassword`, poi la
aggiorna con `updateUser`.

```tsx
const handleChangePassword = async () => {
  setError(null);
  if (newPassword.length < 8) { setError('Minimo 8 caratteri'); return; }

  // 1) verifica password attuale
  const { error: signInErr } = await supabase.auth.signInWithPassword({
    email: player.email,            // (dedotto: email del profilo loggato)
    password: currentPassword,
  });
  if (signInErr) { setError('Password attuale errata'); return; }

  // 2) aggiornamento
  const { error: updateErr } = await supabase.auth.updateUser({ password: newPassword });
  if (updateErr) { setError('Errore aggiornamento password'); return; }

  setPwModalOpen(false);
};
```

### 3.4 Badge categoria età FIT (RF-09, inclusione di UC-18)

```tsx
import { getAgeCategory } from '@/lib/utils';   // (dedotto path)

const ageCategory = getAgeCategory(player.birth_date);
{ageCategory && <Badge>{ageCategory}</Badge>}
```
## 4. UC-19 Gestisci avatar

Selezione file → validazione tipo/dimensione → anteprima → upload su bucket `avatars`
in `{userId}/avatar.{ext}` con `upsert` → URL pubblico con cache-buster → `profiles.photo_url`.

### 4.1 `lib/repositories/profile.repository.ts` · `uploadAvatar`

```ts
async uploadAvatar(userId: string, file: File): Promise<RepoResult<string>> {
  const ext = file.name.split('.').pop()?.toLowerCase() ?? 'jpg';
  const path = `${userId}/avatar.${ext}`;

  const { error: upErr } = await this.client.storage
    .from('avatars')
    .upload(path, file, { upsert: true, contentType: file.type });
  if (upErr) return fail(upErr);

  // URL pubblico + cache-buster per forzare il refresh dell'immagine
  const { data } = this.client.storage.from('avatars').getPublicUrl(path);
  const publicUrl = `${data.publicUrl}?t=${Date.now()}`;

  // Persistenza su profiles.photo_url
  const { error: updErr } = await this.client
    .from('profiles')
    .update({ photo_url: publicUrl })
    .eq('id', userId);
  if (updErr) return fail(updErr);

  return ok(publicUrl);
}
```

### 4.2 `components/AvatarUpload.tsx` validazione + anteprima + upload

```tsx
const ACCEPTED = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_BYTES = 2 * 1024 * 1024;   // 2 MB

const handleFile = async (file: File) => {
  setError(null);
  if (!ACCEPTED.includes(file.type)) { setError('Formato non valido (JPG, PNG, WebP)'); return; }
  if (file.size > MAX_BYTES) { setError('Dimensione massima 2 MB'); return; }

  // Anteprima immediata
  setPreview(URL.createObjectURL(file));

  setUploading(true);
  try {
    const res = await profileRepo.uploadAvatar(userId, file);
    if (res.error) throw res.error;
    onUploaded(res.data);   // aggiorna photo_url nel parent
  } catch {
    setError('Errore durante il caricamento');
  } finally {
    setUploading(false);
  }
};
```

### 4.3 `components/AvatarDisplay.tsx` fallback iniziali

```tsx
export function AvatarDisplay({ photoUrl, fullName, size = 40 }: AvatarDisplayProps) {
  const initials = (fullName ?? '?')
    .split(' ').map((s) => s[0]).slice(0, 2).join('').toUpperCase();

  return photoUrl
    ? <img src={photoUrl} alt={fullName ?? ''} width={size} height={size} className="avatar-img" />
    : <div className="avatar-fallback" style={{ width: size, height: size }}>{initials}</div>;
}
```

## 5. Design in piccolo

### 5.1 `lib/utils.ts` · `getAgeCategory` (categoria età FIT, O(1)) — RF-09 / ADR-4-4

```ts
/** Età in anni compiuti dalla data di nascita; null se data assente/non valida. */
export function getAge(birthDate: string | null): number | null {
  if (!birthDate) return null;
  const b = new Date(birthDate);
  if (Number.isNaN(b.getTime())) return null;
  const now = new Date();
  let age = now.getFullYear() - b.getFullYear();
  const m = now.getMonth() - b.getMonth();
  if (m < 0 || (m === 0 && now.getDate() < b.getDate())) age--;
  return age;
}

/** Categoria FIT. Numero costante di confronti → O(1) in tempo e spazio. */
export function getAgeCategory(birthDate: string | null): string | null {
  const age = getAge(birthDate);          // O(1)
  if (age === null) return null;
  if (age <= 10) return 'U10';
  if (age <= 12) return 'U12';
  if (age <= 14) return 'U14';
  if (age <= 16) return 'U16';
  if (age <= 18) return 'U18';
  if (age <= 34) return 'NOR (Open)';     // allineato a envisioning (ADR-4-4)
  const bucket = Math.min(80, Math.floor(age / 5) * 5);
  return `Over ${bucket}`;
}
```

### 5.2 `deleteFictitious` conferma type-to-confirm (O(n))

```ts
const expected = normalize(`cancella ${student.full_name}`);
const enabled  = normalize(input) === expected;   // abilita pulsante solo se esatto
// on click → guard is_fictitious === true → delete where id = … and is_fictitious = true
```

## 6. Migrazioni SQL colonna RF-05 e RLS allievo fittizio

### 6.1 Colonna `coach_notes` su `goals` (RF-05, ADR-4-1)

```sql
alter table public.goals
  add column if not exists coach_notes text;
```

### 6.2 Profili fittizi: colonne e policy RLS (UC-15, ADR-4-2)

```sql
alter table public.profiles
  add column if not exists is_fictitious boolean not null default false;

-- helper esistente: is_coach() → true se l'utente corrente è maestro

-- INSERT: solo maestri, solo righe fittizie
create policy coach_insert_fictitious_profile
  on public.profiles for insert
  with check ( is_coach() and is_fictitious = true );

-- DELETE: solo maestri, solo righe fittizie (cancellazione fisica)
create policy coach_delete_fictitious_profile
  on public.profiles for delete
  using ( is_coach() and is_fictitious = true );
```

### 6.3 Storage bucket `avatars`: policy RLS per cartella utente (UC-19)

```sql
-- Scrittura/cancellazione limitate alla cartella {auth.uid()}/…
create policy avatar_write_own
  on storage.objects for insert
  with check ( bucket_id = 'avatars'
               and (storage.foldername(name))[1] = auth.uid()::text );

create policy avatar_update_own
  on storage.objects for update
  using ( bucket_id = 'avatars'
          and (storage.foldername(name))[1] = auth.uid()::text );

create policy avatar_delete_own
  on storage.objects for delete
  using ( bucket_id = 'avatars'
          and (storage.foldername(name))[1] = auth.uid()::text );

-- Lettura pubblica del bucket avatars
create policy avatar_read_public
  on storage.objects for select
  using ( bucket_id = 'avatars' );
```