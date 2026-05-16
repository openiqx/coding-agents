# Next.js Engineering Standards

Every change you make must follow these rules without exception.

---

## Stack

- **Framework**: Next.js 14+ with App Router
- **Language**: TypeScript — strict mode, no `any`
- **Styling**: Tailwind CSS — utility classes only
- **Server state**: TanStack Query (React Query) — all client-side data fetching
- **Client state**: Zustand — only for genuinely global UI state
- **Forms**: React Hook Form + Zod resolver
- **HTTP client**: Axios with a configured instance — never raw `fetch` on the client
- **Component library**: shadcn/ui — extend never override
- **Icons**: lucide-react only
- **Date handling**: date-fns — never moment.js
- **Validation**: Zod — shared between forms, API routes, and server actions
- **Testing**: Vitest + React Testing Library + MSW for API mocking
- **Linting**: ESLint + Prettier — zero warnings, CI fails on any lint error

---

## App Router fundamentals

### Server Components vs Client Components

This is the most important decision you make for every component.
Default to Server Components. Only add `'use client'` when you have a specific reason.

**Server Component** — the default. No directive needed.
Runs on the server. Has direct access to databases, file system, environment
variables. Cannot use browser APIs, event handlers, hooks, or state.

**Client Component** — add `'use client'` at the top of the file.
Runs in the browser (and is also pre-rendered on the server).
Required when the component uses: `useState`, `useEffect`, event handlers,
browser APIs, third-party libraries that use those things.

```tsx
// Server Component — default, no directive
// Can fetch data directly, access env vars, run server-only code
export default async function UsersPage() {
  const users = await db.query.users.findMany()
  return <UserList users={users} />
}

// Client Component — only when needed
'use client'
export function SearchInput({ onSearch }: SearchInputProps) {
  const [value, setValue] = useState('')
  return <input value={value} onChange={e => setValue(e.target.value)} />
}
```

**The rule**: push `'use client'` as far down the component tree as possible.
A page should almost never be a Client Component. The interactive leaf nodes
(a search input, a dropdown, a form) should be.

### The component split pattern

```tsx
// UsersPage.tsx — Server Component (fetches data)
export default async function UsersPage() {
  const users = await fetchUsers()
  return (
    <div>
      <PageHeader title="Users" />
      <UserTableClient users={users} />  {/* passes data down to client */}
    </div>
  )
}

// UserTableClient.tsx — Client Component (handles interaction)
'use client'
export function UserTableClient({ users }: { users: User[] }) {
  const [search, setSearch] = useState('')
  const filtered = users.filter(u => u.username.includes(search))
  return (
    <>
      <SearchInput value={search} onChange={setSearch} />
      <UserTable users={filtered} />
    </>
  )
}
```

---

## Project structure

```
src/
  app/                          # Next.js App Router — only routing concerns live here
    (auth)/                     # Route group — auth pages share layout, no URL segment
      login/
        page.tsx
      layout.tsx
    (dashboard)/                # Route group — protected pages
      layout.tsx                # checks auth, renders shell
      dashboard/
        page.tsx
      users/
        page.tsx                # list page
        [id]/
          page.tsx              # detail page
          edit/
            page.tsx
      loading.tsx               # Suspense fallback for the whole group
      error.tsx                 # Error boundary for the whole group
    api/                        # Route handlers (REST API endpoints)
      users/
        route.ts
        [id]/
          route.ts
    layout.tsx                  # root layout — html, body, providers
    not-found.tsx
    error.tsx

  features/                     # one folder per product domain
    users/
      components/               # components used only within this feature
      hooks/                    # client hooks used only within this feature
      actions/                  # server actions for this feature
        users.actions.ts
      api/                      # client-side API calls + TanStack Query hooks
        users.api.ts
        users.queries.ts
      schemas/                  # Zod schemas — shared between server and client
        users.schema.ts
      types/
        users.types.ts
      utils/
      index.ts                  # barrel — only export what other features need

  components/                   # shared UI components used across features
    ui/                         # shadcn/ui generated — never edit directly
    layout/                     # Shell, Sidebar, Header, PageContainer
    feedback/                   # ErrorBoundary, EmptyState, Spinner, Skeleton
    data-display/               # DataTable, StatCard, Badge
    forms/                      # shared form primitives

  hooks/                        # shared hooks used across multiple features
  lib/
    axios.ts                    # configured axios instance
    query-client.ts             # TanStack Query config
    utils.ts                    # cn() and cross-cutting utilities
    config.ts                   # typed env var access — nowhere else

  stores/                       # Zustand stores — genuinely global UI state only
  types/
    api.ts                      # ApiResponse, PaginatedResponse envelopes
    common.ts                   # shared domain types

  utils/
    format.ts                   # date, currency, number formatters
```

