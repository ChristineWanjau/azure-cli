---
name: appconfig-release-process
description: Format PR titles/changelog notes correctly for az appconfig changes (HISTORY.rst is auto-generated, never hand-edited), bump the azure-appconfiguration/azure-mgmt-appconfiguration SDK dependency version, and open the change as a draft pull request against Azure/azure-cli following the CLI guidelines. Use when finishing a PR for appconfig, opening a draft PR, or when a change needs newer SDK functionality.
license: MIT
---
<!-- cspell:words appconfig azconfig configstore kwargs preflight upstream -->
# appconfig-release-process skill

Scope: how appconfig changes get released — PR title/changelog
conventions, SDK dependency version bumps, generating test recordings
for new/changed scenario tests, and opening the pull request (always as
a **draft**). Grounded in
`doc/authoring_command_modules/README.md` ("Submitting Pull Requests"),
`doc/how_to_bump_SDK_version_in_cli.md`, `doc/authoring_tests.md`, and
the repo PR template `.github/pull_request_template.md` — treat all four
as authoritative alongside the rules below.

## Changelog — do NOT hand-edit `src/azure-cli/HISTORY.rst`

`HISTORY.rst` entries are auto-generated from the PR title/description
starting from S165 (01/30/2020) — they are not edited directly in normal
PRs. Instead, the PR title must follow this format:

```
[Component Name] [BREAKING CHANGE: |Fix #N: ]<optional az command:> <Verb> <description>
```

- **`[Component Name]`** (e.g. `[App Configuration]`) = customer-facing;
  the message goes into `HISTORY.rst`. **`{Component Name}`** (curly
  braces) = not customer-facing; excluded from `HISTORY.rst`. This part
  is mandatory in every PR title.
- If it's a breaking change, the second part is `BREAKING CHANGE:`. For a
  hotfix, use `Hotfix`. For an issue fix, use `Fix #<number>`. Otherwise
  this part can be empty.
- Recommended: include the affected command starting with `az`, followed
  by a colon (e.g. `az appconfig create:`).
- Recommended: use a present-tense, capitalized, base-form verb:
  - `Add` — new features.
  - `Change` — changes to existing functionality.
  - `Deprecate` — once-stable features slated for removal.
  - `Remove` — deprecated features removed in this release.
  - `Fix` — bug fixes.

Examples:
```
[App Configuration] BREAKING CHANGE: az appconfig create: Remove --deprecated-arg
[App Configuration] Fix #12345: az appconfig kv list: Fix pagination for large stores
{App Configuration} Add help example for kv set
```

- For **multiple** history notes from one PR, or to **override** the
  title-derived note, use the `History Notes` section of the PR
  description (the PR template already includes this section — delete it
  if not needed). The PR title still must start with
  `[Component Name]`/`{Component Name}` even if it's just a summary in
  this case.
- **Hotfix PRs** (based on the `release` branch) are the *only* case
  where `HISTORY.rst` is manually edited, and only for customer-facing
  changes — the auto-generation process ignores PRs whose title contains
  `Hotfix`. Confirm with the user before treating a change as a hotfix;
  it's rare and follows a distinct branch/merge workflow (merge
  `release` back to `dev` with a merge commit, never squash).

## SDK dependency version bumps

If a change needs new functionality from `azure-mgmt-appconfiguration`
(or a data-plane SDK) that isn't in the currently pinned version:

1. Bump the version in `src/azure-cli/setup.py` and all three
   per-OS requirement files: `requirements.py3.windows.txt`,
   `requirements.py3.Linux.txt`, `requirements.py3.Darwin.txt`.
2. Only if the SDK is **multi-API-profile aware**: update the pinned API
   version in `AZURE_API_PROFILES` for the `'latest'` profile in
   `azure-cli-core/azure/cli/core/profiles/_shared.py` (single API
   version as a plain string, or an `operation=version` mapping for
   `SDKProfile`-style multi-operation SDKs).
3. Run a regression check after bumping: `azdev test --no-exitfirst`
   (playback) to catch anything broken by the new SDK; failures that
   only reproduce live should be re-run with
   `azdev test --live --lf --no-exitfirst`.
4. Fix any regressions the bump surfaces, or add the new feature code
   that depends on the bumped SDK.
5. There is also an internal "Regression Test Pipeline" that automates
   steps 1–3 across the whole repo when bumping broadly-used SDKs — flag
   this option to the user if the bump is large/repo-wide rather than
   appconfig-specific, but for an appconfig-only bump, doing steps 1–4
   directly is usually simpler.

Do not assume unreleased SDK APIs exist without checking the actually
installed/pinned package version first.

## Sending out the PR — always open it as a **draft**

