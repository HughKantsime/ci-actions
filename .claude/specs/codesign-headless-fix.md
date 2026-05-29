---
slug: codesign-headless-fix
title: Make iOS CI codesign reliably headless (kill the keychain password popup)
status: specced
gemini_passes: 1
gemini_status: passed
created: 2026-05-29
repos: [ci-actions, roger-app]
---

# Codesign headless fix

The shared composite `ci-actions/ios-build@v1.1.3` intermittently triggers an
interactive macOS keychain password dialog on the self-hosted M4 runner during
the Archive/codesign step. The dialog asks for a keychain password that no human
value satisfies, blocking the build (manifests as `errSecInternalComponent` and
the GUI popup the operator sees). Verified live this session across roger-app
build runs.

## Root cause (verified in `ci-actions/ios-build/action.yml`)

The "Create isolated keychain" step (lines ~155-242):
1. Imports the dist cert into a per-job keychain and runs
   `set-key-partition-list -s -k "$KEYCHAIN_PASS" "$KEYCHAIN_PATH"` (correct
   password). This is the intended headless path. It is flaky on Sequoia when the
   runner has had no recent interactive session.
2. **The fallback (lines 231-242) is the actual popup source.** It imports the
   cert into the LOGIN keychain with `-A`, then calls
   `security set-key-partition-list -S ... -s -k "" "$LOGIN_KC"` — with an
   **empty password** (`-k ""`). The login keychain's password is NOT empty, so
   this call fails (swallowed by `|| true`). The cert/key is now in the login
   keychain WITHOUT a partition list granting codesign non-interactive access →
   when codesign reaches that key, macOS shows the interactive password prompt.
   It also pollutes the operator's login keychain.

## Fix (ci-actions/ios-build/action.yml — the implementation target)

Goal: the per-job keychain path is reliable enough that the login-keychain
fallback is never needed, and the fallback (popup source) is removed.

### CG-CI.1 — Harden the per-job partition-list (replace the retry block at action.yml:221-230)
- Before `set-key-partition-list`: ensure the per-job keychain is (a) unlocked and
  (b) already in the user search list. The search-list insertion is currently
  AFTER partition-list (action.yml:244-271); move it BEFORE the partition-list call
  (partition-list is more reliable when the keychain is discoverable + unlocked
  in-session). NOTE (A6 Gemini): this ordering change is a HYPOTHESIS, not a proven
  fix — keep it, but the load-bearing reliability comes from the retry + verify.
- Retry up to 3x with `sleep 1` between attempts, re-running
  `security unlock-keychain -p "$KEYCHAIN_PASS" "$KEYCHAIN_PATH"` before each retry.
  Keep the correct `-k "$KEYCHAIN_PASS"`.
- VERIFY: a `security find-identity -v -p codesigning "$KEYCHAIN_PATH"` check
  ALREADY EXISTS at action.yml:274 — STRENGTHEN that existing line (assert exactly
  the Apple Distribution identity is present), do NOT add a duplicate. If all
  retries fail, **`exit 1` with a clear `::error::`** — fail loud, never fall back
  to an interactive path.

### CG-CI.2 — Remove the login-keychain fallback (delete action.yml:233-242)
Delete the entire `LOGIN_KC=` import + empty-password `set-key-partition-list`
block (action.yml:233-242; the comment is 231-232, the empty-password call is
line 241). It is the popup source and pollutes the operator login keychain.
Gemini confirmed (A2) it is NOT load-bearing on the happy path: when the per-job
partition-list succeeds and the archive pins `--keychain` (CG-CI.3), it
contributes nothing. With CG-CI.1 hardened + fail-loud, it is unnecessary.

### CG-CI.3 — Keep codesign pinned to the per-job keychain + close the escape hatch
The input is **`use-keychain-flag`** (action.yml:85, boolean default `'true'`),
NOT `keychain-flag` (B1 Gemini). It gates appending
`OTHER_CODE_SIGN_FLAGS=--keychain=$KEYCHAIN_PATH` in the Archive step
(action.yml:453-455). roger-app's caller (build.yml:209-238) does not set it, so it
inherits `true` — good.
**B2 (Gemini) consistency fix:** the input's description (action.yml:86) says "Set
false when Sequoia ACL fallback via login keychain is needed" and README.md:78-81
instructs consumers to set `use-keychain-flag: 'false'` for that fallback. CG-CI.2
deletes that fallback, so the `false` path now leads to NO `--keychain` pin AND no
fallback → guaranteed dialog/failure. Resolve by EITHER:
 (a) removing the `use-keychain-flag: 'false'` escape hatch entirely (drop the
     input or hard-pin `--keychain` always), updating action.yml:86 description; OR
 (b) keeping the input but rewriting action.yml:86 + README.md:78-81 to state the
     login fallback no longer exists and `false` is unsupported.