---

## Data fetching strategy

### Server Components fetch on the server

In Server Components, fetch data directly — no API call, no TanStack Query.
Use your ORM, database client, or call an internal service function directly.
This data is never cached by TanStack Query because it never touches the client cache.

```tsx
// app/(dashboard)/users/page.tsx
export default async function UsersPage() {
  // Direct DB/service call — runs on server, never exposed to client
  const users = await usersService.list({ page: 1, limit: 20 })
  return <UsersPageClient initialData={users} />
}
```

### TanStack Query for client-side data

Use TanStack Query when:
- Data needs to stay fresh in real-time (polling, refetch on focus)
- Data is fetched in response to user interaction (search, filter, pagination)
- Data is mutated and the UI must update immediately
- You need optimistic updates

Hydrate initial data from the server into TanStack Query to avoid a client-side
waterfall on first load:

```tsx
// Server Component
export default async function UsersPage() {
  const initialData = await usersService.list(defaultParams)

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UsersPageClient initialData={initialData} />
    </HydrationBoundary>
  )
}

// Client Component
'use client'
export function UsersPageClient({ initialData }: Props) {
  const { data } = useUsers(params, { initialData })
  // data is immediately available from initialData, then stays fresh
}
```

### Server Actions for mutations

Use Server Actions for form submissions and mutations that do not need
an intermediate API route. They run on the server, can access the DB directly,
and integrate cleanly with React Hook Form.

```ts
// features/users/actions/users.actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { createUserSchema } from '../schemas/users.schema'

export async function createUserAction(formData: unknown) {
  const parsed = createUserSchema.safeParse(formData)
  if (!parsed.success) {
    return { success: false, errors: parsed.error.flatten().fieldErrors }
  }

  await usersService.create(parsed.data)
  revalidatePath('/users')
  return { success: true }
}
```

### Route Handlers for REST API endpoints

Use Route Handlers (`app/api/`) when:
- An external service needs to call your backend
- A client-side library requires an HTTP endpoint (OAuth callbacks, webhooks)
- You need fine-grained control over headers, streaming, or response format

Every Route Handler follows the same pattern as a backend handler:
validate input → call service → return response envelope.

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { listUsersQuerySchema } from '@/features/users/schemas/users.schema'

export async function GET(req: NextRequest) {
  const parsed = listUsersQuerySchema.safeParse(
    Object.fromEntries(req.nextUrl.searchParams)
  )
  if (!parsed.success) {
    return NextResponse.json(
      { success: false, error: { code: 'VALIDATION_ERROR', fields: parsed.error.flatten().fieldErrors } },
      { status: 400 }
    )
  }

  const result = await usersService.list(parsed.data)
  return NextResponse.json({ success: true, data: result.data, meta: result.meta })
}
```

---

## Routing and pages

### File conventions

```
page.tsx          the UI for a route segment
layout.tsx        shared UI that wraps child segments — persists across navigation
loading.tsx       Suspense fallback — shown while page.tsx is loading
error.tsx         Error boundary — shown when page.tsx throws
not-found.tsx     shown when notFound() is called
route.ts          API Route Handler — no UI
```

### Route groups

Use route groups `(groupname)` to share layouts without adding URL segments:

```
app/
  (auth)/           # login, register, forgot-password — unauthenticated layout
    login/page.tsx
    layout.tsx      # minimal layout, no sidebar
  (dashboard)/      # all protected pages — authenticated layout with sidebar
    users/page.tsx
    layout.tsx      # checks auth, renders shell with sidebar
