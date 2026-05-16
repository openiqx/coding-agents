# Frontend Coding Standards — React

This document defines the coding standards, patterns, and constraints for this
React frontend codebase. Every change you make must follow these rules without
exception. When in doubt, read existing files first and match what is already there.

---

## Stack

- **Framework**: React 18+ with TypeScript (strict mode)
- **Build tool**: Vite
- **Styling**: Tailwind CSS — utility classes only, no custom CSS files except global reset
- **State management**:
  - Server state: TanStack Query (React Query) — all API data lives here
  - Client/UI state: Zustand — only for state that is truly global and client-only
  - Form state: React Hook Form + Zod resolver
- **Routing**: React Router v6 (file-based convention via a routes file)
- **HTTP client**: Axios with a configured instance — never raw fetch
- **Validation**: Zod — shared schemas between forms and API response parsing
- **Component library**: shadcn/ui as the base — extend, never override internals
- **Icons**: lucide-react only — no mixing icon libraries
- **Date handling**: date-fns — never moment.js, never raw Date manipulation outside utils
- **Testing**: Vitest + React Testing Library + MSW (Mock Service Worker) for API mocking
- **Linting**: ESLint + Prettier — zero warnings policy, CI fails on any lint error

---

## Project structure

```
src/
  app/
    router.tsx              # all route definitions in one place
    providers.tsx           # all context providers composed here, imported once in main.tsx
    query-client.ts         # TanStack Query client config

  features/                 # one folder per product feature — this is where most code lives
    auth/
      components/           # components used only within this feature
      hooks/                # custom hooks used only within this feature
      api/                  # API call functions + TanStack Query hooks
        auth.api.ts         # raw axios calls
        auth.queries.ts     # useQuery / useMutation hooks
      schemas/              # Zod schemas for this feature's forms and API responses
      types/                # TypeScript types for this feature
      utils/                # pure utility functions for this feature
      pages/                # page-level components (connected to router)
        LoginPage.tsx
      index.ts              # barrel — only export what other features need
    users/
      components/
      hooks/
      api/
      schemas/
      types/
      pages/
      index.ts
    dashboard/
      ...

  components/               # truly shared UI components used across multiple features
    ui/                     # shadcn/ui generated components — never edit these directly
    layout/                 # Shell, Sidebar, Header, PageContainer etc.
    feedback/               # Toast, ErrorBoundary, EmptyState, Spinner etc.
    data-display/           # DataTable, StatCard, Badge, etc.
    forms/                  # FormField wrapper, shared form elements

  hooks/                    # shared hooks used across multiple features
    use-debounce.ts
    use-pagination.ts
    use-local-storage.ts

  lib/
    axios.ts                # configured axios instance — one place, all interceptors
    query-client.ts         # TanStack Query defaults
    utils.ts                # cn() and other cross-cutting utilities

  stores/                   # Zustand stores — only truly global UI state
    auth.store.ts           # current user session context
    ui.store.ts             # sidebar open/closed, theme etc.

  types/
    api.ts                  # shared API envelope types (ApiResponse, PaginatedResponse)
    common.ts               # shared domain types used across features

  utils/
    format.ts               # date, currency, number formatters
    validation.ts           # shared Zod schemas (pagination, uuid, etc.)

  main.tsx                  # mounts app, imports providers.tsx only
```

---

## Component rules

### One component per file
Every component lives in its own file named with PascalCase matching the component name.
`UserCard.tsx` exports `UserCard`. No multiple components per file except
for small private sub-components used only by that file (and even then, prefer
extracting them when they grow beyond ~30 lines).

### Component anatomy — always in this order
```tsx
// 1. Imports (external libs → internal → relative)
import { useState } from 'react'
import { useQuery } from '@tanstack/react-query'
import { Button } from '@/components/ui/button'
import { useUserPermissions } from '@/features/auth'
import { UserCard } from './UserCard'

// 2. Types (props interface directly above the component that uses it)
interface UserListProps {
  stateId: string
  onSelect: (userId: string) => void
}

// 3. Component — named function declaration, never arrow function at top level
export function UserList({ stateId, onSelect }: UserListProps) {
  // 3a. Hooks — always at the top, never conditionally
  const { data, isLoading, error } = useUsers({ stateId })

  // 3b. Derived state / computed values
  const activeUsers = data?.filter(u => u.isActive) ?? []

  // 3c. Handlers — named functions, not inline arrows in JSX
  function handleSelect(userId: string) {
    onSelect(userId)
  }

  // 3d. Early returns for loading/error states
  if (isLoading) return <Spinner />
  if (error) return <ErrorState error={error} />
  if (activeUsers.length === 0) return <EmptyState message="No active users." />

  // 3e. Render
  return (
    <ul>
      {activeUsers.map(user => (
        <UserCard key={user.id} user={user} onSelect={handleSelect} />
      ))}
    </ul>
  )
}

// 4. Private sub-components used only in this file (if unavoidable)
// 5. Default export at the bottom if needed (prefer named exports)
```

### Naming conventions
- Components: `PascalCase`
- Hooks: `camelCase` prefixed with `use` — `useUsers`, `usePagination`
- Handlers: `handle` prefix — `handleSubmit`, `handleSelect`, `handleDelete`
- Boolean props: `is`, `has`, `can`, `should` prefix — `isLoading`, `hasError`, `canEdit`
- Event props: `on` prefix — `onSelect`, `onDelete`, `onSuccess`
- Constants: `SCREAMING_SNAKE_CASE`
- Everything else: `camelCase`

### Props rules
- Always define a named `interface` for props — never inline types in the function signature.
- Never use `React.FC` — use plain function declarations with explicit return types where needed.
- Destructure props in the function signature.
- Use sensible defaults in destructuring, not inside the function body.
- Keep prop interfaces small. If a component needs more than 6–8 props, it is doing
  too much — split it or lift the complexity into a hook.

---

## Hooks rules

### Custom hooks are the primary abstraction
Business logic, data fetching, and complex state belong in hooks — not in components.
A component that is hard to read is usually a component that should be a hook.

```tsx
// Wrong — logic in the component
export function UserList() {
  const [users, setUsers] = useState([])
  const [isLoading, setIsLoading] = useState(false)
  const [page, setPage] = useState(1)

  useEffect(() => {
    setIsLoading(true)
    fetchUsers({ page }).then(data => {
      setUsers(data)
      setIsLoading(false)
    })
  }, [page])

  // ...
}

// Right — logic extracted to a hook
export function UserList() {
  const { users, isLoading, page, setPage } = useUsers()
  // component is now only layout and interaction
}
```

### Hook rules
- A hook that fetches data uses TanStack Query — never `useState` + `useEffect` for
  async data.
