# Appendice Codice nuovo / modificato

**Tennis Club Bellusco Iterazione 5: Percorsi di obiettivi (Skill Tree)**

## 1. `lib/paths/topo.ts` algoritmo esibito (funzione pura)

```ts
export type NodeId = string;

export interface TopoNode { id: NodeId; }

/** Arco del DAG: `from` è PREREQUISITO di `to`. */
export interface TopoEdge { from: NodeId; to: NodeId; }

export interface PathState {
  /** false => il grafo contiene un ciclo (non è un DAG valido). */
  valid: boolean;
  /** Ordine topologico dei nodi (parziale se `valid` è false). */
  order: NodeId[];
  /** Riga/strato: cammino più lungo dalle radici (radici = 0). */
  layer: Record<NodeId, number>;
  /** true => tutti i prerequisiti del nodo sono completati. */
  unlocked: Record<NodeId, boolean>;
  /** Prerequisiti ancora da completare, per nodo (vuoto se sbloccato). */
  blockedBy: Record<NodeId, NodeId[]>;
}

export function computePathState(
  nodes: TopoNode[],
  edges: TopoEdge[],
  completion: ReadonlySet<NodeId>
): PathState {
  const adj = new Map<NodeId, NodeId[]>();
  const indegree = new Map<NodeId, number>();
  const prereqs = new Map<NodeId, NodeId[]>();

  for (const n of nodes) {
    adj.set(n.id, []);
    indegree.set(n.id, 0);
    prereqs.set(n.id, []);
  }

  // Archi verso nodi inesistenti ignorati (robustezza su dati incoerenti).
  for (const e of edges) {
    if (!adj.has(e.from) || !indegree.has(e.to)) continue;
    adj.get(e.from)!.push(e.to);
    indegree.set(e.to, indegree.get(e.to)! + 1);
    prereqs.get(e.to)!.push(e.from);
  }

  const layer: Record<NodeId, number> = {};
  const queue: NodeId[] = [];
  for (const n of nodes) {
    layer[n.id] = 0;
    if (indegree.get(n.id) === 0) queue.push(n.id); // radici
  }

  // Copia locale dell'in-degree: la funzione resta pura.
  const remaining = new Map(indegree);
  const order: NodeId[] = [];

  let head = 0;
  while (head < queue.length) {
    const u = queue[head++];
    order.push(u);
    for (const v of adj.get(u)!) {
      if (layer[u] + 1 > layer[v]) layer[v] = layer[u] + 1; // longest path
      const d = remaining.get(v)! - 1;
      remaining.set(v, d);
      if (d === 0) queue.push(v);
    }
  }

  // Se non abbiamo processato tutti i nodi => ciclo.
  const valid = order.length === nodes.length;

  const unlocked: Record<NodeId, boolean> = {};
  const blockedBy: Record<NodeId, NodeId[]> = {};
  for (const n of nodes) {
    const missing = prereqs.get(n.id)!.filter((p) => !completion.has(p));
    blockedBy[n.id] = missing;
    unlocked[n.id] = missing.length === 0;
  }

  return { valid, order, layer, unlocked, blockedBy };
}

/** Helper usato dall'editor del maestro al salvataggio. */
export function isAcyclic(nodes: TopoNode[], edges: TopoEdge[]): boolean {
  return computePathState(nodes, edges, new Set()).valid;
}
```

> **Punto d'esame.** Un solo attraversamento ricava *tre* risultati (validità,
> layer, frontiera). `valid`/`order` vengono dall'attraversamento Kahn;
> `unlocked`/`blockedBy` da una passata indipendente sui prerequisiti — corretta
> anche con più componenti. Complessità **O(V + E)** tempo e spazio.

---

## 2. `lib/repositories/types.ts` — contratti nuovi (estratto)

