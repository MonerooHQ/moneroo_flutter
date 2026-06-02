# CLAUDE.md

Guidance for Claude Code and other AI agents working in this repository.

## Project overview

`moneroo_flutter_sdk` is a Flutter package (v0.3.4) that enables mobile apps to integrate Moneroo's African payment gateway. It targets Android and iOS only and is published on pub.dev. Consumers add it as a dependency to their Flutter apps and use either the pre-built `Moneroo` widget or the raw `MonerooApi` class to process payments via mobile money, cards, crypto, and bank transfers across 15+ African currencies.

## Tech stack

- Dart SDK `^3.3.0`, Flutter `>=3.0.0`
- `dio ^5.4.1` — HTTP client for Moneroo REST API calls
- `webview_flutter ^4.7.0` + platform implementations (Android `^4.3.2`, WKWebView `^3.13.0`) — renders the hosted checkout page
- `flutter_spinkit ^5.2.0` — loading indicator while checkout URL is fetched
- `json_annotation ^4.8.1` / `json_serializable ^6.7.1` + `build_runner ^2.4.8` — code-generated JSON serialization (`.g.dart` files)
- `very_good_analysis ^7.0.0` — strict lint rules (`analysis_options.5.1.0.yaml`)
- `mocktail ^1.0.3` — mocking in tests
- External service: `https://api.moneroo.io` (v1)

## Getting started

```bash
# Install package dependencies
flutter pub get

# Regenerate JSON serialization code after editing a model
dart run build_runner build --delete-conflicting-outputs
```

No `.env` or secrets setup is required in the SDK itself; callers supply an API key at runtime.

## Common commands

| Task | Command |
|---|---|
| Get dependencies | `flutter pub get` |
| Analyze code | `flutter analyze` |
| Test | `flutter test` |
| Regenerate `.g.dart` files | `dart run build_runner build --delete-conflicting-outputs` |
| Run example app | `cd example && flutter run` |

## Architecture

The package is entirely under `lib/src/` with a single public barrel file at `lib/moneroo_flutter_sdk.dart`.

- `lib/src/api/` — `MonerooApi` (Dio-based wrapper for `POST /v1/payments/initialize`, `GET /v1/payments/:id/verify`, `GET /utils/payout/methods`) and `Endpoints` constants.
- `lib/src/models/` — JSON-serializable data classes (`MonerooPayment`, `MonerooPaymentInfos`, `MonerooCustomer`, `MonerooRemoteMethod`, `MonerooApiResponse`). All have hand-written `.dart` files and generated `.g.dart` counterparts.
- `lib/src/commons/` — `enums.dart` (currencies, payment methods, statuses, API version) and `exceptions.dart` (`MonerooException`, `ServiceUnavailableException`).
- `lib/src/widgets/` — `Moneroo` (a `StatefulWidget` that calls `MonerooApi.initPayment`, renders a `WebViewController` with the checkout URL, monitors `onUrlChange` for the callback URL, then calls `onPaymentCompleted` or `onError`).
- `test/src/moneroo_api_test.dart` — integration-style tests that call the live API; requires a real API key substituted for `'YOUR-API-KEY'`.
- `example/` — standalone Flutter app demonstrating both widget and API usage.

## Conventions

- Lint rules enforced via `very_good_analysis`; the `analysis_options.yaml` excludes generated `*.g.dart` files. CI runs `flutter analyze` on every push/PR to `main`.
- All public symbols must have doc comments; the 80-char line limit is suppressed only where URLs make compliance impossible (`// ignore_for_file: lines_longer_than_80_chars`).
- JSON models use `json_serializable`; after editing any model annotated with `@JsonSerializable`, re-run `build_runner` to regenerate the `.g.dart` file — never edit `.g.dart` files manually.
- The `Moneroo` widget detects payment completion by watching for a URL that does not contain `'moneroo'`; the `callbackUrl` parameter feeds the return URL passed to the API.
- Sandbox mode is toggled with `sandbox: true` on both `MonerooApi` and the `Moneroo` widget; no separate API key is needed.
- Platform support is explicitly limited to `android` and `ios` in `pubspec.yaml`; do not add web or desktop targets to the SDK package without also implementing or replacing the `webview_flutter` dependency.

## Git Conventions

### 1. Branch names

Enforced regex (`branch_name_pattern`):
```
^(feature|fix|hotfix|chore|docs|refactor|test|ci|perf|build|style)/[a-z0-9._-]+$
```

- Lowercase only, kebab-case after the prefix, **max 50 characters** total.
- Use the full word `feature/` — **never** `feat/` (the short `feat` form is only for commit message types).
- Include the ticket id when relevant: `feature/AXA-123-add-stripe` (the ticket id is lowercased to satisfy the pattern — e.g. `feature/axa-123-add-stripe`).
- **Never** use a `claude/` prefix or any prefix outside the allowed set.
- `main`, `release`, `staging` are permanent protected branches — never push to them directly.
- If a branch is misnamed, rename it before pushing: `git branch -m <old> <new>`.

### 2. Commit messages
Enforced regex (`commit_message_pattern`), applied to **every** commit:
```
^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([^)]+\))?!?: .+
```
- Lowercase type, optional scope in parens, optional `!` for breaking changes, subject after `: `.
- Subject starts with a lowercase letter and has no trailing period.
- Examples: `feat(checkout): add Apple Pay support`, `fix(api): handle expired tokens`, `chore(deps): bump axios from 1.7.2 to 1.15.2`, `refactor!: drop Node 18 support`.
- Do not rewrite Dependabot commits — `chore(deps): bump X from a to b` is already enforced via `.github/dependabot.yml`.

### 3. Files that are always rejected
Never stage or commit:
- `.env`, `.env.*` (only `.env.example` and `.env.sample` are allowed), `**/.env`, `**/.env.*`
- Private keys: `**/id_rsa{,.pub}`, `**/id_dsa`, `**/id_ecdsa`, `**/id_ed25519`, `**/.ssh/id_*`
- Credentials: `**/.aws/credentials`, `**/credentials.json`, `**/service-account.json`, `**/firebase-adminsdk-*.json`, `**/secrets.{yml,yaml}`
- Extensions: `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`, `*.ppk`, `*.asc`, `*.gpg`
- Any file larger than 100 MB (use git LFS)
If a secret is needed, use `.env.example` for env vars and an external secret manager for credentials.

### Pull requests targeting `main`, `release`, `staging`
All three are protected — a PR is required (direct push blocked):
- 1 approval, all conversations resolved, **squash or rebase merge only** (linear history enforced — no merge commits).
- Commits must be GPG- or SSH-signed. Signing is required for `main` (`required-signatures-main` ruleset).
- The PR **title** becomes the squash commit message and must match the commit-message regex above (enforced on all three branches).

**Required workflows run on PRs whose base is `main` only** (not `release`/`staging`): `Branch naming convention`, `PR title — Conventional Commits`, and `PR size labeler`.
If a check shows `Waiting for workflow to run` for over a minute, the third-party action is likely missing from the enterprise allowlist.

When the branch-naming or PR-title check fails, the baseline bot auto-posts rename/title suggestions, following the enforced regex patterns.
If the bot's suggestions are incorrect, edit the PR title or branch name to match the required format.

### Pre-push checklist
Before running `git push`:
1. Branch name matches the regex.
2. Every commit in `origin/main..HEAD` matches the commit pattern (`git log --format=%s origin/main..HEAD`).
3. No staged file is in the blocked paths/extensions list.
4. Commits are signed if the target is `main`.

If any check fails, fix it locally rather than letting the server reject the push.