Every appconfig PR opened via this skill **MUST** be created as a draft
(`gh pr create --draft`), never directly as ready-for-review. Draft is
the intended safety state: CI still runs and reviewers can preview, but
nothing signals the change is finished. The author flips it to "Ready
for review" manually once checks pass and they're satisfied.

### 1. Preflight — style, lint, and (re)generate recordings

Run from the clone root with the dev virtual environment activated (see
the `appconfig-dev-setup` skill):

```pwsh
azdev style appconfig
azdev linter appconfig
```

**Generating recordings is part of authoring the test — not a follow-up.**
A new or changed `ScenarioTest` method has no (or a stale) cassette under
`tests/latest/recordings/`, so it can only pass by recording it live
first. Never hand-write, fabricate, or trim a `.yaml` cassette, and never
relax assertions to make a test pass without a real recording. Record the
affected tests live, then replay the whole module in playback to confirm
the new cassettes pass deterministically:

```pwsh
# Live run writes tests/latest/recordings/<test_method>.yaml
azdev test <test_method> --live --no-exitfirst   # e.g. test_azconfig_kv_description
# Then confirm the module is green in playback (default mode) with the new cassettes
azdev test appconfig --no-exitfirst
```

Live-recording prerequisites (see the test comments and `_test_utils.py`):

- `az login` as a principal holding **App Configuration Data Owner** on
  the target resource group — data-plane tests use `--auth-mode login`
  against a store with local auth disabled. Point tests at a suitable
  group with `AZURE_CLI_APPCONFIG_TEST_RG`, and optionally prefix
  resource names with `AZURE_CLI_LOCAL_TEST_RESOURCE_PREFIX`.
- The live service must actually support the feature under test (e.g. the
  key-value/snapshot `description` property needs the App Configuration
  data-plane API version the pinned SDK targets) — otherwise the live run
  cannot produce a valid recording.

Then **verify each new cassette is scrubbed** (no real store names,
endpoints, connection strings, or secrets — the module's
`CredentialResponseSanitizer`, `OperationLocationSanitizer`, and name
replacers handle most of this) and **commit the generated
`tests/latest/recordings/*.yaml` files together with the code change.**
Fix any remaining failures before proceeding; do not open a PR whose new
tests only pass live.

### 2. Branch off `dev` — never commit on `dev`/`release`

- PRs target the `Azure/azure-cli` **`dev`** branch. (Hotfixes are the
  only exception — see the Hotfix note above — and are never
  draft-automated by this skill.)
- Keep `origin` = your fork and `upstream` = `Azure/azure-cli`, and work
  on a feature branch:

```pwsh
git switch dev
git pull upstream dev
git switch -c appconfig/<short-topic>
git add -A
git commit -m "<same wording as the PR title>"
```

### 3. Push to your fork, not upstream

```pwsh
git push -u origin appconfig/<short-topic>
```

Never push branches directly to `Azure/azure-cli`.

### 4. Open the draft PR with a compliant title

Target `Azure/azure-cli:dev`, reuse the repo PR template body
(`.github/pull_request_template.md`), and fill in **Related command**,
the mandatory **Description**, the **Testing Guide**, and the checklist;
keep or edit **History Notes** (only if you need to override/extend the
title-derived note):

```pwsh
gh pr create --draft `
  --repo Azure/azure-cli `
  --base dev `
  --title "[App Configuration] az appconfig <cmd>: <Verb> <description>" `
  --body-file <path-to-filled-pr-body>
```

- The title MUST follow the format in the "Changelog" section above
  (`[App Configuration] ...` for customer-facing, `{App Configuration}`
  otherwise). **Confirm the final title with the user before running.**
- If `gh` is unavailable, do steps 1–3 and then print the fork compare
  URL plus the exact title and body for the user to open the draft
  manually — still as a draft, never ready-for-review.

## Guardrails

- Never hand-edit `HISTORY.rst` outside of a confirmed hotfix PR.
- Always surface the required PR title format to the user rather than
  silently choosing one — get their confirmation on the Component Name
  and BREAKING CHANGE/Fix # portion, since only they know the full PR
  context.
- When bumping an SDK version, update all three OS-specific requirements
  files together — never just one.
- **Always open the PR as a draft** (`--draft`); never open it
  ready-for-review or enable auto-merge on the agent's behalf.
- Push only to the contributor's fork (`origin`) and target the `dev`
  base branch — never push to, or base the PR on, anything but
  `Azure/azure-cli:dev` (hotfixes excepted).
- Get the user's confirmation on the final PR title (and the fork/base)
  before pushing or running `gh pr create`.
- Don't open the PR while `azdev style/linter/test appconfig` is failing
  without explicitly calling out the failure in the PR **Description**.
- New or changed `ScenarioTest` cases must ship with **committed,
  scrubbed recordings** produced from a live run — never hand-write or
  fabricate a cassette, and never relax assertions to avoid recording.