- A hook must have a single clear responsibility. `useUserForm` handles form state.
  `useUsers` handles fetching the user list. They are not the same hook.
- Always return a plain object from hooks — never a tuple unless there are exactly
  two values and the semantics are obvious (like `useState`).
- Never call hooks inside conditions, loops, or nested functions.
- Prefix all custom hooks with `use`.

### The hook anatomy
```ts
export function useUsers(params: UseUsersParams) {
  // 1. Other hooks
  const queryClient = useQueryClient()

  // 2. Query / mutation
  const query = useQuery({
    queryKey: userKeys.list(params),
    queryFn: () => usersApi.list(params),
  })

  // 3. Derived values
  const activeCount = query.data?.filter(u => u.isActive).length ?? 0

  // 4. Handlers / mutations
  const { mutate: deleteUser } = useMutation({
    mutationFn: usersApi.delete,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: userKeys.all }),
  })

  // 5. Return — explicit object, no excess
  return {
    users: query.data ?? [],
    isLoading: query.isLoading,
    error: query.error,
    activeCount,
    deleteUser,
  }
}
```

---

## Data fetching rules

### TanStack Query is the only way to fetch server data
Never use `useEffect` + `useState` for API data. Never fetch in a component body.
All server state lives in TanStack Query cache.

### Query key factory — always
Every feature with API data defines a query key factory in its `api/` folder.
This is the single source of truth for cache invalidation.

```ts
// features/users/api/user-keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (params: ListUsersParams) => [...userKeys.lists(), params] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
}
```

### API layer separation
Raw axios calls live in `<feature>.api.ts`. TanStack Query hooks wrap them in
`<feature>.queries.ts`. Components only import from `.queries.ts`.

```ts
// features/users/api/users.api.ts — raw calls
export const usersApi = {
  list: (params: ListUsersParams): Promise<PaginatedResponse<User>> =>
    axios.get('/users', { params }).then(r => r.data),

  getById: (id: string): Promise<ApiResponse<User>> =>
    axios.get(`/users/${id}`).then(r => r.data),

  create: (body: CreateUserInput): Promise<ApiResponse<User>> =>
    axios.post('/users', body).then(r => r.data),

  update: (id: string, body: UpdateUserInput): Promise<ApiResponse<User>> =>
    axios.patch(`/users/${id}`, body).then(r => r.data),

  delete: (id: string): Promise<ApiResponse<void>> =>
    axios.delete(`/users/${id}`).then(r => r.data),
}
```

```ts
// features/users/api/users.queries.ts — TanStack Query hooks
export function useUsers(params: ListUsersParams) {
  return useQuery({
    queryKey: userKeys.list(params),
    queryFn: () => usersApi.list(params),
  })
}

export function useUser(id: string) {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: Boolean(id),
  })
}

export function useCreateUser() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: usersApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: userKeys.lists() })
    },
  })
}
```

### Axios instance — one place, all interceptors
```ts
// lib/axios.ts
const axios = Axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,   // always — sends the session cookie
  headers: { 'Content-Type': 'application/json' },
})

// Request interceptor — attach trace ID or any headers
axios.interceptors.request.use(config => {
  return config
})

// Response interceptor — unwrap envelope, handle auth errors globally
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // clear auth store, redirect to login
      useAuthStore.getState().clearUser()
      window.location.replace('/login')
    }
    return Promise.reject(error)
  }
)

export default axios
```

---

## Form rules

### React Hook Form + Zod resolver — always
Never manage form state with `useState`. Every form uses RHF + Zod.

```tsx
// features/users/schemas/create-user.schema.ts
export const createUserSchema = z.object({
  username: z
    .string()
    .min(3, 'Username must be at least 3 characters')
    .max(32, 'Username cannot exceed 32 characters')
    .regex(/^[a-z0-9_]+$/, 'Lowercase letters, numbers, and underscores only'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[a-zA-Z]/, 'Must contain at least one letter')
    .regex(/[0-9]/, 'Must contain at least one number'),
  role: z.enum(['government_admin', 'consultant', 'operator', 'regulator']),
  stateIds: z.array(z.string().uuid()).optional(),
  operatorId: z.string().uuid().optional(),
})
.superRefine((data, ctx) => {
  if (data.role === 'government_admin' && (!data.stateIds || data.stateIds.length !== 1)) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ['stateIds'],
      message: 'Government admin requires exactly one state.',
    })
  }
})

export type CreateUserInput = z.infer<typeof createUserSchema>
```

```tsx
// features/users/components/CreateUserForm.tsx
export function CreateUserForm({ onSuccess }: CreateUserFormProps) {
  const { mutate: createUser, isPending } = useCreateUser()

  const form = useForm<CreateUserInput>({
    resolver: zodResolver(createUserSchema),
    defaultValues: { role: 'government_admin', stateIds: [] },
  })

  function handleSubmit(data: CreateUserInput) {
    createUser(data, {
      onSuccess: () => {
        form.reset()
        onSuccess()
      },
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
              <FormControl>
                <Input {...field} />
              </FormControl>
              <FormMessage />   {/* renders Zod error automatically */}
            </FormItem>
          )}
        />
        <Button type="submit" disabled={isPending}>
          {isPending ? 'Creating...' : 'Create User'}
        </Button>
      </form>
    </Form>
  )
}
```

---

## State management rules

### Decision tree — use in this order

1. **Local component state** (`useState`, `useReducer`) — if only this component
   needs it and it does not need to survive unmount.
2. **URL state** (`useSearchParams`) — pagination, filters, active tab, anything
   the user should be able to bookmark or share.
3. **TanStack Query cache** — anything from the server. Do not copy server data
   into `useState`.
4. **Zustand store** — only for genuinely global client state: current user session,
   sidebar open/closed, theme, notification queue.

If you are reaching for `useContext` + `useReducer` to share state, use Zustand instead.
If you are reaching for Zustand to store server data, use TanStack Query instead.

### Zustand store rules
- One store per domain concern. `auth.store.ts` for session. `ui.store.ts` for UI state.
- Keep stores small and flat — no deeply nested state.
- Define selectors as separate functions — do not select the whole store.
- Never put async logic (API calls) in a Zustand store — that belongs in TanStack Query.

```ts
// stores/auth.store.ts
interface AuthState {
  user: AuthUser | null
  setUser: (user: AuthUser) => void
  clearUser: () => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    set => ({
      user: null,
      setUser: user => set({ user }),
      clearUser: () => set({ user: null }),
    }),
    { name: 'rotcs-auth' }
  )
)

// Selectors — always use these, never select the whole store
export const selectUser = (state: AuthState) => state.user
export const selectRole = (state: AuthState) => state.user?.role ?? null
export const selectIsAuthenticated = (state: AuthState) => state.user !== null
```

