# Migration and verification

## Establish a baseline and move the app

1. Capture the current typecheck, test and lint results. Record route URLs, deep links, native identities, EAS project and runtime policy, storage keys/fixtures, and any dependency exceptions. Read the current manifests and config before using a historical migration record as a baseline.
2. Relocate the existing app under `apps/<name>/` with its configuration, assets, app-local environment setup, and native plugins. Keep sensitive files ignored. Preserve existing routes, update identity, and storage contracts. Keep root tooling and install hooks where they serve the workspace. Existing root commands can forward to the original app while new named commands select a host explicitly.
3. Extract the reusable package with an adapter for the original app before adding the second host. Keep package peers aligned with the consuming apps and install once at the root. Compare lockfile changes with the baseline so relocation and unrelated upgrades remain independently verifiable.
4. Add the second host's own app config, Metro/Babel/TypeScript configuration, env template, build profiles, native identifiers, deep-link scheme, callback configuration, and storage namespace. If a development client must coexist with production, give it its own install identifiers and scheme and use the same variant when serving its bundle.

## Check the points where workspace assumptions break

| Check | What to verify |
| --- | --- |
| Root install and task wiring | Frozen-lockfile install resolves `workspace:*` links and applies required patches. Root typecheck and test tasks cover every app and package. Extend any explicit project or workspace lists when adding one. |
| Static boundaries | Inspect static imports, re-exports, and literal dynamic imports/`require` calls. Catch app aliases and relative imports escaping shared packages, undeclared dependencies, and peer mismatches. Enforce the repo's naming rules with framework exceptions. |
| Tests by environment | Use appropriate native and web/DOM test environments. Keep regression fixtures for routes, app identity, persisted state, and shared behavior across both hosts. |
| Bundles | From each app, run `EXPO_OFFLINE=1 NODE_ENV=production bunx expo export --platform all --source-maps` with a known output directory. Inspect fresh source maps for expected shared modules/assets and forbidden sibling-app, server-only, or app-specific SDK imports. |
| Native and external integration | Inspect native autolinking for both platforms when native peers change. Verify each rebuilt development or release client as appropriate. Validate sign-in and recovery using the actual app scheme or web origin. Export checks cover packaging. Native builds and device checks cover installed behavior. |

## Upgrade and release independently

- For a coordinated Expo upgrade, start with the target SDK's compatibility recommendations. Update consuming apps, shared-package peers, and root test tooling together. Review any intentional exceptions, rebase patches, install at the root, and recheck each app. Review API, config plugin, and native toolchain changes separately from manifest compatibility.
- Start Expo or EAS operations in the selected app directory. Keep bundle IDs, schemes, EAS project IDs, update URLs, profiles, channels, credentials, and runtime version policy scoped to the app. Confirm the resolved config for the selected variant before building or updating it.
- A native module, config plugin, or install identity change needs a rebuilt binary and a runtime version compatible with that binary. Under an `appVersion` policy, bump the app version when changing native compatibility. A JavaScript or asset-only change can use OTA when it fits the existing runtime and release policy.
- If automating release selection, compare against each app's own previous release. Consider that app's files, its transitive workspace packages and resolved dependencies, and shared build inputs. Changes to a sibling app alone should be excluded. Treat incomplete classification as requiring review before publishing.