```

### Authentication guard in layout

Auth checking belongs in the `(dashboard)/layout.tsx` Server Component —
not in every page individually.

```tsx
// app/(dashboard)/layout.tsx
import { redirect } from 'next/navigation'
import { getSession } from '@/lib/auth'

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await getSession()
  if (!session) redirect('/login')

  return <AppShell user={session.user}>{children}</AppShell>
}
```

### Dynamic segments

```
app/users/[id]/page.tsx           →  /users/abc-123
app/users/[id]/edit/page.tsx      →  /users/abc-123/edit
app/[...slug]/page.tsx            →  catch-all
```

Always validate dynamic params before using them:

```tsx
export default async function UserPage({ params }: { params: { id: string } }) {
  const parsed = z.string().uuid().safeParse(params.id)
  if (!parsed.success) notFound()

  const user = await usersService.getById(parsed.data)
  if (!user) notFound()

  return <UserDetail user={user} />
}
```

---

## Caching and revalidation

### Understand the four caching layers

Next.js has four independent caches. Know which one applies to what you are doing.

**1. Request Memoisation** — within a single render pass, identical `fetch()` calls
are deduplicated automatically. Does not persist across requests.

**2. Data Cache** — `fetch()` responses are cached on the server between requests.
Persists until explicitly revalidated or the deployment changes.

**3. Full Route Cache** — statically rendered routes are cached as HTML + RSC payload.
Only for routes that do not use dynamic functions.

**4. Router Cache** — client-side cache of visited RSC payloads. Persists for the
browser session. Prefetched links are stored here.

### Revalidation

After a mutation, invalidate the relevant cache:

```ts
// Time-based — revalidate every N seconds
export const revalidate = 60  // in a layout or page

// On-demand — after a mutation in a Server Action or Route Handler
revalidatePath('/users')               // revalidate a specific path
revalidatePath('/users', 'layout')     // revalidate path and all children
revalidateTag('users')                 // revalidate all fetches tagged 'users'
```

Tag fetches so you can invalidate them precisely:

```ts
const users = await fetch('/api/users', { next: { tags: ['users'] } })
```

### Rules

- Never call `revalidatePath('/')` to invalidate everything — it is a blunt
  instrument that kills performance. Invalidate the specific path or tag.
- Dynamic data (user-specific, role-specific) must opt out of caching:
  `export const dynamic = 'force-dynamic'` or use `cookies()` / `headers()`
  which automatically make a route dynamic.
- Never cache sensitive or user-specific data in the Data Cache.

---

## Server Actions — rules

- Always validate input with Zod at the top of every action before doing anything.
- Return a typed result object — never throw from a Server Action that is called
  from a form (thrown errors become unhandled in the client).
- Call `revalidatePath` or `revalidateTag` after successful mutations.
- Never put sensitive logic (auth checks, permission checks) after the validation —
  check them first.

```ts
'use server'

export async function updateUserAction(id: string, formData: unknown) {
  // 1. Auth check first
  const session = await getSession()
  if (!session) return { success: false, error: 'Unauthenticated' }

  // 2. Validate input
  const parsed = updateUserSchema.safeParse(formData)
  if (!parsed.success) {
    return { success: false, errors: parsed.error.flatten().fieldErrors }
  }

  // 3. Execute
  await usersService.update(id, parsed.data)

  // 4. Revalidate
  revalidatePath(`/users/${id}`)

  return { success: true }
}
```

---

## Component rules

### Anatomy — always in this order

```tsx
// 1. Directive (if Client Component)
'use client'

// 2. Imports — external → internal → relative
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { Button } from '@/components/ui/button'
import { useUsers } from '@/features/users'

// 3. Types
interface UserListProps {
  initialData?: User[]
}

