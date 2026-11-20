---
name: next-app-router-patterns
description: Choose and implement Next.js 16 App Router patterns for server rendering, caching, mutations, request routing, and URL-driven UI. Use when designing an App Router feature or adapting one to Next.js 16 conventions.
---

# Next.js App Router Patterns

Check the installed Next.js version, router, `next.config.*`, and build scripts before applying version-specific behavior. Keep the feature within the project's existing router and caching model unless migration is part of the task.

## Place work at the right boundary

- **Server Components** own initial data reads and non-interactive rendering. Call the server data layer directly. Keep credentials and database modules server-only and pass only the serializable data the browser needs to Client Components.
- **Client Components** own browser APIs, local state, and event handlers. Place `'use client'` at the smallest practical interactive boundary. Pass server-rendered content as children when an interactive wrapper needs it.
- **Server Actions** handle UI mutations such as form submissions. Validate submitted values on the server, verify the caller's permission for the specific record, and invalidate affected data after the write succeeds. Use a Route Handler when a webhook, external client, or explicit HTTP contract needs an endpoint.
- **Proxy** handles early redirects, rewrites, headers, and inexpensive session checks. Scope its matcher to affected paths and enforce authorization again where protected data is accessed or changed. Next.js 16 `proxy.ts` uses Node.js. Check runtime dependencies before migrating Edge Middleware.

## Decide what can render early

- With `cacheComponents: true`, keep static layout and navigation in the prerendered shell. Cache reusable data with `'use cache'` when its freshness and access scope permit it. Use `cacheLife` for freshness and `cacheTag` when a write or webhook needs to invalidate related results.
- Put request-time reads inside child components under `<Suspense>`. The boundary must wrap the component doing the read. Keep cookies, headers, and per-user data outside shared cache scopes. Without Cache Components, follow the app's existing caching conventions, such as `fetch` with `next.revalidate` for timed revalidation.
- Choose invalidation by how fresh the next read must be. Use `updateTag` in a Server Action when the initiating user needs to see their write immediately, `revalidateTag(tag, 'max')` when a stale response during refresh is acceptable, and `revalidatePath` for path-scoped invalidation.

## Choose routing by navigation behavior

- Use `@slot` parallel routes when panels need independent navigation or loading and error states. Add `default.tsx` fallbacks for named slots so hard navigation can resolve unmatched content.
- For a shareable modal, keep a canonical detail page and intercept the same route inside a modal slot on client navigation. For example, `app/catalog/@modal/(.)[id]/page.tsx` can intercept `app/catalog/[id]/page.tsx`. Slot directories do not count as URL segments. Give the modal slot a null fallback and match destinations that should dismiss it with a null-rendering route so soft navigation clears the slot.
- Verify modal open and dismissal, direct URL entry, refresh, and browser Back and Forward. Verify hard loads for affected parallel routes.

## Apply Next.js 16 conventions

- Turbopack is the default for development and production builds. During an upgrade, check Webpack hooks injected by plugins as well as local configuration. Use `--webpack` when the affected command still needs Webpack behavior, and verify the actual production build.
- For TypeScript link checking, use top-level `typedRoutes: true`. Run `next typegen` before standalone type checking when generated route types are needed. Await route `params` and `searchParams` where provided, including `params` in `generateMetadata`.
- Use `next/image` with dimensions or `fill` and accurate `sizes` for responsive images. Reserve `preload` for a known LCP image that needs early discovery. Use `next/font` for font loading and `metadata` or `generateMetadata` for page and sharing metadata.
- For runtime diagnosis, use the project's `AGENTS.md`, development logs, and Next.js DevTools MCP when available. For caching or rendering changes, also check the production build and compare direct loads with client navigation.
