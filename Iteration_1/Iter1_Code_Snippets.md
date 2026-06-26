# Iterazione 1 — Frammenti di codice aggiunti / modificati

> **Tennis Club Bellusco — Iter. 1: Fondamenta auth + approval workflow**
>
> Questo documento elenca i frammenti di codice realmente prodotti durante l'Iterazione 1.
> Per ogni file: scopo, frammenti rilevanti, casi d'uso coperti.
> Lo schema DB è il punto di partenza di tutto: trigger + RLS + tabelle.
> Tutti i percorsi sono relativi alla radice del progetto.

---

## Indice

1. [Schema database (Supabase migration)](#1-schema-database-supabase-migration)
2. [Wrapper Supabase — 3 modalità di accesso](#2-wrapper-supabase--3-modalità-di-accesso)
3. [Proxy Next.js 16 (ex middleware.ts)](#3-proxy-nextjs-16-ex-middlewarets)
4. [Route handler `/auth/callback`](#4-route-handler-authcallback)
5. [`AuthContext` — cuore dell'iterazione](#5-authcontext--cuore-delliterazione)
6. [`LoginPage` — 4 modalità in un componente](#6-loginpage--4-modalità-in-un-componente)
7. [`ResetPasswordPage`](#7-resetpasswordpage)
8. [`AppRouter` + `PendingApprovalScreen`](#8-approuter--pendingapprovalscreen)
9. [Layout root + PWA + Service Worker](#9-layout-root--pwa--service-worker)
10. [Security headers (`next.config.ts`)](#10-security-headers-nextconfigts)

---

## 1. Schema database (Supabase migration)

**File:** `scripts/sql/2026_initial_schema.sql` (da eseguire una volta nel SQL Editor di Supabase)
**Casi d'uso coperti:** UC-01, UC-02, UC-03, UC-05, UC-06
**Scopo:** crea la tabella `profiles`, gli enum del dominio, il trigger atomico `handle_new_user` e le RLS policies.

```sql
-- ─── ENUM ────────────────────────────────────────────────
create type user_role as enum ('maestro', 'allievo');
create type approval_status as enum ('pending', 'approved', 'rejected');

-- ─── TABLE profiles ──────────────────────────────────────
create table public.profiles (
  id              uuid primary key references auth.users(id) on delete cascade,
  role            user_role not null default 'allievo',
  full_name       text,
  first_name      text,
  last_name       text,
  email           text,
  phone           text,
  birth_date      date,
  photo_url       text,
  level           text default 'Principiante',
  ranking         text default 'Non classificato',
  active          boolean not null default true,
  approval_status approval_status not null default 'pending',
  approved_at     timestamptz,
  approved_by     uuid references auth.users(id),
  is_fictitious   boolean not null default false,
  created_at      timestamptz not null default now()
);

-- ─── TRIGGER handle_new_user (ADR-1-2) ───────────────────
-- Atomically creates the profile row when a new auth.users row is inserted.
-- Avoids race condition between supabase.auth.signUp() and an explicit
-- INSERT from the client (which would fail with duplicate key).
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = public
as $$
begin
  insert into public.profiles (
    id, role, full_name, first_name, last_name, email,
    birth_date, ranking, level, approval_status
  ) values (
    new.id,
    'allievo',
    new.raw_user_meta_data->>'full_name',
    new.raw_user_meta_data->>'first_name',
    new.raw_user_meta_data->>'last_name',
    new.email,
    (new.raw_user_meta_data->>'birth_date')::date,
    coalesce(new.raw_user_meta_data->>'ranking', 'Non classificato'),
    coalesce(new.raw_user_meta_data->>'level', 'Principiante'),
    'pending'
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- ─── RLS ─────────────────────────────────────────────────
alter table public.profiles enable row level security;

-- SELECT: ognuno vede il proprio profilo; il maestro vede tutti
create policy "profiles_select"
  on public.profiles for select
  using (
    auth.uid() = id
    or exists (
      select 1 from public.profiles p
      where p.id = auth.uid() and p.role = 'maestro'
    )
  );

-- UPDATE: ognuno aggiorna il proprio profilo (campi limitati lato app);
--         il maestro può aggiornare tutto (incluso approval_status)
create policy "profiles_update"
  on public.profiles for update
  using (
    auth.uid() = id
    or exists (
      select 1 from public.profiles p
      where p.id = auth.uid() and p.role = 'maestro'
    )
  );

-- INSERT: nessuno: gli unici INSERT avvengono via trigger handle_new_user
-- (nessuna policy = nessun accesso)
```

> **Nota ADR-1-1:** l'account maestro non si crea via signup. Viene creato manualmente:
> ```sql
> -- da eseguire una sola volta, dopo aver creato l'utente auth tramite la dashboard Supabase
> update public.profiles
> set role = 'maestro', approval_status = 'approved'
> where email = 'maestro@tennisbellusco.it';
> ```

---

## 2. Wrapper Supabase — 3 modalità di accesso

Next.js 16 con SSR richiede 3 client diversi: browser, server (Server Components / Route Handlers) e middleware/proxy. Sono incapsulati in 3 file dedicati.

### 2.1 Browser client

**File:** `lib/supabase/client.ts`
**Casi d'uso coperti:** UC-01..06 (lato client)

```typescript
import { createBrowserClient } from '@supabase/ssr';
import type { Database } from '../database.types';

/**
 * Browser-side Supabase client.
 *
 * Returns a fully typed client: every `.from('table').select(...)` call now
 * infers the correct `Row` / `Insert` / `Update` shape from `Database`.
 */
export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### 2.2 Server client (RSC + Route Handlers)

**File:** `lib/supabase/server.ts`
**Casi d'uso coperti:** UC-02 (verify OTP), UC-04 (reset via callback)

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';
import type { Database } from '../database.types';

/**
 * Server-side Supabase client (Server Components, Route Handlers, Server Actions).
 *
 * Follows the official @supabase/ssr contract for Next.js 16:
 *   - uses ONLY `getAll` / `setAll` (never `get` / `set` / `remove`)
 *   - the `setAll` try/catch swallows the "called from a Server Component"
 *     error: in that context cookie writes are not allowed but our proxy
 *     refreshes them, so it's safe to ignore.
 */
export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // setAll was called from a Server Component. Safe to ignore —
            // the proxy refreshes user sessions on every request.
          }
        },
      },
    }
  );
}
```

### 2.3 Middleware/proxy client

**File:** `lib/supabase/middleware.ts`
**Casi d'uso coperti:** sessione persistente (tutti i UC)

```typescript
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';
import type { Database } from '../database.types';

/**
 * Refreshes the Supabase auth session cookies on every matched request.
 *
 * Called from the root `proxy.ts` (Next.js 16's renamed Middleware). Must
 * call `supabase.auth.getUser()` at least once or the session will not be
 * refreshed and stale tokens will reach the page handlers.
 */
export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // IMPORTANT: DO NOT REMOVE auth.getUser()
  // This refreshes the session if expired and must be called
  // on every middleware/proxy request.
  await supabase.auth.getUser();

  return supabaseResponse;
}
```

### 2.4 Re-export di compatibilità

**File:** `lib/supabase.ts`
**Scopo:** import unico `import { supabase } from '@/lib/supabase'` per tutti i componenti client.

```typescript
// Re-export the browser client as 'supabase' for backward compatibility.
// All client-side components use this import.
import { createClient } from './supabase/client';

export const supabase = createClient();
```

---

## 3. Proxy Next.js 16 (ex middleware.ts)

**File:** `proxy.ts` (root del progetto)
**Casi d'uso coperti:** sessione persistente (tutti i UC)
**Novità Next.js 16:** il file `middleware.ts` storico è stato rinominato in `proxy.ts` (breaking change v16). La funzione esportata si chiama `proxy`.

```typescript
import { type NextRequest } from 'next/server';
import { updateSession } from '@/lib/supabase/middleware';

export async function proxy(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for the ones starting with:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - sw.js (service worker)
     * - icons/ (PWA icons)
     * - Static assets (svg, png, jpg, etc.)
     */
    '/((?!_next/static|_next/image|favicon.ico|sw\\.js|icons/|.*\\.(?:svg|png|jpg|jpeg|gif|webp|json|webmanifest)$).*)',
  ],
};
```

---

## 4. Route handler `/auth/callback`

**File:** `app/auth/callback/route.ts`
**Casi d'uso coperti:** UC-02 (verify OTP via link), UC-04 (reset password)
**Scopo:** punto di atterraggio dei link email. Gestisce sia il flusso PKCE (parametro `code`) sia il flusso OTP/recovery (parametri `token_hash` + `type`).

```typescript
import { NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';
import type { EmailOtpType } from '@supabase/supabase-js';

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const token_hash = searchParams.get('token_hash');
  const type = searchParams.get('type') as EmailOtpType | null;
  const next = searchParams.get('next') ?? '/';

  // PKCE flow: exchange code for session
  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  // Implicit / OTP flow: verify token hash
  if (token_hash && type) {
    const supabase = await createClient();
    const { error } = await supabase.auth.verifyOtp({ token_hash, type });
    if (!error) {
      // For password recovery, redirect to the reset-password page
      if (type === 'recovery') {
        return NextResponse.redirect(`${origin}/reset-password`);
      }
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  // If we get here, something went wrong
  return NextResponse.redirect(`${origin}/?error=auth_callback_error`);
}
```

---

## 5. `AuthContext` — cuore dell'iterazione

**File:** `contexts/AuthContext.tsx`
**Casi d'uso coperti:** UC-01, UC-02, UC-03 (con approval check), UC-05 (refreshProfile)
**Scopo:** unico stato globale dell'applicazione. Espone `useAuth()` con l'intero contratto di autenticazione.

### 5.1 Interfaccia pubblica del contesto

```typescript
interface AuthContextType {
  user: User | null;
  profile: Profile | null;
  session: Session | null;
  loading: boolean;
  isCoach: boolean;
  signIn: (email: string, password: string) => Promise<{ error: string | null }>;
  signUp: (input: SignUpInput) => Promise<{ error: string | null }>;
  verifySignupOtp: (email: string, token: string) => Promise<{ error: string | null }>;
  resendSignupOtp: (email: string) => Promise<{ error: string | null }>;
  signOut: () => Promise<void>;
  refreshProfile: () => Promise<void>;
}

export interface SignUpInput {
  firstName: string;
  lastName: string;
  /** ISO date string yyyy-MM-dd */
  birthDate: string;
  ranking: string;
  level?: string;
  email: string;
  password: string;
}
```

### 5.2 `fetchProfile` con gestione errori difensiva

```typescript
async function fetchProfile(userId: string): Promise<Profile | null> {
  try {
    console.log('[Auth] Fetching profile for', userId);
    const { data, error } = await supabase
      .from('profiles')
      .select('*')
      .eq('id', userId)
      .single();

    if (error) {
      console.error('[Auth] Profile query error:', error.message, error.code);
      return null;
    }

    console.log('[Auth] Profile loaded:', data?.email, data?.role);
    return data as Profile;
  } catch (err) {
    console.error('[Auth] Profile exception:', err);
    return null;
  }
}
```

### 5.3 `onAuthStateChange` con guard anti-deadlock (ADR-1-4)

```typescript
useEffect(() => {
  mountedRef.current = true;

  // IMPORTANT: onAuthStateChange callback must NOT be async and must NOT
  // call other Supabase functions directly. Doing so causes a deadlock
  // due to the internal lock mechanism in supabase-js.
  // Fix: use setTimeout(0) to defer Supabase calls outside the callback.

  const {
    data: { subscription },
  } = supabase.auth.onAuthStateChange((event, newSession) => {
    if (!mountedRef.current) return;

    if (event === 'SIGNED_OUT') {
      setUser(null); setProfile(null); setSession(null); setLoading(false);
      return;
    }

    if (newSession?.user) {
      // Set user/session immediately (no Supabase calls needed)
      setSession(newSession);
      setUser(newSession.user);

      // Defer profile fetch to avoid deadlock
      const userId = newSession.user.id;
      setTimeout(async () => {
        const p = await fetchProfile(userId);
        if (mountedRef.current) {
          setProfile(p);
          setLoading(false);
        }
      }, 0);
      return;
    }

    // No session (first load without login)
    setUser(null); setProfile(null); setSession(null); setLoading(false);
  });

  // Safety timeout — if no auth event fires at all (very rare)
  const timeout = setTimeout(() => {
    if (mountedRef.current && loading) {
      console.warn('[Auth] Timeout — no auth event in 12s, forcing login');
      setLoading(false);
    }
  }, 12000);

  return () => {
    mountedRef.current = false;
    clearTimeout(timeout);
    subscription.unsubscribe();
  };
}, []);
```

### 5.4 `signIn` con doppio gate di approval (UC-03)

```typescript
const signIn = async (email: string, password: string) => {
  setLoading(true);
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });

  if (error) {
    setLoading(false);
    // Friendlier message for unconfirmed email
    if (/email not confirmed/i.test(error.message)) {
      return {
        error: 'Devi prima verificare la tua email. Controlla la posta e inserisci il codice ricevuto.',
      };
    }
    return { error: error.message };
  }

  // Check approval status before letting them in
  if (data.user) {
    const p = await fetchProfile(data.user.id);
    if (p && p.role === 'allievo' && p.approval_status !== 'approved') {
      await supabase.auth.signOut({ scope: 'local' });
      setLoading(false);
      if (p.approval_status === 'rejected') {
        return {
          error: 'La tua registrazione è stata rifiutata dal maestro. Per chiarimenti contatta il club.',
        };
      }
      return {
        error: 'È necessario essere approvati dal maestro per poter accedere. Attendi la conferma.',
      };
    }
  }

  // onAuthStateChange(SIGNED_IN) will fire and handle profile loading
  return { error: null };
};
```

### 5.5 `signUp` che delega al trigger DB (UC-01, ADR-1-2)

```typescript
const signUp = async (input: SignUpInput) => {
  const { firstName, lastName, birthDate, ranking, level, email, password } = input;
  const cleanFirst = firstName.trim();
  const cleanLast = lastName.trim();
  const fullName = `${cleanFirst} ${cleanLast}`.trim();

  // The DB trigger handle_new_user reads these metadata fields and creates
  // the profile row in a single atomic step. Don't INSERT explicitly here:
  // it would race with the trigger and fail with duplicate key.
  const { error: signUpErr } = await supabase.auth.signUp({
    email,
    password,
    options: {
      data: {
        full_name: fullName,
        first_name: cleanFirst,
        last_name: cleanLast,
        birth_date: birthDate,
        ranking: ranking.trim() || 'Non classificato',
        level: level || 'Principiante',
      },
      emailRedirectTo: `${window.location.origin}/auth/callback`,
    },
  });

  if (signUpErr) return { error: signUpErr.message };
  return { error: null };
};
```

### 5.6 `verifySignupOtp` e `resendSignupOtp` (UC-02)

```typescript
const verifySignupOtp = async (email: string, token: string) => {
  const { error } = await supabase.auth.verifyOtp({
    email,
    token,
    type: 'signup',
  });
  if (error) return { error: error.message };
  return { error: null };
};

const resendSignupOtp = async (email: string) => {
  const { error } = await supabase.auth.resend({
    type: 'signup',
    email,
    options: {
      emailRedirectTo: `${window.location.origin}/auth/callback`,
    },
  });
  if (error) return { error: error.message };
  return { error: null };
};
```

---

## 6. `LoginPage` — 4 modalità in un componente

**File:** `app/login/LoginPage.tsx`
**Casi d'uso coperti:** UC-01, UC-02, UC-03, UC-04
**Scopo:** gestisce 4 stati di UI con una macchina a stati interna (`Mode`).
**Nota architetturale:** è il componente più grande dell'iterazione (~400 LOC), candidato per refactor in iterazioni successive.

### 6.1 Macchina a stati

```typescript
type Mode = 'login' | 'register' | 'verify-email' | 'forgot';

export function LoginPage() {
  const { signIn, signUp, verifySignupOtp, resendSignupOtp } = useAuth();
  const [mode, setMode] = useState<Mode>('login');
  // ...stati del form...
}
```

### 6.2 Cooldown timer per il resend OTP (UX anti-spam)

```typescript
const [resendCooldown, setResendCooldown] = useState(0);

useEffect(() => {
  if (resendCooldown <= 0) return;
  const t = setTimeout(() => setResendCooldown((s) => s - 1), 1000);
  return () => clearTimeout(t);
}, [resendCooldown]);
```

### 6.3 `handleRegister` con validazione completa (UC-01)

```typescript
const handleRegister = async (e: React.FormEvent) => {
  e.preventDefault();
  if (submitting) return;
  setError('');
  setSuccess('');

  // Validation chain with early-return
  if (!firstName.trim()) return setError('Inserisci il nome');
  if (!lastName.trim()) return setError('Inserisci il cognome');
  if (!birthDate) return setError('Inserisci la data di nascita');

  const birthTs = new Date(birthDate).getTime();
  if (Number.isNaN(birthTs)) return setError('Data di nascita non valida');
  const minTs = new Date('1900-01-01').getTime();
  const maxTs = Date.now();
  if (birthTs < minTs || birthTs > maxTs) {
    return setError('Inserisci una data di nascita valida');
  }
  if (!unranked && !ranking.trim()) {
    return setError('Inserisci la classifica FIT oppure spunta "Non classificato"');
  }
  if (password.length < 6) return setError('La password deve avere almeno 6 caratteri');
  if (password !== confirmPassword) return setError('Le password non coincidono');

  setSubmitting(true);
  const { error: err } = await signUp({
    firstName: firstName.trim(),
    lastName: lastName.trim(),
    birthDate,
    ranking: unranked ? 'Non classificato' : ranking.trim(),
    level: unranked ? level : 'Principiante',
    email: email.trim(),
    password,
  });
  if (err) { setError(err); setSubmitting(false); return; }

  setOtpCode('');
  setMode('verify-email');
  setSuccess(`Codice inviato a ${email.trim()}`);
  setResendCooldown(45);
  setSubmitting(false);
};
```

### 6.4 `handleVerifyOtp` con normalizzazione (UC-02)

```typescript
const handleVerifyOtp = async (e: React.FormEvent) => {
  e.preventDefault();
  if (submitting) return;
  setError('');
  // Strip any non-digit character (paste tolerance)
  const code = otpCode.replace(/\D/g, '');
  if (code.length !== 6) return setError('Il codice deve essere di 6 cifre');

  setSubmitting(true);
  const { error: err } = await verifySignupOtp(email.trim(), code);
  if (err) {
    setError(err);
    setSubmitting(false);
    return;
  }
  // On success, AuthContext will pick up SIGNED_IN and show PendingApprovalScreen.
};
```

### 6.5 `handleForgotPassword` (UC-04 — Atto 1)

```typescript
const handleForgotPassword = async (e: React.FormEvent) => {
  e.preventDefault();
  setError('');
  setSuccess('');
  if (!email.trim()) return setError('Inserisci la tua email');

  setSubmitting(true);
  const { error: resetError } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: `${window.location.origin}/auth/callback?next=/reset-password`,
  });
  if (resetError) setError(resetError.message);
  else setSuccess('Email di reset inviata. Controlla la posta e clicca sul link per reimpostare la password.');
  setSubmitting(false);
};
```

---

## 7. `ResetPasswordPage`

**File:** `app/reset-password/page.tsx`
**Casi d'uso coperti:** UC-04 (Atto 2 — reset effettivo dopo click sul link)
**Scopo:** form che permette di impostare la nuova password. Il form è disabilitato finché `getUser()` non conferma che la sessione di recovery è attiva.

### 7.1 Verifica della sessione di recovery

```typescript
const [sessionReady, setSessionReady] = useState(false);

useEffect(() => {
  let cancelled = false;
  // The session should already be set by the auth/callback route.
  // Verify we have a valid session.
  supabase.auth.getUser().then(({ data }) => {
    if (cancelled) return;
    if (data.user) {
      setSessionReady(true);
    } else {
      setError('Sessione non valida. Richiedi un nuovo link di reset.');
    }
  });
  return () => { cancelled = true; };
}, [supabase.auth]);
```

### 7.2 Submit con validazione e aggiornamento password

```typescript
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  setError('');

  if (password.length < 6) {
    setError('La password deve avere almeno 6 caratteri');
    return;
  }
  if (password !== confirmPassword) {
    setError('Le password non corrispondono');
    return;
  }

  setLoading(true);
  const { error: updateError } = await supabase.auth.updateUser({ password });
  if (updateError) setError(updateError.message);
  else setSuccess(true);
  setLoading(false);
};
```

---

## 8. `AppRouter` + `PendingApprovalScreen`

**File:** `app/page.tsx`
**Casi d'uso coperti:** UC-05 (schermata pending), routing per ruolo (parte di UC-03)
**Scopo:** decision tree che sceglie la vista in base allo stato del contesto auth.

### 8.1 `AppRouter` — albero decisionale

```typescript
function AppRouter() {
  const { user, profile, loading } = useAuth();

  if (loading) return <LoadingScreen />;
  if (!user) return <LoginPage />;
  if (!profile) return <ProfileNotFound userId={user.id} />;

  // Coaches always have access
  if (profile.role === 'maestro') return <CoachDashboard />;

  // Allievi: must be approved
  if (profile.approval_status === 'pending') return <PendingApprovalScreen rejected={false} />;
  if (profile.approval_status === 'rejected') return <PendingApprovalScreen rejected={true} />;

  return <StudentDashboard />;
}

export default function Home() {
  return (
    <AuthProvider>
      <AppRouter />
    </AuthProvider>
  );
}
```

### 8.2 `PendingApprovalScreen` (UC-05)

```typescript
function PendingApprovalScreen({ rejected }: { rejected: boolean }) {
  const { profile, signOut, refreshProfile } = useAuth();
  const [refreshing, setRefreshing] = useState(false);

  const handleRefresh = async () => {
    setRefreshing(true);
    await refreshProfile();
    setRefreshing(false);
  };

  return (
    <div className="login-bg flex items-center justify-center px-4 py-12">
      <div className="w-full max-w-[440px] relative z-10">
        <div className="card p-8 text-center animate-slide-up">
          {/* icona conditional rejected/pending */}
          <h2 className="text-xl font-bold text-gray-900 mb-2">
            {rejected ? 'Registrazione rifiutata' : 'In attesa di approvazione'}
          </h2>
          <p className="text-sm text-gray-600 leading-relaxed mb-6">
            {rejected
              ? 'La tua richiesta di registrazione è stata rifiutata dal maestro. Per chiarimenti contatta direttamente il club.'
              : 'La tua email è stata verificata. Ora il tuo account è in attesa di approvazione da parte del maestro: riceverai accesso non appena verrai approvato.'}
          </p>

          <div className="flex flex-col gap-2.5">
            {!rejected && (
              <Button variant="primary" size="lg" loading={refreshing} onClick={handleRefresh}>
                Verifica stato
              </Button>
            )}
            <Button variant="secondary" size="lg" onClick={signOut}>
              Esci
            </Button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 8.3 `ProfileNotFound` (fallback difensivo)

```typescript
function ProfileNotFound({ userId }: { userId: string }) {
  const { refreshProfile } = useAuth();

  const handleLogout = async () => {
    await supabase.auth.signOut();
  };
  const handleRetry = async () => {
    await refreshProfile();
  };

  // ...UI con bottoni "Riprova" e "Torna al login"...
}
```

---

## 9. Layout root + PWA + Service Worker

**File:** `app/layout.tsx`
**Scopo:** wrapper HTML, configurazione fonts, viewport, PWA meta + registrazione Service Worker.

```typescript
import type { Metadata, Viewport } from "next";
import { DM_Sans, Playfair_Display, DM_Mono } from "next/font/google";
import "./globals.css";

const dmSans = DM_Sans({ subsets: ["latin"], variable: "--font-sans", display: "swap" });
const playfairDisplay = Playfair_Display({
  subsets: ["latin"], variable: "--font-display",
  weight: ["400", "500", "600", "700"], display: "swap",
});
const dmMono = DM_Mono({
  subsets: ["latin"], variable: "--font-mono",
  weight: ["400", "500"], display: "swap",
});

export const viewport: Viewport = {
  width: "device-width",
  initialScale: 1,
  maximumScale: 1,
  themeColor: "#C41E3A", // club red
};

export const metadata: Metadata = {
  title: "Tennis Club Bellusco",
  description: "Gestione allievi e maestri - Tennis Club Bellusco",
  appleWebApp: {
    capable: true,
    statusBarStyle: "black-translucent",
    title: "TC Bellusco",
  },
  icons: {
    icon: "/icons/icon-192x192.png",
    apple: "/icons/icon-192x192.png",
  },
  other: { "mobile-web-app-capable": "yes" },
};

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="it" className={`h-full antialiased ${dmSans.variable} ${playfairDisplay.variable} ${dmMono.variable}`}>
      <head>
        <link rel="apple-touch-icon" href="/icons/icon-192x192.png" />
      </head>
      <body className="min-h-full flex flex-col font-sans">
        {children}
        <ServiceWorkerRegister />
      </body>
    </html>
  );
}

// Client component for SW registration (inline to avoid extra file)
function ServiceWorkerRegister() {
  return (
    <script
      dangerouslySetInnerHTML={{
        __html: `
          if ('serviceWorker' in navigator) {
            window.addEventListener('load', function() {
              navigator.serviceWorker.register('/sw.js', { scope: '/', updateViaCache: 'none' })
                .catch(function() {});
            });
          }
        `,
      }}
    />
  );
}
```

---

## 10. Security headers (`next.config.ts`)

**File:** `next.config.ts`
**Scopo:** applica header HTTP di sicurezza a tutte le risposte. Soddisfa RNF-04.

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        // Security headers for all routes
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
      {
        // Service worker: no caching, correct content type
        source: '/sw.js',
        headers: [
          { key: 'Content-Type', value: 'application/javascript; charset=utf-8' },
          { key: 'Cache-Control', value: 'no-cache, no-store, must-revalidate' },
          { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self'" },
        ],
      },
    ];
  },
};

export default nextConfig;
```

---

## Mappa: File ↔ Casi d'Uso

| File | UC-01 | UC-02 | UC-03 | UC-04 | UC-05 | UC-06 |
|---|---|---|---|---|---|---|
| `scripts/sql/2026_initial_schema.sql` | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| `lib/supabase/client.ts` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `lib/supabase/server.ts` | — | ✅ | — | ✅ | — | — |
| `lib/supabase/middleware.ts` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `proxy.ts` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `app/auth/callback/route.ts` | — | ✅ | — | ✅ | — | — |
| `contexts/AuthContext.tsx` | ✅ | ✅ | ✅ | — | ✅ | — |
| `app/login/LoginPage.tsx` | ✅ | ✅ | ✅ | ✅ | — | — |
| `app/reset-password/page.tsx` | — | — | — | ✅ | — | — |
| `app/page.tsx` (AppRouter + PendingApprovalScreen) | — | — | ✅ | — | ✅ | — |
| `app/layout.tsx` | — | — | — | — | — | — |
| `next.config.ts` | — | — | — | — | — | — |

> **Legenda:** ✅ = il file implementa direttamente quel UC. UC-06 (Approva allievo) è coperto a livello DB (RLS) e ha richiesto un componente UI lato maestro che verrà documentato insieme al cruscotto maestro nelle iterazioni successive.

---

## Note finali

- **No Repository Pattern in Iter. 1:** `AuthContext` parla direttamente con `supabase-js`. Il refactor verso `lib/repositories/` è pianificato per le iterazioni successive (vedi roadmap).
- **No test in Iter. 1:** la cartella `__tests__` e i file di configurazione Vitest sono pianificati come parte della consegna Iter. 1 ma non sono ancora presenti nel codice attuale. Vanno aggiunti prima della consegna formale (vedi documento riassuntivo, sezione 5).
- **No integrazione MatchFit:** ADR-0-6 confermato, demandato a iterazioni successive.