// 4. Component — named function declaration, never arrow at top level
export function UserList({ initialData }: UserListProps) {
  // 4a. Hooks
  const { data, isLoading, error } = useUsers({ initialData })

  // 4b. Derived state
  const activeUsers = data?.filter(u => u.isActive) ?? []

  // 4c. Handlers
  function handleSelect(id: string) { ... }

  // 4d. Early returns
  if (isLoading) return <UserListSkeleton />
  if (error) return <ErrorState error={error} />
  if (activeUsers.length === 0) return <EmptyState message="No active users." />

  // 4e. Render
  return (
    <ul>
      {activeUsers.map(user => (
        <UserCard key={user.id} user={user} onSelect={handleSelect} />
      ))}
    </ul>
  )
}
```

### Naming conventions

- Components: `PascalCase`
- Hooks: `camelCase` prefixed with `use`
- Handlers: `handle` prefix — `handleSubmit`, `handleDelete`
- Boolean props: `is`, `has`, `can` prefix — `isLoading`, `hasError`, `canEdit`
- Event props: `on` prefix — `onSelect`, `onDelete`
- Server Actions: `camelCase` with `Action` suffix — `createUserAction`
- Route Handlers: file is `route.ts`, functions are named HTTP verbs — `GET`, `POST`, `PATCH`, `DELETE`

### Never use React.FC

```tsx
// Wrong
const UserCard: React.FC<UserCardProps> = ({ user }) => { ... }

// Right
export function UserCard({ user }: UserCardProps) { ... }
```

---

## Forms

### React Hook Form + Zod — always for client forms

```tsx
'use client'

export function CreateUserForm({ onSuccess }: Props) {
  const { mutate, isPending } = useCreateUser()

  const form = useForm<CreateUserInput>({
    resolver: zodResolver(createUserSchema),
    defaultValues: { role: 'member' },
  })

  function handleSubmit(data: CreateUserInput) {
    mutate(data, {
      onSuccess: () => { form.reset(); onSuccess() },
    })
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(handleSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="username"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit" disabled={isPending}>
          {isPending ? 'Creating...' : 'Create'}
        </Button>
      </form>
    </Form>
  )
}
```

### Server Action forms

When using a Server Action directly with a form, use the `useActionState` hook
to handle the result and pending state:

```tsx
'use client'
import { useActionState } from 'react'
import { createUserAction } from '../actions/users.actions'

export function CreateUserForm() {
  const [state, formAction, isPending] = useActionState(createUserAction, null)

  return (
    <form action={formAction} className="space-y-4">
      <input name="username" />
      {state?.errors?.username && (
        <p className="text-sm text-red-600">{state.errors.username[0]}</p>
      )}
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  )
}
```

---

## State management

### Decision tree — use in this order

1. **URL state** (`useSearchParams`, `useRouter`) — pagination, filters, active tab,
   anything the user should be able to bookmark or share. Always first choice for
   shareable state.
2. **Server Component** — if state is only needed to render and never changes
   client-side, it is not state — it is a prop from the server.
3. **Local component state** (`useState`, `useReducer`) — if only this component
   needs it.
4. **TanStack Query** — all server data. Never copy server data into `useState`.
5. **Zustand** — genuinely global UI state only: sidebar open/closed, theme,
   notification queue, current user session on the client.

Never use `useContext` + `useReducer` to share state across components — use Zustand.
Never store server data in Zustand — use TanStack Query.

---

## TypeScript rules

- Strict mode is on. No `any`. No `as X` without a comment explaining why it is safe.
- Use `type` for object shapes and unions. `interface` only for declaration merging.
- Never use non-null assertion `!` without a guard three lines above.
- Use `unknown` instead of `any` when a type is genuinely unknown — then narrow it.
- Path alias `@/` maps to `src/`. Always use `@/`. Never relative paths across
  feature boundaries.
- `noUncheckedIndexedAccess` is on — guard every array and object access.

---

## Styling rules

### Tailwind only

No CSS files except `globals.css` for the Tailwind base and root CSS variables.
No CSS modules. No styled-components. No inline `style` props unless the value
is genuinely dynamic and cannot be expressed as a utility class.

### `cn()` for conditional classes — always

```tsx
import { cn } from '@/lib/utils'  // clsx + tailwind-merge

<div className={cn(
  'flex items-center px-4 py-2 rounded-md',
  isActive && 'bg-blue-600 text-white',
  isDisabled && 'opacity-50 cursor-not-allowed',
  className,   // always accept and spread external className last
)} />
```

Never concatenate class strings with template literals or `+`.

### Mobile first

Write base styles for mobile. Layer complexity upward with `sm:`, `md:`, `lg:`, `xl:`.
Never write desktop styles first and override downward.

---

## Metadata and SEO

Every page exports a `metadata` object or a `generateMetadata` function.
Never leave a page without a title.

```tsx
// Static metadata
export const metadata: Metadata = {
  title: 'Users — My App',
  description: 'Manage platform users.',
}

// Dynamic metadata
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const user = await usersService.getById(params.id)
  return {
    title: user ? `${user.username} — My App` : 'User not found',
  }
}
```

Root layout sets the default title template:
```tsx
export const metadata: Metadata = {
  title: { template: '%s — My App', default: 'My App' },
}
```

---

## Error handling

### error.tsx boundaries

Every route group and significant route segment has an `error.tsx`.
It must be a Client Component (`'use client'`).
It receives the error and a `reset` function to retry.

```tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <ErrorState
      heading="Something went wrong"
      description={error.message}
      action={<Button onClick={reset}>Try again</Button>}
    />
  )
}
```

### not-found.tsx

Every significant route segment has a `not-found.tsx`.
Call `notFound()` from Server Components when a resource does not exist.

```tsx
// In a page
const user = await usersService.getById(id)
if (!user) notFound()
```

### Route Handler errors

Every Route Handler wraps its body in try/catch and returns the standard
error envelope. Never let an unhandled error produce a 500 with no body.

```ts
export async function GET(req: NextRequest) {
  try {
    const result = await usersService.list()
    return NextResponse.json({ success: true, data: result })
  } catch (err) {
    const appError = isAppError(err) ? err : new InternalError()
    return NextResponse.json(
      { success: false, error: { code: appError.code, message: appError.message } },
      { status: appError.statusCode }
    )
  }
}
```

---

## Performance

### Images — always use next/image

```tsx
import Image from 'next/image'

