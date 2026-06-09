---
name: nextjs-best-practices
description: Next.js App Router patterns reverse-engineered from Vercel's open-source eval suite (vercel/next-evals-oss). Each rule names the specific eval it derives from, the anti-pattern penalized, and the idiomatic answer. Use when writing, reviewing, or refactoring Next.js 15/16 App Router code — covers server components vs client, data fetching, Server Actions, `'use cache'`, `updateTag`/`revalidateTag`, async `cookies()`/`headers()`/`params`, `next/link`/`next/image`/`next/font`, `forbidden()`/`unauthorized()`, middleware/proxy, PPR, View Transitions, and the parallel-await/derive-don't-sync class of bugs.
---

# Next.js Best Practices (Reverse-Engineered from `vercel/next-evals-oss`)

These are the patterns that Vercel's open-source evaluation suite tests AI agents against. Each section names the eval it derives from, the anti-pattern being penalized, and the idiomatic answer the hidden `EVAL.ts` is checking for.

Confidence is **high** unless marked otherwise. Where the eval name is ambiguous (e.g. `unstable-instant`), the entry is marked _Inferred_ and should be sanity-checked against current docs before adopting.

Target Next.js version: **15.x and 16.x** (App Router).

---

## 1. App Router fundamentals

### 1.1 Use App Router, not Pages Router

_Evals: `agent-000-app-router-migration-simple`, `agent-030-app-router-migration-hard`_

- Pages live in `app/` as `page.tsx`, not `pages/index.tsx`.
- Every route segment can have a `layout.tsx`; the root `app/layout.tsx` is mandatory and must render `<html>` and `<body>`.
- Use the `metadata` export (or `generateMetadata`) instead of `next/head`.
- Use `next/navigation` (`useRouter`, `usePathname`, `useSearchParams`) — never the legacy `next/router`.

```tsx
// app/layout.tsx
export const metadata = { title: 'My App' }

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

### 1.2 Don't use `getServerSideProps` / `getStaticProps`

_Eval: `agent-023-avoid-getserversideprops`_

In App Router, server components fetch data directly. Both legacy data-loader exports are dead.

```tsx
// ❌ Pages Router
export async function getServerSideProps() {
  return { props: { user: await fetchUser() } }
}

// ✅ App Router — fetch inline in a server component
export default async function Page() {
  const user = await fetchUser()
  return <Profile user={user} />
}
```

---

## 2. Data fetching

### 2.1 Never `fetch` inside `useEffect`

_Eval: `agent-021-avoid-fetch-in-effect`_

Server components fetch on the server with no waterfall, no loading flash, and no client JS cost. This is the single highest-signal Next.js anti-pattern.

```tsx
// ❌ Client waterfall, loading flash, no SEO
'use client'
export function Profile() {
  const [user, setUser] = useState(null)
  useEffect(() => {
    fetch('/api/users/profile')
      .then(r => r.json())
      .then(setUser)
  }, [])
  if (!user) return <Spinner />
  return <div>{user.name}</div>
}

// ✅ Server component — runs on server, ships zero JS for the fetch
export async function Profile() {
  const user = await getUser()
  return <div>{user.name}</div>
}
```

If the data is genuinely client-only (user interaction-driven), use a typed data layer with Suspense + `use()` or a query library — but the default answer is "move it to the server."

### 2.2 Prefer Server Actions for mutations

_Eval: `agent-022-prefer-server-actions`_

Don't write an API route plus a `fetch('/api/...', { method: 'POST' })` for mutations originating in your own UI. Use a Server Action.

```tsx
// app/actions.ts
'use server'
export async function createPost(formData: FormData) {
  await db.posts.create({ title: formData.get('title') })
  revalidatePath('/posts')
}

// app/new/page.tsx
import { createPost } from '../actions'
export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button>Create</button>
    </form>
  )
}
```

### 2.3 Parallel data fetching — never serial `await`

_Eval: `agent-026-no-serial-await`_

If two requests don't depend on each other, fire them in parallel. Serial `await` is the most common silent performance bug in App Router code.

```tsx
// ❌ Sequential — total time = A + B + C
const analytics = await fetch('/api/analytics').then(r => r.json())
const notifications = await fetch('/api/notifications').then(r => r.json())
const settings = await fetch('/api/settings').then(r => r.json())