```ts
// Percorsi (Skill Tree) 

/** Il grafo completo di un percorso: nodi + archi (prerequisiti). */
export interface PathGraph {
  nodes: PathNode[];
  edges: PathEdge[];
}

/** Bozza di nodo usata dall'editor (id generato lato client). */
export interface PathNodeDraft {
  id: string;
  title: string;
  category: GoalCategory;
  description: string | null;
  goal_template_id: string | null;
  sort_order: number;
}

export interface IPathRepository {
  list(): Promise<RepoResult<Path[]>>;
  getById(id: string): Promise<RepoResult<Path | null>>;
  create(input: { createdBy: string; data: Partial<Path> }): Promise<RepoResult<Path>>;
  update(id: string, patch: Partial<Path>): Promise<RepoResult<Path>>;
  delete(id: string): Promise<RepoResult<void>>;
  getGraph(pathId: string): Promise<RepoResult<PathGraph>>;
  /** Il chiamante DEVE aver già validato l'aciclicità con `isAcyclic`. */
  saveGraph(
    pathId: string,
    nodes: PathNodeDraft[],
    edges: { from: string; to: string }[]
  ): Promise<RepoResult<void>>;
}

export interface ActiveStudentPath {
  studentPath: StudentPath;
  path: Path;
}

export interface IStudentPathRepository {
  listActiveByStudent(studentId: string): Promise<RepoResult<ActiveStudentPath[]>>;
  listActivationsByPath(pathId: string): Promise<RepoResult<string[]>>;
  /** Idempotente. Ritorna l'id di `student_paths`. */
  activate(pathId: string, studentId: string): Promise<RepoResult<string>>;
  /** Rimuove l'istanza e CANCELLA i goal materializzati per l'allievo. */
  deactivate(pathId: string, studentId: string): Promise<RepoResult<void>>;
}
```

Estensione di `IGoalRepository` (separazione Kanban ↔ percorso):

```ts
export interface IGoalRepository {
  /** Kanban: solo obiettivi liberi (path_node_id IS NULL). */
  listByStudent(studentId: string): Promise<RepoResult<Goal[]>>;
  /** Goal materializzati di uno specifico percorso. */
  listByStudentPath(studentId: string, pathId: string): Promise<RepoResult<Goal[]>>;
  // …create / update / changeStatus / setProgress / delete invariati…
}
```
## 3. `lib/repositories/path.repository.ts` lettura grafo + RPC (estratto)

```ts
async getGraph(pathId: string): Promise<RepoResult<PathGraph>> {
  const [nodesRes, edgesRes] = await Promise.all([
    this.client.from('path_nodes').select('*').eq('path_id', pathId).order('sort_order'),
    this.client.from('path_edges').select('*').eq('path_id', pathId),
  ]);
  if (nodesRes.error) return fail(nodesRes.error);
  if (edgesRes.error) return fail(edgesRes.error);
  return ok({
    nodes: (nodesRes.data ?? []) as PathGraph['nodes'],
    edges: (edgesRes.data ?? []) as PathGraph['edges'],
  });
}

async saveGraph(
  pathId: string,
  nodes: PathNodeDraft[],
  edges: { from: string; to: string }[]
): Promise<RepoResult<void>> {
  // RPC transazionale: sostituisce nodi e archi in un colpo solo.
  const { error } = await this.client.rpc('save_path_graph', {
    p_path_id: pathId,
    p_nodes: nodes as unknown as Record<string, unknown>[],
    p_edges: edges as unknown as Record<string, unknown>[],
  });
  if (error) return fail(error);
  return ok(undefined);
}
```

## 4. `lib/repositories/student-path.repository.ts` attivazione (estratto)

```ts
async listActiveByStudent(studentId: string): Promise<RepoResult<ActiveStudentPath[]>> {
  const { data, error } = await this.client
    .from('student_paths')
    .select('*, paths(*)')               // join sul percorso
    .eq('student_id', studentId)
    .order('activated_at', { ascending: false });
  if (error) return fail(error);
  const rows = (data ?? []) as unknown as (StudentPath & { paths: Path | null })[];
  return ok(
    rows.filter((r) => r.paths !== null).map((r) => {
      const { paths, ...studentPath } = r;
      return { studentPath: studentPath as StudentPath, path: paths as Path };
    })
  );
}

async activate(pathId: string, studentId: string): Promise<RepoResult<string>> {
  const { data, error } = await this.client.rpc('activate_path', {
    p_path_id: pathId, p_student_id: studentId,
  });
  if (error) return fail(error);
  return ok(data as string);
}

async deactivate(pathId: string, studentId: string): Promise<RepoResult<void>> {
  const { error } = await this.client.rpc('deactivate_path', {
    p_path_id: pathId, p_student_id: studentId,
  });
  if (error) return fail(error);
  return ok(undefined);
}
```
## 5. `lib/repositories/goal.repository.ts` separazione viste (modificato)

