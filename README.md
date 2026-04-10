# ci-actions

Shared GitHub Actions for the JLLS Holdings infrastructure.

These actions live in one repo so a fix lands in all consumers via a tag bump.

## Available actions

| Action | Path | Used by |
|---|---|---|
| **iOS Build (hermetic)** | [`ios-build/`](ios-build/) | Tolera, AoT, ODIN Native, Roger app |

## How consumers reference actions

Pin to a tag in production workflows:

```yaml
- uses: HughKantsime/ci-actions/ios-build@v1
  with:
    scheme: MyApp
    bundle-id: com.example.myapp
    # ...
```

Or pin to a SHA for maximum reproducibility:

```yaml
- uses: HughKantsime/ci-actions/ios-build@<sha>
```

`@main` is allowed during development but **never in production** — it
means a push to this repo can break every consumer at once.

## Versioning

Semver via git tags (`v1.0.0`, `v1.1.0`). Movable major tags (`v1`) point
to the latest minor of that major.

```bash
# After making a change:
git tag v1.1.0
git tag -f v1
git push origin v1.1.0
git push origin v1 --force
```

Breaking changes get a new major version. Consumers update at their own pace.

## Adding a new action

```
ci-actions/
├── ios-build/
│   ├── action.yml
│   └── README.md
├── new-action/      ← create here
│   ├── action.yml
│   └── README.md
└── README.md
```

Each action gets its own subdirectory with `action.yml` (the composite
definition) and `README.md` (inputs, requirements, failure modes prevented).

## Testing changes

1. Make the change on a branch
2. Update one consumer to point at the branch (`uses: HughKantsime/ci-actions/ios-build@my-branch`)
3. Push and verify the consumer's CI passes
4. Merge to main
5. Tag a new version
6. Update consumers to the new tag

## Self-hosted runner requirements

The current actions assume the runner has:
- Xcode + command line tools
- Homebrew at `/Users/ollama/homebrew/bin` or `/opt/homebrew/bin`
- Python 3 with `PyJWT[crypto]` for ASC API calls
- Standard simulator templates (e.g., `iPhone 17 Pro Max` in `iOS 26.4`)