// ✅ Parallel — total time = max(A, B, C)
const [analytics, notifications, settings] = await Promise.all([
  fetch('/api/analytics').then(r => r.json()),
  fetch('/api/notifications').then(r => r.json()),
  fetch('/api/settings').then(r => r.json()),
])
```

For independent components, prefer **start fetching, render later**: kick off the promise without awaiting, then pass it to a child that `use()`s it under Suspense.

### 2.4 `'use cache'` directive

_Evals: `agent-029-use-cache-directive`, `agent-032-use-cache-directive` (with cache components)_

Mark cacheable work explicitly. The directive can sit at the top of a file, a function, or a component, and replaces the older `unstable_cache` API.

```tsx
async function getPosts() {
  'use cache'
  return db.posts.findMany()
}
```

For component-level caching with Cache Components enabled, the same directive works inside a server component body. Pair with `cacheTag()` / `cacheLife()` for invalidation control.

### 2.5 `updateTag()` for read-your-own-writes

_Eval: `agent-037-updatetag-cache`_

After a mutation, the user expects to see their own write on the _next render_. Use `updateTag()` (Next 16) — a synchronous, request-scoped invalidation that beats `revalidateTag()` for this case because it doesn't require waiting for a background revalidation.

```tsx
'use server'
import { updateTag } from 'next/cache'

export async function likePost(id: string) {
  await db.likes.create({ postId: id })
  updateTag(`post-${id}`) // current request sees fresh data
}
```

### 2.6 `revalidatePath` / `revalidateTag` for explicit refresh

_Eval: `agent-038-refresh-settings`_

When a mutation changes data shown on a specific route, call `revalidatePath('/settings')` from the Server Action. Pair with `router.refresh()` on the client only if you need the current view to re-render without a navigation.

---

## 3. Framework primitives over raw HTML

### 3.1 `next/link`, not `<a>`

_Eval: `agent-025-prefer-next-link`_

```tsx
import Link from 'next/link'
<Link href="/about">About</Link>   // ✅ client-side nav + prefetch
<a href="/about">About</a>         // ❌ full page reload
```

Exception: external URLs, or links that genuinely need full-page navigation (e.g. logout that must clear state).

### 3.2 `next/image`, not `<img>`

_Eval: `agent-027-prefer-next-image`_

`next/image` gives you automatic format conversion (AVIF/WebP), responsive `srcset`, lazy loading, and — critically — reserves layout space to prevent CLS.

```tsx
import Image from 'next/image'
;<Image src="/hero.jpg" alt="" width={1200} height={600} priority />
```

Mark above-the-fold images with `priority` (skip lazy loading, set as LCP candidate). Always specify `width`/`height` or `fill`.

### 3.3 `next/font`, not `<link>` to Google Fonts

_Eval: `agent-028-prefer-next-font`_

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'
const inter = Inter({ subsets: ['latin'] })

export default function RootLayout({ children }) {
  return (
    <html className={inter.className}>
      <body>{children}</body>
    </html>
  )
}
```

This self-hosts the font at build time, eliminates the third-party request, and prevents the font-swap layout shift.

---

## 4. React fundamentals

### 4.1 Derive, don't sync

_Eval: `agent-024-avoid-redundant-usestate`_

If a value is computable from other state or props, **compute it during render**. Don't mirror props into state and reconcile via `useEffect`.

```tsx
// ❌ Redundant state, drift bug waiting to happen
const [fullName, setFullName] = useState(`${first} ${last}`)
useEffect(() => {
  setFullName(`${first} ${last}`)
}, [first, last])

// ✅ Derived during render
const fullName = `${first} ${last}`
```

This rule is already in your project `CLAUDE.md` — the eval just makes it measurable.

---

## 5. Auth boundaries

### 5.1 `forbidden()` and `forbidden.tsx`

_Eval: `agent-033-forbidden-auth`_

Don't return a custom JSX 403 from a server component — that ships HTTP 200. Use the built-in auth boundary so the response status is correct.

```tsx
// app/admin/page.tsx
import { forbidden } from 'next/navigation'

export default async function AdminPage() {
  const session = await getSession()
  if (session?.role !== 'admin') forbidden()
  return <Admin />
}

// app/forbidden.tsx — renders the 403 body
export default function Forbidden() {
  return <div>You don't have access to this page.</div>
}
```

The sibling `unauthorized()` + `unauthorized.tsx` exists for 401.

---

## 6. Async runtime APIs (Next 15+)

In Next.js 15 the request-scoped accessors became async. This is one of the most common causes of latent breakage when upgrading.

### 6.1 `await cookies()` / `await headers()` / `await params` / `await searchParams`

_Eval: `agent-034-async-cookies`_

```tsx
// ❌ Next 14
const c = cookies()
const token = c.get('token')

// ✅ Next 15+
const c = await cookies()
const token = c.get('token')

// Same shift for route params:
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  // ...
}
```

