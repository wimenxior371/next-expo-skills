# Package and host design

## Keep the app structure proportional

Use a domain owner first and a technical role second. For example, `features/booking/hooks/use-booking.ts` keeps the hook with the booking feature. Create subfolders as the feature grows. Features may import each other explicitly when composing an experience. Code that still represents the domain belongs with its owner even if several features use it.

- `app/` holds Expo Router files and route declarations. Keep a route's implementation in its owning screen or service. For `+api.ts`, import the server implementation directly and keep it out of client-facing entry points.
- `core/` composes providers, startup, navigation, the root store, connectivity, and updates. When a root store combines feature slices, import leaf defaults, types, and pure transforms. Feature actions can consume the constructed store. Importing those actions during construction risks using a partially initialized singleton.
- `shared/` contains reusable app-level primitives, hooks, themes, and adapters. Code can move to `packages/` once multiple hosts need the same capability and can supply its app-specific dependencies.
- Use kebab-case directories and ordinary modules, PascalCase React component files, and `use-*.ts` hook filenames with camelCase exports. Co-locate matching `.test` files. Preserve router, generated, and external contract names. Keep platform siblings on the same base name and extension, such as `Controls.tsx` and `Controls.web.tsx`, and import them without the extension. A generic `.ts` fallback can be picked before a platform-specific `.tsx` sibling.

## Extract a capability around its public contract

For a shared interactive feature embedded in a larger app and mounted in a standalone app:

| Package owns | Host owns |
| --- | --- |
| Engine, UI, rules, feature dictionaries, and assets | Router, app chrome, account screens, selected locale, loaded fonts, and safe areas |
| Progress types, defaults, normalization, and operations | Storage adapter, key namespace, hydration lifecycle, and recovery UI |
| Service contract and optional reusable service implementation | Client instance, availability policy, session identity, and completion callbacks |

Expose a component or provider at the root export. Add subpaths for independent consumers, such as `./progress`, `./locales`, or a backend adapter, so those imports can stay independent of UI and native initialization. Point `exports` and the main/type entry points at source for an Expo-consumed TypeScript package. When an export needs explicit platform conditions, provide the appropriate browser/native entry and a fallback. Verify both targets and prefer direct imports over broad barrels that initialize unrelated modules.

Declare `"@example/feature": "workspace:*"` in each consuming app. Keep package assets beside the code that uses them and reference them with Metro-compatible static paths. If the host styles its own app with a scanner-based system, verify which roots it scans. A package using explicit React Native styles can render in either host without inheriting that styling toolchain.

## Share data access without coupling screens

When hosts need the same backend contract, keep request functions, response types, and mapping together behind a UI-independent export. Accept a client or transport from the host. Each host can call those functions from its own data hooks and keep form, navigation, and screen state local. Within a single app, keep the same separation inside the owning feature or an app-local data module.

If hosts already share a client cache integration, expose it separately from the plain request functions. Reuse the same key builders for reads and post-write invalidation. When verifying an extraction, test mapping and service behavior with a controlled transport, then exercise each host's UI with a controlled service.

## Preserve data ownership

When extracting an existing feature, let the old host adapt its existing persisted slice to the new package contract. Give a standalone app separate session, preference, and progress keys, even if both apps connect to the same backend. App bundles and local storage are independent.

Hydrate before mounting a feature that can write progress. Preserve unreadable data for recovery and serialize writes where they can overlap. If a storage key or format changes, migrate before startup consumes it, make the migration resumable, and confirm the replacement write before removing old data. Cover an upgrade from saved data on both native and web where their adapters differ.

When sharing offline synchronization or auth callback handling, scope deduplication to the configured host/runtime. For queued uploads, remove successful entries only, keep failed and unattempted entries, and stop submissions for a previous user on account change. Client-side deduplication protects concurrent calls in one runtime. Crash-safe retries across local and remote writes require server-side idempotency.
