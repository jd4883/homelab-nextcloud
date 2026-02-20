# CI: Release on merge and release notes

Merge to `main` (chart changes) → **Release on merge to main** creates the next `v*` tag and GitHub Release → **Release notes from PR** fills the release body from the merged PR (OpenAI bullet summary).

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| [release-on-merge.yml](release-on-merge.yml) | Push to `main` (helm/Chart.yaml, helm/values.yaml, helm/README.md, helm/charts/**) | Bump patch version, create tag (e.g. `v0.1.1`) and publish GitHub Release |
| [release-notes.yml](release-notes.yml) | **Release published** or *Release on merge* completes | Set release body from merged PR; OpenAI summarizes into bullet points |

Both use reusable actions from **expectedbehaviors/github-actions** (`@main`). To pin a version, use `@v1` (or another tag) in the workflow files.

## Required secrets

| Secret | Required? | Used by |
|--------|-----------|---------|
| **GITHUB_TOKEN** | No (automatic) | All workflows |
| **OPENAI_API_KEY** | **Yes** (for release notes) | Release notes from PR — workflow fails if unset |

Add **OPENAI_API_KEY** under **Settings → Secrets and variables → Actions** (e.g. from [platform.openai.com/api-keys](https://platform.openai.com/api-keys)).

## What runs when

- **Merge a PR into `main`** that touches the chart under **helm/** (paths above): **Release on merge** runs → next tag and GitHub Release. **Release notes** runs on that release (or when the workflow completes) and sets the release body from the merged PR description (OpenAI summary).
- **Push to `main`** without those paths: no new release.
- **Manual:** Run **Release on merge to main** or **Release notes from PR** from the Actions tab (**Run workflow**). Release notes accepts optional input `release_tag` (e.g. `v0.1.0`, default: latest release).