<Image
  src={src}
  alt={alt}
  width={800}
  height={400}
  className="w-full h-auto object-cover"
  priority={isAboveFold}   // only for the largest above-the-fold image
/>
```

Never use `<img>` directly except for images with truly unknown dimensions.

### Fonts — always use next/font

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], display: 'swap' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  )
}
```

Never load fonts via a `<link>` tag in the document head.

### Dynamic imports for heavy Client Components

```tsx
import dynamic from 'next/dynamic'

const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,   // only when the component uses browser-only APIs
})
```

### Suspense boundaries

Wrap async Server Components in `<Suspense>` to stream them independently:

```tsx
export default function DashboardPage() {
  return (
    <div>
      <PageHeader title="Dashboard" />
      <Suspense fallback={<StatCardsSkeleton />}>
        <StatCards />          {/* async Server Component */}
      </Suspense>
      <Suspense fallback={<RecentActivitySkeleton />}>
        <RecentActivity />     {/* async Server Component */}
      </Suspense>
    </div>
  )
}
```

Never await multiple independent data fetches sequentially — use `Promise.all`
or parallel Suspense boundaries.

```ts
// Wrong — sequential, slow
const users = await fetchUsers()
const stats = await fetchStats()

// Right — parallel
const [users, stats] = await Promise.all([fetchUsers(), fetchStats()])
```

---

## Security

- Never expose secret environment variables to the client. Variables without
  the `NEXT_PUBLIC_` prefix are server-only. Never prefix a secret with `NEXT_PUBLIC_`.
- Auth cookies: `HttpOnly; Secure; SameSite=Strict` in production.
- Never trust `params`, `searchParams`, or any user-supplied value without
  validating with Zod first.
- Rate limit Route Handlers and Server Actions that perform auth or sensitive operations.
- Never use `dangerouslySetInnerHTML` without explicit sanitisation.
- Content Security Policy headers in `next.config.js`.

```ts
// next.config.js
const securityHeaders = [
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
]

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

---

## Environment variables

Accessed only through `src/lib/config.ts` — never `process.env.X` or
`import.meta.env.X` directly anywhere else in the codebase.

```ts
// src/lib/config.ts
export const config = {
  apiUrl: process.env['API_URL']!,
  appEnv: process.env['APP_ENV'] as 'development' | 'production',
  // Public (safe to expose to client)
  publicAppUrl: process.env['NEXT_PUBLIC_APP_URL']!,
} as const