---

## TypeScript rules

- Strict mode is on. No `any`. No `as X` unless you are narrowing a type that
  TypeScript cannot infer and you add a comment explaining why.
- Use `type` for object shapes and unions. Use `interface` only when you
  intentionally want declaration merging (rare).
- All function parameters and return types must be explicitly typed when inference
  is not obvious.
- Use `unknown` instead of `any` when a type is genuinely unknown — then narrow it.
- Never use non-null assertion `!` unless you have just checked the value is
  non-null three lines above and TypeScript still cannot infer it.
- Prefer `readonly` arrays and objects for props and hook return values.
- Use discriminated unions for state that has multiple modes:
  ```ts
  type AsyncState<T> =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: T }
    | { status: 'error'; error: Error }
  ```
- Path aliases: `@/` maps to `src/`. Always use `@/` — never relative paths
  crossing feature boundaries.

---

## Styling rules

### Tailwind utility classes only
No separate `.css` files except `globals.css` for the Tailwind base import and
root CSS variables. No CSS modules. No styled-components. No inline `style` props
unless absolutely required (e.g. dynamic values that cannot be expressed as classes).

### Class organisation order — always
```tsx
// 1. Layout (display, position, flex/grid)
// 2. Sizing (width, height, padding, margin)
// 3. Visual (background, border, shadow, opacity)
// 4. Typography (font, text, line-height)
// 5. Interactive (cursor, pointer-events, transition)
// 6. Responsive modifiers last

<div className="flex items-center gap-3 w-full px-4 py-2 bg-white border border-gray-200 rounded-lg text-sm font-medium hover:bg-gray-50 transition-colors md:px-6" />
```

### The `cn()` helper — always for conditional classes
```tsx
import { cn } from '@/lib/utils'   // clsx + tailwind-merge

<button
  className={cn(
    'flex items-center px-4 py-2 rounded-md font-medium transition-colors',
    variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
    variant === 'ghost' && 'text-gray-600 hover:bg-gray-100',
    disabled && 'opacity-50 cursor-not-allowed',
    className,  // always accept and spread external className last
  )}
/>
```

Never concatenate class strings with template literals or `+`.

### Responsive design — mobile first
Write base styles for mobile. Use `sm:`, `md:`, `lg:` modifiers to add complexity
upward. Never write desktop-first and override for mobile.

---

## Routing rules

### Route definitions in one file
All routes are defined in `src/app/router.tsx`. No implicit file-based routing.

```tsx
// app/router.tsx
export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <RootErrorBoundary />,
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      {
        path: 'dashboard',
        element: <ProtectedRoute roles={['superadmin', 'government_admin', 'consultant', 'regulator']} />,
        children: [
          { index: true, element: <DashboardPage /> },
        ],
      },
      {
        path: 'users',
        element: <ProtectedRoute roles={['superadmin']} />,
        children: [
          { index: true, element: <UsersPage /> },
          { path: ':id', element: <UserDetailPage /> },
        ],
      },
    ],
  },
  {
    path: '/login',
    element: <LoginPage />,
  },
])
```

### Route protection
`ProtectedRoute` component checks the auth store and role. If the user is not
authenticated, redirects to `/login`. If the user lacks the required role,
renders a 403 page. Never check auth inside a page component.

```tsx
export function ProtectedRoute({ roles }: { roles?: UserRole[] }) {
  const user = useAuthStore(selectUser)

  if (!user) return <Navigate to="/login" replace />

  if (roles && !roles.includes(user.role)) {
    return <ForbiddenPage />
  }

  return <Outlet />
}
```

---

## Error handling rules

### Error boundaries — everywhere meaningful
Every page-level component is wrapped in an `ErrorBoundary`. The router's
`errorElement` catches routing errors. Feature-level error boundaries catch
render errors within a section without crashing the whole page.

```tsx
// At page level
export function UsersPage() {
  return (
    <ErrorBoundary fallback={<PageErrorState />}>
      <UserList />
    </ErrorBoundary>
  )
}
```

### API error handling
TanStack Query surfaces errors via `query.error`. Handle them in the component
using the `error` state — never in a catch block that swallows them.

Map API error codes to user-facing messages in one place:

```ts
// lib/error-messages.ts
export function getErrorMessage(error: unknown): string {
  if (isAxiosError(error)) {
    const code = error.response?.data?.error?.code
    const messages: Record<string, string> = {
      DUPLICATE_USERNAME: 'This username is already taken.',
      INVALID_CREDENTIALS: 'Incorrect username or password.',
      ACCOUNT_DISABLED: 'Your account has been disabled. Contact your administrator.',
      OUT_OF_SCOPE: 'You do not have access to this resource.',
    }
    return messages[code] ?? error.response?.data?.error?.message ?? 'Something went wrong.'
  }
  return 'An unexpected error occurred.'
}
```

### Toast notifications
Success and error feedback uses a toast system (shadcn/ui Sonner).
Call toast from mutation `onSuccess` / `onError` callbacks — never from render.

```ts
useMutation({
  mutationFn: usersApi.create,
  onSuccess: () => {
    toast.success('User created successfully.')
    queryClient.invalidateQueries({ queryKey: userKeys.lists() })
  },
  onError: (error) => {
    toast.error(getErrorMessage(error))
  },
})
```

---

## Performance rules

### Never optimise prematurely — but know when to act
Apply `useMemo`, `useCallback`, and `React.memo` only when you have a measured
performance problem. Premature memoisation adds noise and can cause bugs.

When to use:
- `React.memo` — a component re-renders visibly too often with the same props
- `useMemo` — an expensive computation runs on every render
- `useCallback` — a callback is a dependency in another hook's dependency array

### Code splitting — every page
Every page component is lazy-loaded:
```tsx
const UsersPage = lazy(() => import('@/features/users/pages/UsersPage'))
```

Wrap lazy routes in `<Suspense fallback={<PageSkeleton />}>` in the router.

### List rendering
Every list item must have a stable, unique `key` — never the array index unless
the list is static and will never be reordered.

Long lists (100+ items) use virtualisation — `@tanstack/react-virtual`.

### Images
Use `loading="lazy"` on all images below the fold.
Always specify `width` and `height` to prevent layout shift.

---

## Accessibility rules

These are not optional. Every component must meet them.

