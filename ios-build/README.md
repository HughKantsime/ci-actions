# ios-build composite action

Hermetic iOS build/test/archive/upload pipeline. Designed for the multi-app
self-hosted Mac runner where multiple iOS pipelines need to run in parallel
without contention.

## Why this exists

A naive iOS CI workflow on a shared Mac will break when two pipelines run
concurrently. The failure modes are subtle and hard to debug:

| Failure | Cause |
|---|---|
| `errSecInternalComponent` mid-archive | Both jobs created `build.keychain` and `default-keychain -s` ran twice. The second job stomped the first job's keychain reference. |
| `Unable to boot device in current state` | Both jobs called `simctl boot "iPhone 17 Pro Max"`. Second one fails because the device is already booted by the first. |
| Tests run on the wrong simulator state | Job A booted iPhone 17 Pro Max, then Job B's tests grabbed the same booted device and saw Job A's app installed. |
| Random codesign timeouts | `CoreSimulatorService` daemon got overloaded by parallel boot/install/launch from two pipelines. |
| `xcodebuild` deadlocks | Both jobs tried to write to `~/Library/Developer/Xcode/DerivedData` at the same time. |

## What this action does

Every job that uses this action gets:

1. **Per-job keychain** at `$RUNNER_TEMP/build-<run_id>.keychain` with a
   randomly generated password. Never modifies `default-keychain`. Adds the
   per-job keychain to the user search list only for the duration of the job,
   then removes it during cleanup.
2. **Cloned simulator** — a fresh clone of the requested template device.
   The template is never booted by this job, only the clone. The clone is
   deleted in the cleanup step.
3. **Isolated DerivedData** at `$RUNNER_TEMP/DerivedData-<run_id>`.
4. **Per-job archive and export paths** under `$RUNNER_TEMP`.
5. **Per-job ASC API key directory** so altool's global config dir isn't
   shared between jobs.
6. **Apple WWDR intermediate cert** imported into the per-job keychain so
   codesign can build the trust chain.
7. **Pre-build identity verification** — `find-identity -v` runs right
   after import. If the cert chain is broken, the job fails in 2 seconds
   instead of 20 minutes later in Archive.
8. **Cleanup runs `if: always()`** — deletes the keychain, simulator clone,
   and all temp files even if the job failed.

## Requirements

- Self-hosted Mac runner with:
  - Xcode + command line tools
  - Homebrew
  - Python 3 with `PyJWT[crypto]` for ASC API calls
  - The template simulator devices already created (e.g., `iPhone 17 Pro Max`
    in `iOS 26.4` runtime)
- Workflow `env.PATH` set to include the user's homebrew bin

## Usage

```yaml
- uses: actions/checkout@v4

- name: Build, Test, Archive, Upload
  uses: ./.github/actions/ios-build
  with:
    scheme: MyApp
    test-scheme: MyAppUITests
    bundle-id: com.example.myapp
    template-device: 'iPhone 17 Pro Max'
    runtime: 'iOS 26.4'
    marketing-version: '1.0'
    dist-cert-p12: ${{ secrets.DIST_CERT_P12 }}
    dist-cert-password: ${{ secrets.DIST_CERT_PASSWORD }}
    provisioning-profile-name: 'MyApp AppStore'
    asc-key-id: ${{ secrets.ASC_KEY_ID }}
    asc-issuer-id: ${{ secrets.ASC_ISSUER_ID }}
    asc-private-key: ${{ secrets.ASC_PRIVATE_KEY }}
    archive: ${{ github.event_name == 'push' }}
    upload-testflight: ${{ github.event_name == 'push' }}
    uitest-class: 'MyAppUITests/QATests'
```

For runners where macOS Sequoia rejects `codesign --keychain=<per-job>` with
`errSecInternalComponent`, set `use-keychain-flag: 'false'`. The action still
uses the isolated keychain and login-keychain fallback, but lets `codesign`
search the keychain list for the selected identity.

## Inputs

See `action.yml` for the complete list. Required:
- `scheme`, `bundle-id`
- `dist-cert-p12`, `dist-cert-password`
- `provisioning-profile-name`
- `asc-key-id`, `asc-issuer-id`, `asc-private-key`

## What this action does NOT do

- Profile regeneration on App Store Connect (each app's required
  capabilities differ — keep this in your project workflow)
- TestFlight beta group assignment (project-specific group names)
- Project generation via XcodeGen (the action calls `xcodegen generate`
  but assumes a `project.yml` exists)
- Worker/backend builds (this is iOS-only)

## Hermetic verification

Run two pipelines simultaneously on the same Mac. Both should succeed with
no `errSecInternalComponent`, no simulator collisions, no shared state.

## When to update this action

This action should live in **one place per ecosystem** and be copied or
referenced (via `uses: org/shared-actions/ios-build@vN`) into each iOS
project. When you update it:

1. Test on one repo first
2. Run two pipelines in parallel to verify no regressions
3. Roll out to other iOS repos

If you find you need to override behavior per-project, prefer adding an
input to this action over forking it. The whole point is one source of truth.
