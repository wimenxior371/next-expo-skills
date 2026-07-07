---
name: next-expo-project-structure
description: Structure and organize a large Next.js or Expo project — where components, features, hooks, lib, types, constants, and routes live. Use when scaffolding a new app, reorganizing a growing codebase, or deciding where a new file should go.
---

# next-expo-project-structure

A pragmatic, feature-oriented layout for large file-based-routing apps. It works for **Next.js** (App Router) and **Expo** (Expo Router) because both use a file-based `app/` directory; the difference is only what lives *around* that directory.

## When to use

- Scaffolding a new Next.js or Expo app that you expect to grow.
- Reorganizing a codebase where business logic has leaked into UI components or global "junk drawer" folders (`utils.ts`, `helpers.ts`, `data.ts`).
- Deciding where a new file should live: "Is this global or feature-specific?"

Do **not** over-apply this to small apps. Start simple and add structure as the app grows.

## Core philosophy

> Keep route-level code in `app`, shared code in `components`, business logic in `lib`, and feature-specific logic inside feature folders.

The one question that decides every placement:

- **Reused everywhere → global folder** (`components/`, `hooks/`, `lib/`, `types/`).
- **Belongs to one feature → that feature's folder** (`features/<name>/…`).

## The `app/` directory is flexible

The layout below is agnostic about *where* the app directory lives and *which* framework owns it. Detect the existing convention and match it — do not force a move.

| Setup | Route directory | Everything else |
| --- | --- | --- |
| Next.js with `src/` | `src/app/` | `src/components`, `src/features`, … |
| Next.js without `src/` | `app/` (repo root) | `components`, `features`, … at root |
| Expo Router with `src/` | `src/app/` | `src/components`, `src/features`, … |
| Expo Router without `src/` | `app/` (repo root) | `components`, `features`, … at root |

Rules for staying flexible:

1. **Follow the existing convention.** If the project already uses `src/`, keep everything under `src/`. If routes already sit at the repo root `app/`, put the sibling folders (`components/`, `features/`, …) at the root too. Never mix both.
2. **The `app/` folder only holds routes** — regardless of framework or location.
3. Configure a path alias (`@/*`) so imports don't care where the root is. Next.js: `tsconfig.json` `paths`. Expo: `tsconfig.json` `paths` + `babel.config.js` `expo-router/babel` is automatic, but add the alias to `tsconfig`.

## Reference structure

`ROOT` below is either the repo root or `src/`, whichever the project uses.

```
ROOT/
  app/                      # ROUTES ONLY — file-based routing
    ...                     # Next.js: page.tsx/route.ts/layout.tsx
                            # Expo:    index.tsx/_layout.tsx + (groups)
  components/               # Global, reusable UI
    ui/                     # Small design primitives (Button, Input, Modal)
    layout/                 # Navbar, Sidebar, Footer, TabBar
    shared/                 # Cross-feature app-level (EmptyState, LoadingSpinner)
  features/                 # ⭐ Business modules — the most important folder
    auth/
      components/
      hooks/
      actions/             # (Next.js server actions) or services/
      types.ts
    dashboard/
      components/
      hooks/
      services/
      types.ts
    billing/
      components/
      hooks/
      services/
      types.ts
  hooks/                    # GLOBAL hooks only (useDebounce, useMediaQuery)
  lib/                      # Shared logic + external service clients
    prisma.ts / db.ts
    auth.ts
    stripe.ts
    openai.ts
    utils.ts
  types/                    # GLOBAL TypeScript types (user.ts, api.ts, common.ts)
  constants/               # Unchanging values (routes.ts, roles.ts, navigation.ts)
  styles/                  # Global CSS (Next.js globals.css / Expo global.css for NativeWind)
```

Framework-specific additions:

- **Next.js:** `middleware.ts` at `ROOT`. API routes live in `app/api/<name>/route.ts`.
- **Expo:** `assets/` (fonts, images, icons) usually at the repo root. API routes, if used, live in `app/<name>+api.ts` (Expo Router API routes).