- All interactive elements are reachable by keyboard (`Tab`, `Enter`, `Space`, `Escape`).
- All images have meaningful `alt` text. Decorative images have `alt=""`.
- All form inputs have associated `<label>` elements — use shadcn's `FormLabel`.
- Colour contrast meets WCAG AA (4.5:1 for normal text, 3:1 for large text).
- Loading states use `aria-busy="true"` or appropriate ARIA live regions.
- Modals/dialogs trap focus and return focus on close — shadcn Dialog handles this.
- Use semantic HTML: `<nav>`, `<main>`, `<header>`, `<section>`, `<article>`,
  `<button>` (not `<div onClick>`).
- Icon-only buttons have `aria-label`.

```tsx
// Wrong
<div onClick={handleDelete} className="cursor-pointer">🗑</div>

// Right
<button onClick={handleDelete} aria-label="Delete user" className="...">
  <Trash2Icon className="h-4 w-4" aria-hidden="true" />
</button>
```

---

## Testing rules

### What to test and how

**Unit tests** — pure functions, hooks, utility functions, Zod schemas.
Use `renderHook` from React Testing Library for hooks.
Use `@testing-library/user-event` — never `.click()` directly.

**Component tests** — test behaviour, not implementation.
Assert what the user sees and can interact with.
Never test internal state. Never test that a function was called unless it
is an `on` prop passed from a parent.

**Integration tests** — test a feature end-to-end from user interaction
through to API call and rendered result. Use MSW to mock the API at the
network level — never mock modules or axios directly.

```tsx
// tests/features/users/UserList.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { http, HttpResponse } from 'msw'
import { server } from '@/tests/msw/server'
import { UserList } from '@/features/users/components/UserList'
import { renderWithProviders } from '@/tests/utils'

test('displays list of users', async () => {
  server.use(
    http.get('/v1/users', () =>
      HttpResponse.json({
        success: true,
        data: [{ id: '1', username: 'testuser', role: 'consultant', isActive: true }],
        meta: { page: 1, limit: 20, total: 1, totalPages: 1 },
      })
    )
  )

  renderWithProviders(<UserList />)

  expect(await screen.findByText('testuser')).toBeInTheDocument()
})

test('shows empty state when no users', async () => {
  server.use(
    http.get('/v1/users', () =>
      HttpResponse.json({ success: true, data: [], meta: { page: 1, limit: 20, total: 0, totalPages: 0 } })
    )
  )

  renderWithProviders(<UserList />)

  expect(await screen.findByText(/no users/i)).toBeInTheDocument()
})
```

### Testing utilities
Create a `renderWithProviders` helper that wraps components with all necessary
providers (QueryClientProvider, router, auth store) so tests do not need to
set up providers manually.

---

## Security rules

- `withCredentials: true` on the axios instance — the session cookie is HttpOnly
  and must be sent automatically, never read from JavaScript.
- Never store session tokens, JWTs, or sensitive data in `localStorage` or
  `sessionStorage` — they are accessible to XSS.
- Sanitise any content rendered as HTML — never use `dangerouslySetInnerHTML`
  unless the content is from a trusted, already-sanitised source.
- Never expose the API base URL in client-side error messages.
- Role-based rendering — hide UI elements the user cannot access. But always
  enforce access control on the backend too — frontend checks are UX, not security.

```tsx
// Conditional rendering based on role
const user = useAuthStore(selectUser)

{user?.role === 'superadmin' && (
  <Button onClick={handleDelete}>Delete User</Button>
)}
```

---

## Environment variables

All env vars are prefixed with `VITE_` (Vite requirement for client exposure).
Accessed only through a typed config object — never `import.meta.env.VITE_X` directly
outside `src/lib/config.ts`.

```ts
// src/lib/config.ts
export const config = {
  apiUrl: import.meta.env['VITE_API_URL'] as string,
  appEnv: import.meta.env['VITE_APP_ENV'] as 'development' | 'production',
} as const

// Validate at startup
if (!config.apiUrl) {
  throw new Error('VITE_API_URL is required')
}
```

---

## What never to do

- Never use `useEffect` to fetch data — use TanStack Query.
- Never store server state in `useState` — use TanStack Query cache.
- Never store server data in Zustand — use TanStack Query cache.
- Never use `any` — use `unknown` and narrow it.
- Never use non-null assertion `!` without a comment explaining why it is safe.
- Never concatenate Tailwind classes with template literals — use `cn()`.
- Never write custom CSS for layout or spacing that Tailwind can express.
- Never use array index as a React `key` for dynamic lists.
- Never call hooks conditionally or inside loops.
- Never put business logic in a component — extract it to a hook.
- Never use `React.FC` — use plain function declarations.
- Never import `import.meta.env` directly outside `src/lib/config.ts`.
- Never use `dangerouslySetInnerHTML` without explicit sanitisation.
- Never store tokens or sensitive data in localStorage or sessionStorage.
- Never check access control only on the frontend — it must be enforced on the backend.
- Never render a list without a stable, unique `key` on every item.
- Never use `<div onClick>` for interactive elements — use `<button>` or `<a>`.
- Never ship console.log statements — remove all debug logs before committing.
- Never mix icon libraries — use lucide-react only.
- Never use `moment.js` — use `date-fns`.
- Never manually invalidate query cache by string — always use key factory functions.

---

## UI and UX standards

These rules apply to every component, every screen, every interaction.
They are not optional polish — they are part of the definition of done.

---

### Visual consistency

**Spacing and sizing follow the Tailwind scale — no exceptions.**
Never use arbitrary values like `w-[437px]` or `mt-[13px]`. If the design
calls for a value not on the scale, round to the nearest step. Consistency
across the product matters more than pixel-perfect matching of a single screen.

**Typography hierarchy must be clear on every screen.**
Every page has exactly one `h1`. Section headings are `h2`. Sub-sections are `h3`.
Body copy is always the same base size. Never increase font size to create emphasis —
use weight (`font-semibold`) or colour instead.

**Colour usage is semantic, not decorative.**
- Blue → primary actions, links, interactive elements
- Green → success, active, healthy status
- Yellow/Amber → warnings, pending states, items needing attention
- Red → errors, destructive actions, failed states, inactive/suspended
- Gray → secondary text, borders, disabled states, metadata

Never use colour as the only signal — always pair with an icon or text label
for accessibility.

**Elevation and depth signal interactivity and hierarchy.**
Cards use `shadow-sm`. Modals use `shadow-lg`. Dropdowns use `shadow-md`.
Flat surfaces (tables, page backgrounds) use no shadow. Do not apply shadows
decoratively.

---

### Loading states

**Every data-dependent UI has a loading state.**
Never leave a blank space while data loads. Use skeleton loaders that match the
shape of the content that will appear — not a generic spinner in the middle of
a content area.