// Validate at startup
if (!config.apiUrl) throw new Error('API_URL is required')
```

---

## Testing

### Unit tests

Pure functions, hooks, utilities, Zod schemas. No rendering.
Mock all I/O. Fast, no network calls.

### Component tests

Test behaviour — what the user sees and interacts with.
Never test internal state or implementation details.
Use `@testing-library/user-event` for interactions, never `.click()` directly.

```tsx
test('shows empty state when no users returned', async () => {
  server.use(
    http.get('/api/users', () =>
      HttpResponse.json({ success: true, data: [], meta: { total: 0 } })
    )
  )
  renderWithProviders(<UserList />)
  expect(await screen.findByText(/no users/i)).toBeInTheDocument()
})
```

### Server Action tests

Test Server Actions directly — call the function, assert the return value.

```ts
test('returns field errors for invalid input', async () => {
  const result = await createUserAction({ username: 'x' })
  expect(result.success).toBe(false)
  expect(result.errors?.username).toBeDefined()
})
```

### MSW for API mocking

All API calls in tests are intercepted by MSW at the network level.
Never mock axios, fetch, or modules directly. Handlers live in `tests/msw/handlers.ts`.

### renderWithProviders

Every component test uses a `renderWithProviders` helper that wraps with
`QueryClientProvider`, session context, and any other required providers.
Never set up providers manually in individual tests.

---

## Accessibility

- All interactive elements reachable by keyboard.
- All images have meaningful `alt` text. Decorative images have `alt=""`.
- All form inputs have associated `<label>` elements via shadcn's `FormLabel`.
- Colour contrast meets WCAG AA.
- Never use `<div onClick>` — use `<button>` or `<a>`.
- Icon-only buttons have `aria-label`.
- Modals trap focus and return it on close — shadcn Dialog handles this.
- Use semantic HTML: `<nav>`, `<main>`, `<header>`, `<section>`, `<article>`.

---

## Responsive design

Mobile first always. Write base styles for mobile, layer upward.

```
Base (no prefix)  →  0px+      mobile
sm:               →  640px+    mobile landscape
md:               →  768px+    tablet
lg:               →  1024px+   tablet landscape / small desktop
xl:               →  1280px+   desktop
2xl:              →  1536px+   wide desktop
```

Touch targets: minimum 44×44px on all interactive elements.
Input height: `h-11` on mobile, `h-9` on desktop.
Tables: horizontal scroll on mobile with sticky first column, or card list for simple tables.
Modals: bottom sheet on mobile, centred overlay on tablet+.
Sidebar: off-canvas on mobile, icon-only or full on desktop.
Never let content overflow or clip at any viewport width.
Use `min-w-0` on flex children that contain truncated text.

---

## What never to do

- Never use `'use client'` on a page or layout — push it to leaf components
- Never fetch data in a Client Component with `useEffect` + `useState` — use TanStack Query
- Never copy server state into `useState` — pass it as `initialData` to TanStack Query
- Never call `revalidatePath('/')` — invalidate specific paths or tags only
- Never expose secret env vars with `NEXT_PUBLIC_` prefix
- Never read `process.env` directly outside `src/lib/config.ts`
- Never use `<img>` — always `next/image`
- Never load fonts with a `<link>` tag — always `next/font`
- Never use `any` — use `unknown` and narrow it
- Never concatenate Tailwind classes with template literals — use `cn()`
- Never use array index as a React `key` for dynamic lists
- Never call hooks conditionally or inside loops
- Never put business logic in a component — extract it to a hook or Server Action
- Never use `React.FC`
- Never use `dangerouslySetInnerHTML` without explicit sanitisation
- Never store tokens or sensitive data in localStorage or sessionStorage
- Never ship console.log statements in production code
- Never mix icon libraries — lucide-react only
- Never use moment.js — date-fns only
- Never manually write cache key strings — always use query key factory functions
