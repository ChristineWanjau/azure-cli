---
name: appconfig-dev-setup
description: Set up a local Azure CLI development environment to work on the az appconfig command module using the modern azdev-based flow (fork, clone, virtual env, azdev setup, verify). Use when onboarding to appconfig development, preparing to build/run/test the module locally, or fixing a broken/outdated CLI dev setup.
license: MIT
---
<!-- cspell:words appconfig azconfig configstore azdev venv pyenv virtualenv Scripts pythonpath -->
# appconfig-dev-setup skill

Scope: getting a working local Azure CLI dev environment so `az appconfig`
runs from source and can be tested. Grounded in
`doc/configuring_your_machine.md` and `CONTRIBUTING.rst` — treat both as
authoritative alongside the steps below. This skill only covers
environment setup; for PR title/changelog conventions and SDK version
bumps see the `appconfig-release-process` skill.

## Prerequisites

- A GitHub account. Microsoft contributors: follow
  https://opensource.microsoft.com/ to create, configure, and link your
  account.
- **Fork** https://github.com/Azure/azure-cli into your own account, then
  submit pull requests against `Azure/azure-cli`. (App Configuration CLI
  lives in the public repo — always fork from there.)
- `git` installed and on `PATH`.
- Python **>= 3.10** (see `python_requires` in `src/azure-cli/setup.py`
  for the current floor; 3.10–3.14 are listed as supported).

## 1. Get the source

```pwsh
git clone https://github.com/<your-account>/azure-cli.git
cd azure-cli
```

Run all remaining commands from the clone root unless noted.

## 2. Create and activate a virtual environment

Create it as close to the clone root as possible (e.g. an `env` folder).

```pwsh
python -m venv env
```

Activate it:

- PowerShell (Windows): `env\Scripts\Activate.ps1`
- cmd (Windows): `env\Scripts\activate.bat`
- bash/zsh (macOS/Linux): `. env/bin/activate`

> If PowerShell blocks activation, allow scripts for the current user:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

## 3. Install azdev and run setup

`azdev` is the official tool for developing CLI command modules. With the
virtual environment **activated**:

```pwsh
pip install azdev
azdev setup
```

- `azdev setup` is interactive; to point it at this clone
  non-interactively use `azdev setup -c <clone-root>` (add `-r <path>` if
  you also develop extensions). Running from the clone root, `azdev setup -c .`
  works.
- This installs the CLI and all command modules (including `appconfig`)
  into your virtual environment in editable/development mode, so source
  edits take effect without reinstalling.

## 4. Verify

```pwsh
az
az appconfig -h
```

`appconfig` should appear in the command group list, and
`az appconfig -h` should list its subgroups (`kv`, `feature`, `snapshot`,
`replica`, `credential`, `identity`, ...). You are now ready to develop.

## 5. Common next commands

Run from the clone root with the environment activated:

- Tests: `azdev test appconfig`
- Style: `azdev style appconfig`
- Linter: `azdev linter appconfig`

## Do NOT do these (obsolete steps)

Older internal setup notes are out of date. Avoid the following:

- **Do not remove `azconfig` from the `DEPENDENCIES` list in
  `src/azure-cli/setup.py`.** `azconfig` is no longer a dependency, so
  there is nothing to remove.
- **Do not run `python scripts/dev_setup.py`.** That script is no longer
  supported and will error. Use `pip install azdev` + `azdev setup`
  instead.
- **Do not set `PYTHONPATH` manually** (e.g. to
  `.../azure-cli/src`). `azdev setup` installs the CLI in
  editable/development mode and configures the environment for you.

## Links

- Setup guide: `doc/configuring_your_machine.md`
- Contributing guide: `CONTRIBUTING.rst`
- azdev (Azure CLI Dev Tools): https://github.com/Azure/azure-cli-dev-tools
- Debugging in VS Code: `doc/debug/debug_in_vs_code.md`