```tsx
// Wrong — generic spinner in content area
if (isLoading) return <div className="flex justify-center"><Spinner /></div>

// Right — skeleton that matches the content shape
if (isLoading) return <UserListSkeleton />
```

Skeleton components live in the same feature folder as the component they represent.
`UserListSkeleton` is in `features/users/components/UserListSkeleton.tsx`.

**Full-page loading is only acceptable on the initial app load.**
Subsequent navigations and data fetches use localised skeletons, not full-page
spinners. Use TanStack Query's `isLoading` (first load) vs `isFetching` (refetch)
to show the right indicator.

**Buttons show a loading state while their action is in flight.**
Never leave a button clickable while its mutation is pending — disable it and
show a loading indicator.

```tsx
<Button type="submit" disabled={isPending}>
  {isPending
    ? <><Loader2Icon className="mr-2 h-4 w-4 animate-spin" /> Creating...</>
    : 'Create User'
  }
</Button>
```

---

### Empty states

**Every list, table, and data view has an empty state.**
An empty state is not a blank screen. It tells the user why there is nothing
and what they can do about it. Every empty state has:
- An icon or illustration relevant to the content type
- A heading that states what is empty
- A supporting sentence that explains why or what to do
- A primary action if the user can resolve it

```tsx
// Good empty state
<EmptyState
  icon={<UsersIcon className="h-8 w-8 text-gray-400" />}
  heading="No users yet"
  description="Get started by creating the first user for this state."
  action={
    <Button onClick={openCreateModal}>
      <PlusIcon className="mr-2 h-4 w-4" />
      Create User
    </Button>
  }
/>
```

Empty states for filtered/searched results are different from empty states
for truly empty data. "No results for 'xyz'" needs a "Clear filters" action,
not a "Create" action.

---

### Error states

**Every error state tells the user what happened and what they can do.**
Never show a raw error message from the API or a generic "Something went wrong"
without a recovery path.

Every error state has:
- A clear, human-readable message (not an error code)
- A recovery action (Retry, Go back, Contact support)

```tsx
<ErrorState
  heading="Could not load users"
  description="There was a problem connecting to the server. This is usually temporary."
  action={<Button onClick={() => refetch()}>Try again</Button>}
/>
```

404 errors (resource not found) look different from 500 errors (server problem).
403 errors (no access) explain that the user lacks permission — not that the
resource does not exist.

---

### Feedback and confirmation

**Every user action gets feedback.**

| Action type | Feedback |
|---|---|
| Successful create | Toast success + form resets or modal closes |
| Successful update | Toast success + UI reflects new state immediately (optimistic or on invalidation) |
| Successful delete | Toast success + item removed from list |
| Failed action | Toast error with the specific reason |
| Form validation error | Inline field errors — never a toast for validation |

**Destructive actions always require confirmation.**
Delete, deactivate, revoke, suspend — any action that is hard or impossible to
undo must show a confirmation dialog before executing. The dialog names the specific
resource being affected — never just "Are you sure?".

```tsx
// Confirmation dialog copy
<AlertDialog>
  <AlertDialogHeader>
    <AlertDialogTitle>Delete user?</AlertDialogTitle>
    <AlertDialogDescription>
      This will permanently delete <strong>{user.username}</strong> and
      invalidate all their active sessions. This cannot be undone.
    </AlertDialogDescription>
  </AlertDialogHeader>
  <AlertDialogFooter>
    <AlertDialogCancel>Cancel</AlertDialogCancel>
    <AlertDialogAction
      onClick={handleDelete}
      className="bg-red-600 hover:bg-red-700"
    >
      Delete user
    </AlertDialogAction>
  </AlertDialogFooter>
</AlertDialog>
```

---

### Forms and inputs

**Labels are always visible — never placeholder-only.**
Placeholder text disappears when the user types. It cannot serve as the label.
Use `<FormLabel>` always. Placeholders may exist as examples but they are
supplementary — "e.g. lagos_admin" not "Enter username".

**Validation errors appear inline, immediately below the relevant field.**
Never show all errors in a banner at the top of the form. Show them under the
field they relate to. Show them on blur (when the user leaves the field), not
on every keystroke.

**Required fields are marked — but only if some fields in the form are optional.**
If every field is required, don't mark them all — it adds noise. Only mark
required fields when the form has a mix of required and optional.

**Disabled states must be visually obvious.**
Disabled inputs use `opacity-50 cursor-not-allowed`. Never disable a button or
input without making it visually clear that it is disabled.

**Submission errors from the API appear above the submit button**, not in a toast,
when they are form-level errors (e.g. "Username already taken" — this belongs
inline on the username field, not in a toast).

**Forms never reset on error.** The user should not have to re-enter data after
a submission fails. Preserve all field values. Only reset on success.

---

### Navigation and wayfinding

**The user always knows where they are.**
- Active nav items are clearly highlighted
- Page titles are always present and descriptive
- Breadcrumbs on deep pages (detail views, nested settings)
- Browser tab title reflects the current page: "Users — ROTCS" not just "ROTCS"

**Navigating away from an unsaved form warns the user.**
Use a `beforeunload` listener or React Router's `useBlocker` to warn before
navigating away from a form with unsaved changes.

**Links look like links. Buttons look like buttons.**
`<a>` tags navigate to a new URL. `<button>` tags perform an action on the
current page. Never use `<button>` for navigation. Never use `<a>` for actions.

---

### Tables and data displays

**Tables always show:**
- Column headers that are sortable where sorting is supported (visual sort indicator)
- Row count / pagination info ("Showing 1–20 of 143 users")
- A loading skeleton while data loads
- An empty state when there are no rows
- An error state when the fetch fails
- Row hover state for visual feedback

**Pagination is always visible** when there is more than one page.
Show current page, total pages, and next/prev controls.
Allow the user to change page size (20 / 50 / 100).

**Long text in table cells is truncated with an ellipsis**, not wrapped.
Full content is visible in a tooltip on hover.

**Status values use a consistent Badge component** with semantic colour:
active → green, suspended → amber, revoked/inactive → red, pending → blue.
Always pair colour with a text label.

---

### Responsive behaviour

**Every screen is usable on tablet (768px) and desktop (1024px+).**
This is a government/enterprise tool — full mobile support is secondary,
but nothing should break or become unusable at tablet width.

**Tables on narrow viewports** either scroll horizontally (preferred for
data-dense tables) or collapse to a card-based list layout. Never let a table
overflow and hide columns silently.

**Modals on narrow viewports** become bottom sheets or full-screen overlays.
Never let a modal clip off-screen.

**Sidebars collapse to an icon-only state or a hamburger menu** on tablet width.
The main content area always gets the full available width when the sidebar
is collapsed.

---