Prefer (a) — always pin `--keychain=$KEYCHAIN_PATH`; simplest and removes the
foot-gun. Update README.md:78-81 (the FULL block, A5 Gemini), not just line 80.

### CG-CI.4 — Cleanup hygiene (one-time login-keychain de-pollution)
Confirm the existing cleanup step still removes the per-job keychain from the
search list and deletes it (keep as-is).
Because CG-CI.2 removes the fallback, NO NEW login-keychain pollution can occur —
this cleanup is a ONE-TIME migration to clear residue left by prior `@v1.1.3`
runs. Implement carefully:
- **B3 (Gemini): `security delete-identity` is NOT a valid subcommand.** Use
  `security delete-certificate -Z <SHA-1> -t "$LOGIN_KC"` (the `-t` also removes
  the matching private key). Verify flags against the local `security` man page.
- **B4 (Gemini): match by SHA-1 ONLY, never common name.** The CN
  (`Apple Distribution: SHANE ANDREW SMITH (6RXAHHH6UL)`) may be the operator's
  OWN legitimately-installed identity used for local Xcode — deleting by `-c`/CN
  would destroy it. Capture the SHA-1 of THIS job's cert from the pre-flight PEM
  (action.yml:187-209) BEFORE `rm -f "$CERT_PEM"` (e.g.
  `openssl x509 -in "$CERT_PEM" -noout -fingerprint -sha1`), normalize to the hex
  form `security` expects, and delete only that exact fingerprint from
  `$LOGIN_KC`. Best-effort (`|| true`), never fail the build on cleanup.
- Given the risk, if capturing/normalizing the SHA-1 reliably is uncertain,
  it is acceptable to SKIP the login-keychain cleanup entirely and instead note in
  the PR that a one-time manual `security delete-certificate -Z <sha1> -t login.keychain-db`
  may be needed on the runner. Do NOT ship a CN-based deletion.

### CG-CI.5 — Tag a new composite version
Cut `v1.1.4` (and move/update the `v1` major alias if the repo maintains one).
The fix has no effect until consumers reference the new tag.

## Fix (roger-app — consumer ref bump)

### CG-CI.6 — `roger-app/.github/workflows/build.yml`
Bump `uses: HughKantsime/ci-actions/ios-build@v1.1.3` (line ~210) to `@v1.1.4`.
The macOS job's inline keychain step (build.yml ~364-402) does NOT have the buggy
login-keychain fallback, so it is out of scope — but it shares the per-job
partition-list flakiness. OPTIONAL: mirror CG-CI.1's retry+verify+fail-loud into
that inline step for consistency. If done, keep it a separate, clearly-labeled
edit; do not introduce a login-keychain fallback there.

## Out of scope / notes
- **Build number:** the burned `229` (run 2's stalled altool consumed the version)
  is resolved separately by triggering a FRESH workflow run (new `github.run_number`
  → 230), NOT by re-running run 229. This spec does not change versioning.
- Do not change the ASC upload (altool) logic here.

## Build & verify
- ci-actions: `action.yml` is YAML — lint/parse clean (`yamllint` or
  `actionlint` if available; otherwise a YAML parse check). No runtime build.
- roger-app: the real verification is a FRESH push triggering build 230. Watch:
  (1) NO interactive keychain popup on the M4, (2) codesign/archive succeeds,
  (3) altool upload completes, (4) build 230 appears on TestFlight (ASC API).
  The IPA-snapshot guard stays on during the watch.
- Live ASC check after: `GET /v1/builds?filter[app]=6761531576&filter[version]=230`
  shows the build PROCESSING/VALID.

## Ship gate
ci-actions: push + tag v1.1.4. roger-app: push build.yml bump → fresh run 230.
Operator should expect NO password prompt this run; if one appears, the fix
failed and we stop (do not hand-feed the dialog).