## Folder responsibilities

### `app/` — routes only
Pages/routes connect a URL to a feature; they contain **minimal business logic**. A page imports from `features/…` and composes; it does not implement.

### `components/` — global UI, three buckets
- `ui/` — small, reusable, presentational primitives.
- `layout/` — structural chrome (navbars, sidebars, footers, tab bars).
- `shared/` — app-level components used across multiple features.

If a component belongs to exactly one feature, it does **not** go here — it goes in `features/<name>/components/`.

### `features/` — the critical folder
Group code by **business module**, not by file type. Each feature is self-contained: `components/`, `hooks/`, `services/` (or `actions/`), and `types.ts`. Billing logic lives in `features/billing/` — never scattered across global folders.

This is the antidote to type-based sprawl (one giant `hooks/`, one giant `types/`, one giant `services/`).

### `lib/` — shared logic & external clients
Database clients, SDK setup, auth helpers, generic utilities. **No feature-specific logic here** — no billing rules, no dashboard queries.

### `hooks/`, `types/`, `constants/`, `styles/`
Global-only. Feature-specific hooks/types belong in the feature (e.g. `features/billing/hooks/useSubscription.ts`). Keep global type files small; push domain types into features to avoid monolithic files.

## Naming conventions

Names should say what the thing does:

- Components: `Button.tsx`, `UserCard.tsx`, `DashboardStats.tsx`
- Hooks: `useDashboardData.ts`, `useDebounce.ts`
- Services: `billingService.ts`

Avoid vague names: `helper.ts`, `data.ts`, `misc.ts`, `utils.ts`-as-a-dumping-ground.

## Server vs. client components (Next.js) / native rendering (Expo)

**Next.js App Router:**
- **Server Components (default):** data fetching, static pages, protected server logic, SEO-critical pages.
- **Client Components (`"use client"`):** forms, modals, dropdowns, interactive dashboards, local state, browser APIs.

**Expo Router:** most screens are client components. Move data fetching into `features/<name>/services/` + hooks, and keep screens in `app/` thin — the same "routes stay thin, logic lives in features" rule applies.

## Anti-patterns to avoid

1. Business logic stuffed inside UI components.
2. Monolithic utility files mixing unrelated functions.
3. Feature-specific logic scattered across global directories.
4. Over-engineering structure for a small project. **Start simple; add structure as the app grows.**

## Related skills

Once the structure is in place, defer framework specifics to the dedicated skills:

**If the project uses Expo / Expo Router:**
- `expo-router` — file-based routes, groups, dynamic routes, layouts, tabs, modals inside `app/`.
- `building-native-ui` — building the components in `components/ui`.
- `native-data-fetching` — implementing `features/*/services` and data hooks.
- `expo-api-routes` — Expo Router API routes.
- `expo-ui` — native SwiftUI/Compose components.

**If the project uses Next.js:**
- `next-dev-loop` — verify routes/components actually work at runtime after restructuring.
- `next-cache-components-adoption` / `next-cache-components-optimizer` — caching and rendering strategy for the `app/` routes.

## Instructions

1. **Detect the framework and app-directory convention.** Check for `app.json`/`expo` deps (Expo) vs `next` dep (Next.js), and whether routes live under `src/app` or root `app`. Match what exists.
2. **Set `ROOT`** to `src/` or the repo root accordingly, and confirm the `@/*` path alias resolves to it.
3. **Create the sibling folders** (`components/{ui,layout,shared}`, `features/`, `hooks/`, `lib/`, `types/`, `constants/`, `styles/`) next to `app/`.
4. **Place each file by the global-vs-feature test.** Reused everywhere → global folder; belongs to one feature → `features/<name>/`.
5. **Keep `app/` routes thin** — they compose features, not implement them.
6. **Hand off** framework specifics to the related skills above.
