# ansible-deploy

Reusable workflow that runs an Ansible playbook from the calling repository
against its VM, with secrets taken from Keybase.

- Workflow: [`.github/workflows/ansible-deploy.yml`](../.github/workflows/ansible-deploy.yml)
- Toolchain: [`pyproject.toml`](./pyproject.toml) and [`uv.lock`](./uv.lock) in this directory pin
  the Python and `ansible-core` versions used by every deployment.

## What it does

1. Checks out the calling repository (the "ops" repository with the playbook and inventory).
2. Checks out this directory and installs `ansible-core` from `uv.lock` with [uv](https://docs.astral.sh/uv/).
3. Installs the Galaxy roles and collections from the caller's `requirements.yml`, if it has one.
4. Logs in to Keybase as a bot and copies the secrets directory (e.g. `/keybase/team/ceproject`)
   to a temporary folder on the runner.
5. Runs `ansible-playbook`, passing that folder as the `secrets_dir` variable.
6. Deletes the secrets, the SSH key and the Keybase session, even if the deploy failed.

Two deployments of the same repository never run at the same time: they are queued.

## Usage

In the ops repository, add `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      tags:
        description: Ansible tags
        default: myapp.run
      check_mode:
        description: Dry run (--check), changes nothing on the VM
        type: boolean
        default: true

jobs:
  deploy:
    uses: EPFL-CePro/ansible-workflows/.github/workflows/ansible-deploy.yml@main
    with:
      keybase_dir: /keybase/team/myapp
      tags: ${{ inputs.tags || 'myapp.run' }}
      check_mode: ${{ inputs.check_mode || false }}
    secrets: inherit
```

A push to `main` deploys for real. A manual run from the Actions tab is a dry run
unless you untick "check_mode".

## Inputs

| Input | Default | Description |
|---|---|---|
| `keybase_dir` | *(required)* | Keybase directory holding the secrets. |
| `playbook` | `playbook.yml` | Path to the playbook in the calling repository. |
| `inventory` | `inventory/prod.yml` | Path to the inventory. |
| `requirements` | `requirements.yml` | Galaxy requirements file. The step is skipped if the file does not exist. |
| `tags` | *(empty: whole playbook)* | Comma-separated Ansible tags. |
| `extra_vars` | *(empty)* | Space-separated `key=value` pairs, each passed as `-e`. |
| `check_mode` | `false` | Run with `--check`: Ansible reports what would change, without changing anything. |
| `environment` | `production` | GitHub environment of the job (protection rules, environment secrets). |
| `runs_on` | `["self-hosted", "linux"]` | Runner labels, as JSON. The runner must reach the VM over SSH. |
| `workflows_ref` | `main` | Ref of this repository to take the toolchain from. Keep it equal to the `@ref` in `uses:` (only differs when testing a branch). |

## Secrets

Pass them with `secrets: inherit`. Defining them at the organization level, limited
to the ops repositories, avoids repeating them in each repository.

| Secret | Content |
|---|---|
| `SSH_PRIVATE_KEY` | Private key allowed to log in to the VM (as the `ansible_user` of the inventory). |
| `SSH_KNOWN_HOSTS` | Host key of the VM, e.g. the output of `ssh-keyscan myapp.epfl.ch`. |
| `KEYBASE_USERNAME` | Keybase bot account, member of the team that owns `keybase_dir`. |
| `KEYBASE_PAPERKEY` | Paper key of that bot account. |

## What the ops repository must provide

- **A `secrets_dir` variable for every secret file.** Tasks read secrets with
  `src: "{{ secrets_dir }}/.env"`, never a hard-coded Keybase path. Give it a default
  pointing at your Keybase mount, so local runs keep working:

  ```yaml
  # roles/<role>/defaults/main.yml
  secrets_dir: /keybase/team/myapp
  ```

- **An inventory that works without anyone's `~/.ssh/config`**: set `ansible_host` and
  `ansible_user` explicitly.

- **Every collection in `requirements.yml`.** The toolchain installs `ansible-core` only,
  without the collections bundled in the `ansible` package. Anything outside
  `ansible.builtin` (e.g. `community.general`, `ansible.posix`, `community.docker`) must
  be listed there, preferably with a pinned version.

- **No `--diff` and no task printing secrets.** Secret files are not GitHub secrets, so
  GitHub does not mask them in the logs. Add `no_log: true` to any task that could display
  a secret's value.

## Prerequisites

- A self-hosted runner inside the EPFL network, registered for the organization, since
  GitHub-hosted runners cannot reach our VMs. Keybase is installed on the first run if
  missing (this needs `sudo`); preinstalling it is better.
- A `production` environment in each calling repository, if you want required reviewers
  or to restrict deployments to `main`.
- Callers can read this repository: either it is public (it contains no secrets), or
  *Settings → Actions → General → Access* allows the organization's repositories.

## Running locally

Use the same Ansible version as the CI, from a clone of this repository:

```bash
uv sync --project path/to/ansible-workflows/ansible-deploy
uv run --project path/to/ansible-workflows/ansible-deploy \
  ansible-playbook -i inventory/prod.yml playbook.yml -t myapp.run
```

Run it from the ops repository. Without `-e secrets_dir=...`, the role's default
(your Keybase mount) is used.

## Updating the toolchain

1. Edit the versions in `pyproject.toml` (Python in `requires-python`, `ansible-core`
   in `dependencies`). Read the `ansible-core` porting guides before changing the minor version.
2. Run `uv lock` in this directory and commit `pyproject.toml` and `uv.lock`.
3. Open a pull request and test from its branch: in a caller, set `@my-branch` in `uses:`
   and `workflows_ref: my-branch`, then run it manually in check mode.
4. Merge. Every caller uses `@main`, so the next deployment of each repository picks up the change.

   If a change would break existing callers (new required input, removed input, major
   `ansible-core` jump), add it as a new workflow file (e.g. `ansible-deploy-v2.yml`)
   and migrate callers one by one, instead of changing this one in place.