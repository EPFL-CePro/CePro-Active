# CePro-Active

Shared GitHub Actions workflows for the CePro team.

The goal is to keep deployments identical across all our repositories, without copy-pasting the same CI/CD boilerplate into each of them. A repository calls
a workflow from here in a few lines, and every improvement or fix merged into `main` reaches all of them.

Callers use `@main`. Since every merge goes live for every repository, changes go through a pull request, and are tested from their branch before merging (see each workflow's README).

| Directory | What it is |
|-----------|------------|
| [`ansible-deploy/`](./ansible-deploy) | Deploy an Ansible "ops" repository to its VM, with secrets from Keybase. Used by the repositories that deploy external apps (e.g. CeProject). |

The workflow files themselves live in [`.github/workflows/`](./.github/workflows), because GitHub only allows reusable workflows there. Each directory at the root holds the documentation and support files for one workflow.