### Interaction and motion

**Motion is purposeful, not decorative.**
Transitions communicate state changes — a modal fading in, a skeleton giving
way to content, a row being removed from a list. They are not for style.

**Transition durations:**
- Micro-interactions (button states, hover effects): 100–150ms
- Element entering/leaving the DOM (modals, dropdowns, toasts): 200ms
- Page transitions: 150ms

Never animate anything longer than 300ms in a data-dense application.
Users are here to work, not watch animations.

**All interactive elements have a visible focus ring** — never remove the
browser default outline without replacing it with `focus-visible:ring-2`.
This is a keyboard navigation and accessibility requirement, not optional.

---

### Perceived performance

**Optimistic updates for fast feedback.**
When a mutation is highly unlikely to fail (toggling a status, updating a name),
update the UI immediately and roll back on error. Users should not wait for
a server round trip to see the result of a simple action.

```ts
useMutation({
  mutationFn: usersApi.update,
  onMutate: async (newData) => {
    await queryClient.cancelQueries({ queryKey: userKeys.detail(id) })
    const previous = queryClient.getQueryData(userKeys.detail(id))
    queryClient.setQueryData(userKeys.detail(id), newData)   // optimistic
    return { previous }
  },
  onError: (_err, _new, context) => {
    queryClient.setQueryData(userKeys.detail(id), context?.previous)  // rollback
    toast.error('Update failed.')
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: userKeys.detail(id) })
  },
})
```

**Stale-while-revalidate for perceived speed.**
TanStack Query serves cached data immediately while fetching fresh data in the
background. Configure `staleTime` appropriately — reference data (states, operators)
can be stale for 30 minutes. User lists can be stale for 30 seconds.

```ts
// In query-client.ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,       // 30 seconds default
      gcTime: 5 * 60_000,      // 5 minutes in cache after unmount
      retry: 1,                // one retry on failure
      refetchOnWindowFocus: true,
    },
  },
})
```

---

## Responsive design — complete specification

This is a government/enterprise web application. The primary viewport is desktop
(1280px+). Tablet (768px–1279px) must be fully functional. Mobile (320px–767px)
must be usable for read-heavy flows. Nothing breaks, clips, overflows, or becomes
unreachable at any viewport width.

### Breakpoint system

Always use Tailwind's breakpoint prefixes. Never write custom media queries.

```
Base (no prefix)  →  0px+      Mobile portrait — base layer, always written first
sm:               →  640px+    Mobile landscape
md:               →  768px+    Tablet portrait
lg:               →  1024px+   Tablet landscape / small desktop
xl:               →  1280px+   Desktop — primary target
2xl:              →  1536px+   Wide desktop
```

Mobile-first always. Write base styles for the smallest viewport.
Layer complexity upward with breakpoint prefixes.

```tsx
// Wrong — desktop first, overriding down
<div className="flex-row md:flex-col sm:flex-col" />

// Right — mobile first, scaling up
<div className="flex-col md:flex-row" />
```

---

### Layout system

**Page shell**

The application shell has three zones: sidebar, topbar, content area.
Each zone responds independently.

```tsx
// Shell layout
<div className="min-h-screen bg-gray-50">
  <Sidebar />                        {/* collapses at md, hidden at base */}
  <div className="flex flex-col md:pl-64 lg:pl-72 transition-all duration-200">
    <Topbar />
    <main className="flex-1 px-4 py-6 sm:px-6 lg:px-8 max-w-screen-2xl mx-auto w-full">
      <Outlet />
    </main>
  </div>
</div>
```

Content area padding:
- Mobile: `px-4 py-4`
- Tablet: `px-6 py-6`
- Desktop: `px-8 py-8`

Maximum content width: `max-w-screen-2xl` centred with `mx-auto`.
Never let content stretch full-width on ultrawide monitors.

**Sidebar**

Three states across breakpoints:

| Viewport | Default state | Toggle |
|---|---|---|
| Mobile (`< md`) | Hidden completely (off-canvas) | Hamburger button in topbar → slides in as overlay with backdrop |
| Tablet (`md`) | Icon-only (collapsed, 64px wide) | Toggle button → expands to full width (256px) over content |
| Desktop (`xl`) | Full width (256px), always visible | Toggle → collapses to icon-only, content area expands |

```tsx
// Sidebar responsive classes
<aside className={cn(
  // Base positioning
  'fixed inset-y-0 left-0 z-50 flex flex-col bg-white border-r border-gray-200',
  'transition-all duration-200 ease-in-out',
  // Mobile: off-canvas, slide in when open
  '-translate-x-full md:translate-x-0',
  isOpen && 'translate-x-0',
  // Width
  'w-64',
  // Tablet collapsed
  isCollapsed && 'md:w-16',
)}>
```

Mobile sidebar overlay backdrop:
```tsx
{isMobileOpen && (
  <div
    className="fixed inset-0 z-40 bg-black/50 md:hidden"
    onClick={closeMobileSidebar}
  />
)}
```

**Grid layouts**

Use responsive grid columns — never fixed column counts:

```tsx
// Stat cards — 1 col mobile, 2 col tablet, 4 col desktop
<div className="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-4 gap-4 lg:gap-6" />

// Form layout — single col mobile, two col tablet+
<div className="grid grid-cols-1 md:grid-cols-2 gap-4" />

// Dashboard panels — stack mobile, side-by-side desktop
<div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <div className="lg:col-span-2">...</div>   {/* main panel */}
  <div>...</div>                              {/* sidebar panel */}
</div>
```

---

### Navigation — responsive behaviour

**Mobile navigation (hamburger menu)**

The sidebar is hidden on mobile. A hamburger button in the topbar opens it as
an overlay drawer from the left. The drawer covers the full viewport height.
Tapping the backdrop or a nav item closes it.

The topbar on mobile shows:
- Hamburger button (left)
- App logo / page title (centre)
- User avatar / notification icon (right)

```tsx
// Mobile topbar
<header className="sticky top-0 z-30 flex h-14 items-center gap-4 border-b bg-white px-4 md:hidden">
  <button onClick={toggleMobileSidebar} aria-label="Open menu">
    <MenuIcon className="h-5 w-5" />
  </button>
  <span className="font-semibold text-sm">ROTCS</span>
  <div className="ml-auto flex items-center gap-2">
    <UserAvatarMenu />
  </div>
</header>
```

**Breadcrumbs**

Breadcrumbs truncate on mobile. Only the immediate parent and current page
are shown on small viewports. Full path on desktop.