```ts
async listByStudent(studentId: string): Promise<RepoResult<Goal[]>> {
  // Solo obiettivi "liberi" del Kanban: NON appartenenti a un percorso.
  const { data, error } = await this.client
    .from('goals')
    .select('*')
    .eq('student_id', studentId)
    .is('path_node_id', null)           // <─ filtro nuovo
    .order('sort_order');
  if (error) return fail(error);
  return ok((data ?? []) as Goal[]);
}

async listByStudentPath(studentId: string, pathId: string): Promise<RepoResult<Goal[]>> {
  // Join sul nodo (FK goals.path_node_id -> path_nodes.id) filtrato sul path.
  const { data, error } = await this.client
    .from('goals')
    .select('*, path_nodes!inner(path_id)')
    .eq('student_id', studentId)
    .eq('path_nodes.path_id', pathId)
    .order('sort_order');
  if (error) return fail(error);
  return ok((data ?? []) as unknown as Goal[]);
}
```

## 6. `scripts/sql/2026_paths.sql` schema + RLS (estratto)

```sql
-- ARCHI del DAG (prerequisiti). from = PREREQUISITO, to = DIPENDENTE.
create table if not exists public.path_edges (
  id           uuid primary key default gen_random_uuid(),
  path_id      uuid not null references public.paths(id) on delete cascade,
  from_node_id uuid not null references public.path_nodes(id) on delete cascade,
  to_node_id   uuid not null references public.path_nodes(id) on delete cascade,
  created_at   timestamptz not null default now(),
  constraint path_edges_no_self check (from_node_id <> to_node_id),  -- no self-loop
  constraint path_edges_unique  unique (from_node_id, to_node_id)
);

-- Istanza: percorso attivato per un allievo (un percorso per allievo).
create table if not exists public.student_paths (
  id           uuid primary key default gen_random_uuid(),
  student_id   uuid not null references public.profiles(id) on delete cascade,
  path_id      uuid not null references public.paths(id)    on delete cascade,
  activated_at timestamptz not null default now(),
  activated_by uuid references public.profiles(id) on delete set null,
  constraint student_paths_unique unique (student_id, path_id)
);

-- RLS: legge l'autenticato, scrive il maestro.
create policy "select_student_paths" on public.student_paths
  for select to authenticated
  using (student_id = auth.uid() or public.is_coach(auth.uid()));
```

## 7. `scripts/sql/2026_paths_iteration_b.sql` RPC transazionali

> **DT-2 (debito tecnico).** La FK `goals.path_node_id` qui è
> **`ON DELETE CASCADE`** (Iter. B/C), mentre `2026_paths.sql` (Iter. A) la
> dichiarava `ON DELETE SET NULL`. Lo stato reale del DB è CASCADE: eliminare un
> nodo/percorso elimina i goal materializzati (richiesto da "Disattiva percorso").

```sql
-- FK goals -> path_nodes con CASCADE (stato attuale).
alter table public.goals drop constraint if exists goals_path_node_id_fkey;
alter table public.goals
  add constraint goals_path_node_id_fkey
  foreign key (path_node_id) references public.path_nodes(id) on delete cascade;

-- Anti-duplicati: un goal per (allievo, nodo).
create unique index if not exists goals_student_pathnode_uidx
  on public.goals (student_id, path_node_id)
  where path_node_id is not null;
```

### RPC `activate_path` materializza i nodi come goal `planned`

```sql
create or replace function public.activate_path(p_path_id uuid, p_student_id uuid)
returns uuid language plpgsql security definer
set search_path to 'pg_catalog', 'public' as $$
declare v_student_path_id uuid;
begin
  if not public.is_coach(auth.uid()) then raise exception 'not authorized'; end if;

  insert into public.student_paths (student_id, path_id, activated_by)
  values (p_student_id, p_path_id, auth.uid())
  on conflict (student_id, path_id)
    do update set activated_at = public.student_paths.activated_at  -- idempotente
  returning id into v_student_path_id;

  insert into public.goals
    (student_id, category, title, description, status, progress, created_by, path_node_id, sort_order)
  select p_student_id, pn.category::public.goal_category, pn.title, pn.description,
         'planned'::public.goal_status, 0, auth.uid(), pn.id, pn.sort_order
  from public.path_nodes pn
  where pn.path_id = p_path_id
    and not exists (                       -- non materializzare due volte
      select 1 from public.goals g
      where g.student_id = p_student_id and g.path_node_id = pn.id
    );

  return v_student_path_id;
end; $$;
```

