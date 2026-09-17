# Data-Fetching Layer: TanStack Query vs SWR

**Status:** Draft for architecture review
**Scope:** Cache invalidation and data freshness — the core integration concern for our React data-fetching layer.
**Sources:** Each library's current documentation (TanStack Query v5 guides; SWR 2.x docs), linked inline. No blog posts.

---

## TanStack Query — cache invalidation via `invalidateQueries`

TanStack Query does not use a normalized cache. Instead of patching cached entities after a write, it prescribes **targeted invalidation, background-refetching, and ultimately atomic updates**.

The `QueryClient` exposes an `invalidateQueries` method that does two things:

1. **Marks matched queries as stale.** This stale state overrides any `staleTime` configuration set on `useQuery` or related hooks.
2. **Refetches in the background** any matched query that is currently being rendered via `useQuery` or related hooks.

Matching is driven by query keys:

- `invalidateQueries({ queryKey: ['todos'] })` invalidates every query whose key *starts with* `todos` (prefix matching), e.g. both `['todos']` and `['todos', { page: 1 }]`.
- Pass `exact: true` to match only the key with no additional variables/subkeys.
- For finer control, pass a `predicate` function that inspects each `Query` instance in the cache.

The canonical integration pattern is to invalidate from a mutation's `onSuccess`, so write operations drive cache freshness:

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query'

const queryClient = useQueryClient()

const mutation = useMutation({
  mutationFn: addTodo,
  onSuccess: async () => {
    // Marks all queries whose key starts with `todos` as stale
    // (overriding staleTime) and background-refetches the mounted ones.
    await queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

Returning a promise from `onSuccess` (i.e. awaiting the invalidation) keeps `isPending` true until the cache has actually been updated.

**Docs:** [Query Invalidation](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation) · [Invalidations from Mutations](https://tanstack.com/query/latest/docs/framework/react/guides/invalidations-from-mutations)

---

## SWR — automatic revalidation plus `mutate`

SWR's freshness model is **event-driven revalidation**: it refetches in response to triggers rather than waiting on a staleness timer alone. Out of the box it revalidates when:

- The window **regains focus** (`revalidateOnFocus`, enabled by default) — syncs stale mobile tabs or woken laptops.
- The **network reconnects** (`revalidateOnReconnect`, enabled by default).
- The component **mounts with stale cached data** (`revalidateIfStale`, enabled by default).
- Optionally, on a fixed **interval** — and only while the component is actually on screen:

```js
useSWR('/api/todos', fetcher, { refreshInterval: 1000 })
```

For immutable resources that should never refetch, `useSWRImmutable` (or setting `revalidateIfStale`, `revalidateOnFocus`, and `revalidateOnReconnect` to `false`) opts out entirely.

For **manual** control, SWR provides the `mutate` API (and `useSWRMutation` for on-demand remote mutations):

- `mutate(key, data)` updates the client cache directly — the async update revalidates by default (disable with `revalidate: false`), and supports `optimisticData`, `populateCache`, and `rollbackOnError`.
- `mutate(key)` with **no data** triggers a revalidation: it marks the data as expired and triggers a refetch, broadcasting to every SWR hook sharing that key.

```js
import useSWR, { useSWRConfig } from 'swr'

const { mutate } = useSWRConfig()

// Force a refresh: mark '/api/user' as expired and refetch it
mutate('/api/user')

// Update the cached data directly, then revalidate
mutate('/api/user', { name: 'jane' })
```

Gotchas worth knowing before we commit:

- The **global** `mutate` called with only a key will *not* update the cache or trigger revalidation unless a mounted SWR hook is using that key.
- The **bound** `mutate` returned from `useSWR` is functionally equivalent but doesn't require the key.
- `useSWRMutation`'s requests only fire via `trigger()` and don't share state with other hooks; its `populateCache` defaults to `false`.

**Docs:** [Automatic Revalidation](https://swr.vercel.app/docs/revalidation) · [Mutation](https://swr.vercel.app/docs/mutation)

---

## TL;DR

- **TanStack Query:** staleness is an explicit, key-driven state you control — after a write you *declare* which queries are out of date, and the mounted ones refetch in the background.
- **SWR:** freshness is mostly automatic and event-driven — focus/reconnect/mount revalidate for you, and `mutate` is the escape hatch for forced refreshes or direct cache updates.
