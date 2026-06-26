# Costruire una chiamata completa: gli obiettivi di uno studente dall'account del maestro

> Documento tecnico Tennis Bellusco Project
> Come si compone, passo dopo passo, la richiesta HTTP che recupera gli obiettivi
> di un allievo qualsiasi partendo dal login del maestro, e cosa inserire in
> Postman per vederne la risposta.

---

## 1. Lo scenario

Il maestro è loggato e vuole aprire la scheda di un allievo per vederne gli
obiettivi del Kanban. Nell'app questo accade dentro `PlayerView`, che chiama
`goalRepo.listByStudent(studentId)`. Ma `studentId` da dove arriva? Dalla lista
allievi che il maestro ha caricato prima. Quindi una singola schermata nasconde
**tre chiamate in sequenza**, e per replicarle in Postman dobbiamo ripercorrerle
tutte e tre:

1. **Login** del maestro → ottengo un *access token* (JWT).
2. **Lista allievi** → scelgo un allievo e ne leggo l'`id`.
3. **Obiettivi di quell'allievo** → la chiamata finale.

Il motivo per cui si parte "dall'account del maestro" non è formale: è la
**Row Level Security (RLS)** di PostgreSQL a decidere cosa vedi. Con il token del
maestro (`role = 'maestro'`) puoi leggere gli obiettivi di *qualunque* allievo;
con il token di un allievo vedresti solo i tuoi. È il token, non l'URL, a
sbloccare l'accesso.

---

## 2. Anatomia di una chiamata Supabase

Tutte le chiamate ai dati hanno la stessa forma. L'API REST (PostgREST) vive su:

```
https://zcvmfqmrljhckilqhipq.supabase.co/rest/v1/
```

e ogni richiesta porta **sempre** questi header:

| Header | Valore | Perché |
|---|---|---|
| `apikey` | la *anon key* del progetto | Identifica il progetto al gateway Supabase |
| `Authorization` | `Bearer <access_token>` | Identifica **l'utente**: è ciò che la RLS valuta |
| `Content-Type` | `application/json` | Solo per richieste con body (POST/PATCH) |

> La `anon key` è pubblica per design. Da sola NON dà accesso ai dati: senza un token
> utente valido nel `Bearer`, la RLS ti tratta come anonimo e le query tornano
> vuote o vengono rifiutate.

### Come i filtri del codice diventano querystring

PostgREST traduce ogni metodo della catena `supabase-js` in un parametro URL:

| Codice | URL |
|---|---|
| `.select('*')` | `?select=*` |
| `.eq('student_id', x)` | `&student_id=eq.x` |
| `.is('path_node_id', null)` | `&path_node_id=is.null` |
| `.order('sort_order')` | `&order=sort_order` |
| `.order('match_date', { ascending: false })` | `&order=match_date.desc` |

---

## 3. Ricostruzione passo per passo

### Passo 1 Login del maestro (ottenere il token)

Nel codice è `supabase.auth.signInWithPassword({ email, password })`. In HTTP è
una POST all'endpoint di autenticazione (GoTrue):

```
POST https://zcvmfqmrljhckilqhipq.supabase.co/auth/v1/token?grant_type=password
```

Header:
```
apikey: <anon key>
Content-Type: application/json
```

Body:
```json
{
  "email": "maestro@tennisbellusco.it",
  "password": "********"
}
```

Risposta (estratto):
```json
{
  "access_token": "eyJhbGciOiJIUzI1Ni...",   // ← questo è il token che ci serve
  "token_type": "bearer",
  "expires_in": 3600,
  "refresh_token": "....",
  "user": { "id": "...", "email": "maestro@tennisbellusco.it" }
}
```

Da qui in poi useremo `access_token` come `Authorization: Bearer ...`.

---

### Passo 2 Elenco degli allievi (trovare lo `student_id`)

Nel codice è `profileRepo.listApprovedStudents()`, cioè:

```js
.from('profiles').select('*')
  .eq('role', 'allievo')
  .eq('approval_status', 'approved')
  .eq('active', true)
  .order('full_name')
```

In HTTP:

```
GET https://zcvmfqmrljhckilqhipq.supabase.co/rest/v1/profiles?select=*&role=eq.allievo&approval_status=eq.approved&active=is.true&order=full_name
```

Header:
```
apikey: <anon key>
Authorization: Bearer <access_token del maestro>
```

Risposta (estratto): una lista di profili. Da uno di essi copio il campo `id`,
che è il nostro `studentId` per il passo finale:

```json
[
  {
    "id": "8f3c1a90-...-d21b",     // ← studentId
    "role": "allievo",
    "full_name": "Marco Rossi",
    "approval_status": "approved",
    "active": true
  }
]
```

---

### Passo 3 Gli obiettivi di quell'allievo (la chiamata finale)

Nel codice è `goalRepo.listByStudent(studentId)`:

```js
.from('goals').select('*')
  .eq('student_id', studentId)
  .is('path_node_id', null)     // solo gli obiettivi "liberi" del Kanban
  .order('sort_order')
```

> Nota: `path_node_id=is.null` esclude gli obiettivi generati da un Percorso
> (skill tree). Quelli si leggono separatamente con `listByStudentPath`. Se vuoi
> **tutti** gli obiettivi, togli quel filtro.

In HTTP la chiamata completa è:

```
GET https://zcvmfqmrljhckilqhipq.supabase.co/rest/v1/goals?select=*&student_id=eq.8f3c1a90-...-d21b&path_node_id=is.null&order=sort_order
```

Header:
```
apikey: <anon key>
Authorization: Bearer <access_token del maestro>
```

Risposta attesa (estratto):
```json
[
  {
    "id": "a1b2...",
    "student_id": "8f3c1a90-...-d21b",
    "category": "tecnica",
    "title": "Rovescio in topspin",
    "status": "in_progress",
    "progress": 40,
    "deadline": "2026-07-31",
    "sort_order": 1,
    "path_node_id": null
  },
  {
    "id": "c3d4...",
    "student_id": "8f3c1a90-...-d21b",
    "category": "fisico",
    "title": "Migliorare lo split-step",
    "status": "planned",
    "progress": 0,
    "sort_order": 2,
    "path_node_id": null
  }
]
```

Cosa è successo dietro le quinte: PostgREST ha ricevuto la GET, PostgreSQL ha
verificato il JWT nel `Bearer`, la RLS ha visto `role = 'maestro'` e ha
**autorizzato** la lettura delle righe di un altro utente, poi ha eseguito la
query con i filtri e ha restituito il JSON.

---