### RPC `deactivate_path` rimuove goal + istanza

```sql
create or replace function public.deactivate_path(p_path_id uuid, p_student_id uuid)
returns void language plpgsql security definer
set search_path to 'pg_catalog', 'public' as $$
begin
  if not public.is_coach(auth.uid()) then raise exception 'not authorized'; end if;

  delete from public.goals
  where student_id = p_student_id
    and path_node_id in (select id from public.path_nodes where path_id = p_path_id);

  delete from public.student_paths
  where student_id = p_student_id and path_id = p_path_id;
end; $$;
```

### RPC `save_path_graph` upsert nodi, sostituzione archi

```sql
create or replace function public.save_path_graph(p_path_id uuid, p_nodes jsonb, p_edges jsonb)
returns void language plpgsql security definer
set search_path to 'pg_catalog', 'public' as $$
begin
  if not public.is_coach(auth.uid()) then raise exception 'not authorized'; end if;

  delete from public.path_edges where path_id = p_path_id;

  -- Nodi rimossi: cancellati (i goal collegati vanno via in cascata).
  delete from public.path_nodes
  where path_id = p_path_id
    and id not in (
      select (n->>'id')::uuid
      from jsonb_array_elements(coalesce(p_nodes, '[]'::jsonb)) as n
    );

  -- Nodi nuovi o mantenuti: upsert per id (i goal dei mantenuti restano).
  insert into public.path_nodes
    (id, path_id, goal_template_id, title, category, description, sort_order)
  select (n->>'id')::uuid, p_path_id, nullif(n->>'goal_template_id','')::uuid,
         n->>'title', n->>'category', nullif(n->>'description',''),
         coalesce((n->>'sort_order')::int, 0)
  from jsonb_array_elements(coalesce(p_nodes, '[]'::jsonb)) as n
  on conflict (id) do update set
    goal_template_id = excluded.goal_template_id, title = excluded.title,
    category = excluded.category, description = excluded.description,
    sort_order = excluded.sort_order;

  insert into public.path_edges (path_id, from_node_id, to_node_id)
  select p_path_id, (e->>'from')::uuid, (e->>'to')::uuid
  from jsonb_array_elements(coalesce(p_edges, '[]'::jsonb)) as e;
end; $$;

-- Esecuzione: revocata a public, concessa solo a authenticated.
revoke execute on function public.activate_path(uuid, uuid)              from public;
revoke execute on function public.deactivate_path(uuid, uuid)            from public;
revoke execute on function public.save_path_graph(uuid, jsonb, jsonb)    from public;
grant  execute on function public.activate_path(uuid, uuid)              to authenticated;
grant  execute on function public.deactivate_path(uuid, uuid)            to authenticated;
grant  execute on function public.save_path_graph(uuid, jsonb, jsonb)    to authenticated;
```

## 9. Integrazione UI (descrizione)

- **`app/student/PlayerView.tsx`**  un solo caricamento per la tab "Il mio
  percorso" (`goalRepo.listByStudent` per i liberi, `studentPathRepo.listActiveByStudent`,
  `pathRepo.getGraph`, `goalRepo.listByStudentPath`). Da nodi+archi+goal calcola
  `computePathState`; con la frontiera `unlocked` costruisce **sia il Kanban**
  (obiettivi liberi **+ solo i nodi sbloccati**) **sia l'albero** (tutti i nodi).
  Le azioni sul nodo (`changeStatus`, `setProgress`) chiamano `triggerRefresh()`
  → ricarica → ricalcolo Kahn.
- **`components/PathTreeView.tsx`**  riceve i nodi con `layer` (riga) e lo stato
  visivo (bloccato / disponibile / in corso / completato). Al ricalcolo, il
  **diff** tra la frontiera precedente e la nuova marca i nodi appena sbloccati
  con la classe **`.animate-unlock`** I connettori prerequisito→nodo
  sono SVG misurati sul DOM reale.
- **`components/PathEditor.tsx`**  editor a form: si aggiungono nodi (dal catalogo
  o custom) e per ciascuno si scelgono i prerequisiti via chip. `isAcyclic` valida
  in tempo reale; se compare un ciclo il salvataggio è bloccato. Quando il grafo è
  valido, l'anteprima auto-layout usa lo **stesso `layer`** della vista allievo.
