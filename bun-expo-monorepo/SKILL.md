---
name: bun-expo-monorepo
description: Structure two or more independently released Expo apps in a Bun workspace. Use when migrating an Expo app into apps/, adding a standalone host, extracting reusable Expo code into packages/, or changing workspace dependency, bundling, or release boundaries.
---

# Bun Expo Monorepo

Use one repository and install for the codebase. Give each app its own runtime configuration, entry points, integration code, and release identity. Extract only the code that needs to run in more than one app.

## Set workspace ownership

```text
package.json                 Private root with apps/* and packages/* workspaces
bun.lock                     One lockfile for all workspaces
bunfig.toml                  Workspace linker configuration
apps/<app>/                  Expo Router, host adapters, native and web configuration
packages/<capability>/       Private source package, public exports, reusable assets
scripts/                     Workspace checks and orchestration
patches/                     Install-time patches shared by the workspace
```

- Pin the repo's Bun version in `packageManager`. Install from the root and use `bun install --frozen-lockfile` in CI. For the hoisted layout, set `[install]` and `linker = "hoisted"` in `bunfig.toml`. Keep app code dependencies in app manifests and shared tooling at the root.
- Put Expo config, Metro, Babel, TypeScript, environment files, native plugins, and EAS config in each app. Use Expo's workspace resolution with the app directory as the Metro project root. Start from the existing Expo Metro configuration before adding custom resolvers or watchers.
- Keep `@/` scoped to the app that defines it. Use named `workspace:*` dependencies across workspace boundaries. From the root, use named scripts that dispatch with `bun run --cwd apps/<app>` and give independently runnable apps distinct dev ports.

## Choose the smallest reuse boundary

- **One feature in one app** lives in `features/<domain>/`. In larger apps, group its screens, components, hooks, API calls, state operations, and logic within that feature. Expo Router entries in `app/` handle navigation and compose feature screens. Use `core/` for application-wide coordination and `shared/` for generic code within an app. A small host can stay flatter.
- **Several apps consuming one capability** use a private workspace package. Expose a small public API and purposeful subpaths. A package can contain TypeScript, React Native UI, and package-relative assets. It must be buildable without importing either host's routes, aliases, singletons, or UI.
- **App integration** belongs in a thin host adapter. Inject navigation actions, account context, locale, loaded fonts, layout insets, storage, and network services as needed. Keep reusable rules and UI in the package. Keep credentials, native plugins, and release settings with their app.
- **Dependencies** must be declared where used even when hoisting makes a module available by accident. Align supported Expo, React, React Native, and shared native library versions across apps. Declare host-provided modules as package peers and directly in each consuming app for Metro and native autolinking.

## Verify the workspace contract

- Run root typecheck, tests, lint, and boundary checks. Include every workspace in the aggregate tasks and preserve app-specific test environments. Check for forbidden app imports, undeclared dependencies, and incompatible versions, including host peer declarations.
- For changes to workspace layout, dependencies, or exports, export every affected app for web, iOS, and Android with source maps. Confirm each intended bundle includes the shared sources and assets and excludes sibling-app code, app-only SDKs, and server-only modules. Use native builds or device tests when the change calls for runtime evidence and the project's testing instructions allow it.
- Treat each app's binary and OTA update as a separate release. A package change reaches an installed app through a release built for that app's native runtime.

Read [Package and host design](references/package-and-host-design.md) when extracting a capability, sharing data access across hosts, or deciding where app code belongs. Read [Migration and verification](references/migration-and-verification.md) when introducing a workspace, splitting an app, upgrading shared native dependencies, or changing release routing.