### 6.2 `connection()` to opt a server component into dynamic rendering

_Eval: `agent-035-connection-dynamic`_

Replaces the older `noStore()`. Call it when a render must run per-request (e.g. uses `Math.random()` or shows time-sensitive data) but you're not reading cookies/headers that would already trigger dynamic.

```tsx
import { connection } from 'next/server'

export default async function Page() {
  await connection()
  return <div>Random: {Math.random()}</div>
}
```

### 6.3 `after()` for post-response work

_Eval: `agent-036-after-response`_

Run analytics, logging, or other side effects _after_ the response has been streamed to the user. Doesn't block TTFB.

```tsx
import { after } from 'next/server'

export default async function Page() {
  after(() => {
    logPageView({ url: '/dashboard' })
  })
  return <Dashboard />
}
```

Works in server components, Route Handlers, and Server Actions.

---

## 7. Middleware (aka "proxy" in Next.js 16)

### 7.1 Single `middleware.ts` at the project root

_Evals: `agent-031-proxy-middleware`, `agent-039-indirect-proxy`_

```ts
// middleware.ts
import { NextResponse, type NextRequest } from 'next/server'

export function middleware(req: NextRequest) {
  console.log(req.nextUrl.pathname)
  const res = NextResponse.next()
  res.headers.set('X-Request-Id', crypto.randomUUID())
  return res
}

export const config = { matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'] }
```

In **Next.js 16** the file/concept is renamed to **proxy** (`proxy.ts` + `export function proxy`). The runtime semantics are the same: it runs on every matched request before the route handler / page, in the Edge runtime by default. Use `matcher` to scope it — middleware that runs on every asset request is a performance bug.

---

## 8. Rendering & navigation

### 8.1 Partial Pre-Rendering (PPR)

_Evals: `agent-041-optimize-ppr-shell`, `agent-042-enable-ppr`_

PPR pre-renders a **static shell** at build time and streams dynamic holes in at request time. To enable:

```ts
// next.config.ts
export default {
  experimental: { ppr: 'incremental' }, // or true once stable
}

// app/dashboard/page.tsx
export const experimental_ppr = true

export default function Page() {
  return (
    <>
      <StaticHeader />
      <Suspense fallback={<Skeleton />}>
        <DynamicWidget /> {/* uses cookies/headers — becomes the dynamic hole */}
      </Suspense>
    </>
  )
}
```

Optimizing the PPR shell (eval 041) means **maximizing what can be statically pre-rendered** — push dynamic reads (`cookies()`, `headers()`, `connection()`) as deep into the tree as possible and wrap only those subtrees in `<Suspense>`.

### 8.2 View Transitions

_Eval: `agent-043-view-transitions`_

Use the View Transitions API for cross-page shared-element morphs.

```tsx
import { unstable_ViewTransition as ViewTransition } from 'react'

// Both the grid card and detail page wrap the shared image:
;<ViewTransition name={`product-${id}`}>
  <Image src={product.image} alt="" width={300} height={300} />
</ViewTransition>
```

Always honor `prefers-reduced-motion`:

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

### 8.3 `unstable_instant` / instant navigation

_Eval: `agent-040-unstable-instant` — **Inferred**_

Newer Next.js exposes APIs for "instant" navigation between cached routes (zero-latency client-side transitions for prefetched destinations). Treat this section as a flag to confirm against current Next.js release notes before relying on it — the API surface is still moving.

---

## Quick checklist

When writing or reviewing a Next.js feature, walk this list:

- [ ] Is this an **App Router** route? (no `pages/`, no `getServerSideProps`)
- [ ] Could this `'use client'` component become a **server component**?
- [ ] Any `useEffect(() => fetch(...))` → move to server component
- [ ] Any `<a href>` to an internal route → `<Link>`
- [ ] Any `<img>` → `<Image>` with dimensions and `priority` on the LCP one
- [ ] Any sequential `await` of independent calls → `Promise.all`
- [ ] Any mutation via API route + client `fetch` → **Server Action**
- [ ] After a Server Action mutates data → `revalidatePath` / `revalidateTag` / `updateTag`
- [ ] Any `useState` mirroring a prop → derive during render instead
- [ ] Any 403/401 path → `forbidden()` / `unauthorized()` (not custom JSX with 200)
- [ ] Any `cookies()` / `headers()` / `params` access → `await` it (Next 15+)
- [ ] Any logging/analytics on the request path → `after()` instead
- [ ] Static page with one dynamic widget → wrap the widget in `<Suspense>` and enable PPR