```tsx
<nav className="flex" aria-label="Breadcrumb">
  {/* On mobile: show only last 2 segments */}
  <ol className="flex items-center gap-1 text-sm text-gray-500">
    <li className="hidden md:flex items-center gap-1">
      <Link to="/">Home</Link>
      <ChevronRight className="h-3 w-3" />
    </li>
    <li className="hidden md:flex items-center gap-1">
      <Link to="/users">Users</Link>
      <ChevronRight className="h-3 w-3" />
    </li>
    <li className="text-gray-900 font-medium">Edit User</li>
  </ol>
</nav>
```

---

### Tables — responsive behaviour

Data tables are the most complex responsive challenge. Three strategies — choose
based on column count and data density:

**Strategy 1: Horizontal scroll (default for data-dense tables)**
The table scrolls horizontally on narrow viewports. A sticky first column
keeps context while scrolling. Use for tables with 5+ columns.

```tsx
<div className="overflow-x-auto -mx-4 sm:mx-0 rounded-lg border border-gray-200">
  <table className="min-w-full divide-y divide-gray-200">
    <thead>
      <tr>
        {/* Sticky first column */}
        <th className="sticky left-0 z-10 bg-gray-50 px-4 py-3 text-left text-xs font-semibold text-gray-500 uppercase">
          Username
        </th>
        <th className="px-4 py-3 whitespace-nowrap ...">Role</th>
        <th className="px-4 py-3 whitespace-nowrap ...">State</th>
        <th className="px-4 py-3 whitespace-nowrap ...">Status</th>
        <th className="px-4 py-3 whitespace-nowrap ...">Last login</th>
        <th className="px-4 py-3 whitespace-nowrap ...">Actions</th>
      </tr>
    </thead>
  </table>
</div>
```

**Strategy 2: Priority columns (hide low-priority columns on small viewports)**
For tables with 4–6 columns where some columns are secondary information.
Columns are hidden progressively as viewport narrows.

```tsx
<th className="hidden lg:table-cell ...">Last login</th>    {/* desktop only */}
<th className="hidden md:table-cell ...">Created at</th>    {/* tablet+ */}
<th className="...">Username</th>                           {/* always visible */}
<th className="...">Status</th>                             {/* always visible */}
```

**Strategy 3: Card list (for simple tables on mobile)**
On mobile, each table row becomes a card. On tablet+, the normal table renders.
Use for tables with 3–4 columns where cards make semantic sense.

```tsx
export function UserList({ users }: UserListProps) {
  return (
    <>
      {/* Mobile: card list */}
      <ul className="md:hidden space-y-3">
        {users.map(user => <UserCard key={user.id} user={user} />)}
      </ul>
      {/* Tablet+: table */}
      <div className="hidden md:block overflow-x-auto">
        <UserTable users={users} />
      </div>
    </>
  )
}
```

**Table toolbar on mobile**

The table toolbar (search + filters + actions) stacks vertically on mobile
and sits in a row on tablet+.

```tsx
<div className="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between mb-4">
  <div className="flex flex-col gap-2 sm:flex-row sm:items-center">
    <SearchInput className="w-full sm:w-64" />
    <FilterDropdown />
  </div>
  <Button className="w-full sm:w-auto">
    <PlusIcon className="mr-2 h-4 w-4" />
    Create User
  </Button>
</div>
```

**Pagination on mobile**

On mobile, show only Prev / page indicator / Next. Hide numbered page buttons.

```tsx
<div className="flex items-center justify-between">
  <Button variant="outline" size="sm" disabled={page === 1} onClick={prevPage}>
    <ChevronLeftIcon className="h-4 w-4 mr-1" /> Prev
  </Button>
  {/* Page numbers — hidden on mobile */}
  <div className="hidden sm:flex items-center gap-1">
    {pageNumbers.map(n => <PageButton key={n} page={n} current={page} />)}
  </div>
  {/* Mobile page indicator */}
  <span className="text-sm text-gray-500 sm:hidden">
    {page} / {totalPages}
  </span>
  <Button variant="outline" size="sm" disabled={page === totalPages} onClick={nextPage}>
    Next <ChevronRightIcon className="h-4 w-4 ml-1" />
  </Button>
</div>
```

---

### Forms — responsive behaviour

**Form layout**

Single column on mobile. Two columns on tablet+ for long forms where fields
logically pair (first name / last name, start date / end date, etc.).

```tsx
<form className="space-y-4">
  {/* Always full width */}
  <FormField name="username" ... />

  {/* Two column on tablet+ */}
  <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
    <FormField name="firstName" ... />
    <FormField name="lastName" ... />
  </div>

  {/* Role + state assignment side by side on desktop */}
  <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
    <FormField name="role" ... />
    <FormField name="stateId" ... />
  </div>
</form>
```

**Form action buttons**

On mobile, action buttons are full width and stacked. On tablet+, they are
inline and right-aligned.

```tsx
<div className="flex flex-col-reverse gap-2 sm:flex-row sm:justify-end pt-4 border-t">
  <Button variant="outline" className="w-full sm:w-auto" onClick={onCancel}>
    Cancel
  </Button>
  <Button type="submit" className="w-full sm:w-auto" disabled={isPending}>
    {isPending ? 'Saving...' : 'Save changes'}
  </Button>
</div>
```

Note `flex-col-reverse` on mobile — the primary action appears at the top
(visually bottom in DOM order) so it is within thumb reach.

---

### Modals and dialogs — responsive behaviour

**On mobile, modals become bottom sheets.**
Full-width, anchored to the bottom of the screen, with a drag handle visual.
Slides up from the bottom. Can be dismissed by swiping down or tapping the backdrop.

**On tablet+, modals are centred overlays** with a maximum width.

```tsx
<DialogContent className={cn(
  // Mobile: bottom sheet
  'fixed bottom-0 left-0 right-0 top-auto translate-y-0 rounded-t-2xl rounded-b-none max-h-[90dvh]',
  // Tablet+: centred modal
  'sm:top-1/2 sm:left-1/2 sm:bottom-auto sm:-translate-x-1/2 sm:-translate-y-1/2',
  'sm:rounded-lg sm:max-w-lg sm:max-h-[85vh]',
  // Always
  'overflow-y-auto',
)}>
```

**Modal max widths by type:**
- Confirmation dialogs: `max-w-md`
- Single-entity forms (create/edit user): `max-w-lg`
- Complex forms (multi-step, rich content): `max-w-2xl`
- Data preview / detail panels: `max-w-3xl`

**Full-screen modals on mobile** for complex forms that would feel cramped
as a bottom sheet:

```tsx
<DialogContent className="sm:max-w-2xl h-full sm:h-auto rounded-none sm:rounded-lg">
```

---

### Cards and panels — responsive behaviour

**Stat cards** stack to a single column on mobile, 2-up on tablet, 4-up on desktop:

```tsx
<div className="grid grid-cols-1 sm:grid-cols-2 xl:grid-cols-4 gap-4">
  <StatCard title="Total Operators" value={142} trend="+3 this month" />
</div>
```

**Detail panels** (e.g. user detail page) stack vertically on mobile and
split into main + sidebar on desktop:

```tsx
<div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <div className="lg:col-span-2 space-y-6">
    <UserInfoCard user={user} />
    <UserActivityCard userId={user.id} />
  </div>
  <div className="space-y-6">
    <UserAssignmentsCard user={user} />
    <UserSessionsCard userId={user.id} />
  </div>
</div>
```

---

### Touch and pointer targets

**Minimum touch target size is 44×44px** (Apple HIG and WCAG 2.5.5).
This applies to all interactive elements on mobile — buttons, links, icon
buttons, checkboxes, radio buttons, select inputs.

```tsx
// Icon button — visually 20px icon but touch target is 44px minimum
<button className="flex items-center justify-center h-11 w-11 rounded-md hover:bg-gray-100">
  <TrashIcon className="h-5 w-5 text-gray-500" />
</button>
```

**Row actions in tables on mobile** become a "More actions" menu (`...` button)
rather than showing multiple action buttons inline. Inline action buttons are
only shown on `lg:` and above.

```tsx
<td>
  {/* Mobile: single actions menu */}
  <div className="lg:hidden">
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="sm">
          <MoreHorizontalIcon className="h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem onClick={() => onEdit(user)}>Edit</DropdownMenuItem>
        <DropdownMenuItem onClick={() => onDelete(user)} className="text-red-600">Delete</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  </div>
  {/* Desktop: inline action buttons */}
  <div className="hidden lg:flex items-center gap-2">
    <Button variant="ghost" size="sm" onClick={() => onEdit(user)}>Edit</Button>
    <Button variant="ghost" size="sm" className="text-red-600" onClick={() => onDelete(user)}>Delete</Button>
  </div>
</td>
```

**Input heights on mobile** use `h-11` (44px) for comfortable touch interaction.
Desktop inputs use `h-9` (36px) default.

```tsx
<Input className="h-11 md:h-9" {...field} />
```

---

### Typography — responsive scaling

Headings scale down on mobile. Use responsive text size classes — never a
fixed size that is too large for mobile.

```tsx
// Page heading
<h1 className="text-xl font-semibold text-gray-900 sm:text-2xl lg:text-3xl" />

// Section heading
<h2 className="text-base font-semibold text-gray-900 sm:text-lg" />

// Card title
<h3 className="text-sm font-semibold text-gray-900 sm:text-base" />

// Body text — never changes
<p className="text-sm text-gray-600" />

// Table header — never changes
<th className="text-xs font-semibold text-gray-500 uppercase" />
```

Long strings (usernames, email addresses, URLs) must truncate on narrow viewports:

```tsx
<span className="truncate max-w-[120px] sm:max-w-[200px] lg:max-w-none block">
  {user.email}
</span>
```

---

### Images and media — responsive behaviour

Always use responsive image sizing. Never fixed pixel widths on images.

```tsx
// Responsive image
<img
  src={src}
  alt={alt}
  className="w-full h-auto object-cover"
  loading="lazy"
  width={800}
  height={400}
/>

// Avatar — fixed size is fine since it is a known small UI element
<img
  src={avatarUrl}
  alt={`${username} avatar`}
  className="h-8 w-8 rounded-full object-cover"
/>
```

---

### Overflow and clipping — zero tolerance

The following must never occur at any viewport width:
- Horizontal scrollbar on the `<body>` (except inside an intentionally scrollable container)
- Content clipped by the viewport edge
- Buttons or inputs extending beyond their container
- Modal content overflowing off-screen
- Text overflowing its container without truncation or wrapping

**Prevention rules:**
- Every container that could hold dynamic-length content uses `overflow-hidden`,
  `truncate`, or `break-words` as appropriate.
- Fixed-width elements inside flex or grid containers use `min-w-0` so they
  can shrink below their natural width.

```tsx
// Common culprit — flex child with long text does not shrink
// Wrong
<div className="flex items-center gap-2">
  <UserIcon className="h-4 w-4" />
  <span>{user.email}</span>
</div>

// Right — min-w-0 allows the span to shrink and truncate
<div className="flex items-center gap-2 min-w-0">
  <UserIcon className="h-4 w-4 shrink-0" />
  <span className="truncate">{user.email}</span>
</div>
```

---

### Focus management on mobile

When a modal or drawer opens, focus moves to the first focusable element inside it.
When it closes, focus returns to the element that triggered it. shadcn/ui Dialog
and Sheet handle this automatically — do not override this behaviour.

When the mobile sidebar opens, focus traps inside the sidebar.
When it closes, focus returns to the hamburger button.

---

### Viewport meta tag

Always present in `index.html`. Never add `user-scalable=no` — users must be
able to zoom for accessibility.

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

---

### Testing responsiveness

Every new component must be visually verified at these widths before it is
complete: 375px (iPhone SE), 768px (iPad), 1280px (desktop), 1536px (wide).

Use Tailwind's responsive prefix as a checklist — if a layout has `md:` modifiers,
manually verify it at 767px (just below) and 768px (at the breakpoint).

In component tests, use `window.resizeTo` or `matchMedia` mocks to test
responsive hook behaviour where applicable.

---

### Summary — responsive rules at a glance

| Element | Mobile | Tablet | Desktop |
|---|---|---|---|
| Sidebar | Off-canvas drawer | Icon-only or toggle | Full width, persistent |
| Navigation | Hamburger → overlay | Collapsed sidebar | Full sidebar |
| Page padding | px-4 py-4 | px-6 py-6 | px-8 py-8 |
| Grid columns | 1 col | 2 col | 3–4 col |
| Tables | Horizontal scroll or cards | Horizontal scroll, fewer columns | Full columns |
| Table actions | `...` dropdown menu | `...` dropdown menu | Inline buttons |
| Modals | Bottom sheet | Centred overlay | Centred overlay |
| Forms | Single column | 1–2 column | 2 column |
| Form buttons | Full width, stacked, reversed | Inline, right-aligned | Inline, right-aligned |
| Touch targets | min 44×44px | min 44×44px | min 32×32px |
| Input height | h-11 (44px) | h-11 or h-9 | h-9 (36px) |
| Pagination | Prev / n of total / Next | Prev / numbers / Next | Prev / numbers / Next |
| Typography | Smaller scale | Mid scale | Full scale |
| Long text | Truncate with tooltip | Truncate or wrap | Full or truncate |
