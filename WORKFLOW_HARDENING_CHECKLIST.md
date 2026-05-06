# GitHub Actions Workflow Hardening Checklist

Derived from the [uw-ssec GitHub Actions audit (2026-05-06)](https://github.com/uw-ssec/.github/blob/main/WORKFLOW_HARDENING_CHECKLIST.md), which found 400 findings across 134 workflow files in 45 repositories.

Copy the template workflows in [`.github/workflow-templates/`](.github/workflow-templates/) to get a head start — they demonstrate every pattern below and pass `zizmor` without modifications.

---

## 1. Deny-all permissions at the workflow level

```yaml
# At the top of every workflow file, before any jobs:
permissions: {}
```

Then opt in per job to only what that job needs:

```yaml
jobs:
  test:
    permissions:
      contents: read        # checkout only
  publish:
    permissions:
      id-token: write       # OIDC trusted publishing only
  upload-pages:
    permissions:
      pages: write
      id-token: write
```

**Why:** Without an explicit `permissions:` block, `GITHUB_TOKEN` defaults to the repository or organisation setting, which is often `read/write` on all scopes. Least-privilege limits blast radius if a step is compromised.

**Common scopes and when you need them:**

| Scope | When needed |
|---|---|
| `contents: read` | All checkouts |
| `pull-requests: write` | Posting PR comments |
| `id-token: write` | OIDC trusted publishing (PyPI, npm, cloud) |
| `attestations: write` | Generating build provenance |
| `packages: write` | Pushing to GitHub Container Registry |
| `pages: write` + `id-token: write` | GitHub Pages deploy |
| `security-events: write` | Uploading SARIF to GitHub Security tab |

---

## 2. Pin all actions to full commit SHAs

```yaml
# Bad — tag is mutable; a compromised maintainer can change v4 to point to malicious code
- uses: actions/checkout@v4

# Good — immutable SHA + human-readable comment
- uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
```

**Common uw-ssec action SHAs** (valid as of 2026-05-06 — re-pin when updating):

| Action | SHA | Tag |
|---|---|---|
| `actions/checkout` | `93cb6efe18208431cddfb8368fd83d5badbf9bfd` | v5.0.1 |
| `actions/setup-python` | `a309ff8b426b58ec0e2a45f0f869d46889d02405` | v6.2.0 |
| `actions/upload-artifact` | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` | v7.0.1 |
| `actions/download-artifact` | `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c` | v8.0.1 |
| `pypa/gh-action-pypi-publish` | `cef221092ed1bacb1cc03d23a2d87d1d172e277b` | release/v1 |
| `pre-commit/action` | `2c7b3805fd2a0fd8c1884dcaebf91fc102a13ecd` | v3.0.1 |
| `zizmorcore/zizmor-action` | `b1d7e1fb5de872772f31590499237e7cce841e8e` | v0.5.3 |

**Keeping SHAs up to date:** Enable [Dependabot for GitHub Actions](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) — it opens PRs to bump SHAs automatically:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## 3. Disable credential persistence on checkout

```yaml
- uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
  with:
    persist-credentials: false  # don't write GITHUB_TOKEN to .git/config
```

**Why:** `persist-credentials: true` (the default) writes the `GITHUB_TOKEN` into the local `.git/config`. Any subsequent step that runs untrusted or attacker-influenced code can read and exfiltrate it. Set `persist-credentials: false` on every checkout unless you explicitly need git push in that job.

---

## 4. Never use `pull_request_target` with untrusted code checkout

```yaml
# Dangerous — privileged workflow checks out attacker-controlled code
on:
  pull_request_target:
jobs:
  test:
    steps:
      - uses: actions/checkout@...
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # ← attacker controlled
```

`pull_request_target` runs with write access to the base repository. If you check out the PR's head ref and then execute it (build, test, import), an attacker can exfiltrate secrets or write to your repo.

**Safe alternatives:**

- Use `pull_request` (not `_target`) for CI that runs on fork PRs. It has read-only tokens by default.
- For workflows that need write access (auto-labellers, comment posters): use `pull_request_target` but **do not check out the PR head**. Use `github.event.pull_request.number` as data, not as code.
- For `pull_request_target` + checkout: split into two workflows — an untrusted build stage (`pull_request`) that uploads artifacts, and a trusted post-processing stage triggered by `workflow_run` that downloads and uses those artifacts with minimal permissions.

See [GitHub's security guide for `pull_request_target`](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/).

---

## 5. Pass event data through environment variables

```yaml
# Dangerous — injects PR title directly into shell; title can contain shell metacharacters
- run: echo "PR title: ${{ github.event.pull_request.title }}"

# Safe — pass through env; shell expands $PR_TITLE as a variable, not as code
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR title: $PR_TITLE"
```

**Why:** Direct interpolation of `${{ github.event.* }}` into `run:` steps creates command injection opportunities. Even "safe" fields like PR numbers can be manipulated if they reach a shell command.

**Validating numeric fields:**

```yaml
- env:
    PR_NUMBER: ${{ github.event.pull_request.number }}
  run: |
    if ! [[ "$PR_NUMBER" =~ ^[0-9]+$ ]]; then
      echo "ERROR: unexpected PR number format" >&2; exit 1
    fi
    echo "Processing PR #$PR_NUMBER"
```

---

## 6. Use OIDC trusted publishing for PyPI and npm

Instead of storing long-lived `PYPI_PASSWORD` or `NPM_TOKEN` secrets, use OIDC:

```yaml
publish:
  environment: release        # protected environment with required reviewers
  permissions:
    id-token: write           # only permission needed; no stored secret
  steps:
    - uses: pypa/gh-action-pypi-publish@cef221092ed1bacb1cc03d23a2d87d1d172e277b # release/v1
```

Set up the trusted publisher on PyPI at **pypi.org → Account settings → Publishing → Add a new publisher** with your repository and workflow filename.

**Why:** OIDC tokens are short-lived and scoped to a single job run. Compromising one job cannot reuse the credential. Long-lived secrets in repo settings are accessible from any workflow run in that repo.

**Pattern: split build from publish**

Keep the build job unprivileged (`contents: read` only) and the publish job minimal (`id-token: write` only). The publish job downloads signed artifacts from the build job — it never sees source code.

See [`.github/workflow-templates/hardened-release-pypi.yml`](.github/workflow-templates/hardened-release-pypi.yml) for a complete example.

---

## 7. Protect release environments

```yaml
# in your workflow
publish:
  environment: release
```

Then in **Repository Settings → Environments → release**:
- Add required reviewers (at least one maintainer)
- Optionally restrict to protected branches only

**Why:** Without environment protection, a workflow triggered by `on: release: types: [published]` runs immediately with no human gate. A mistaken tag or a compromised release process can publish to PyPI/npm before anyone notices.

---

## 8. Do not pipe remote scripts to shell

```yaml
# Never do this — you execute whatever the remote URL serves at that moment
- run: curl -fsSL https://example.com/install.sh | bash
```

**Safe alternatives:**

- Download the script, inspect it, commit it, and run the local copy.
- Use a package manager (`pip install`, `npm install`, `brew install`) with a locked dependency manifest.
- Pin the script URL to a content-addressed URL (e.g., a specific commit on GitHub) and verify a checksum before executing.

---

## 9. Enable first-time contributor approval

In **Repository Settings → Actions → General → Fork pull request workflows**:

Select **"Require approval for first-time contributors"** (or the stricter **"Require approval for all outside contributors"** for sensitive repos).

**Why:** Without approval gates, a first-time contributor's PR immediately triggers all workflows in the base repository, including any that receive secrets via `pull_request_target`.

---


## 10. Add `zizmor` workflow scanning to CI

### Preferred: call the org reusable workflow

Create `.github/workflows/zizmor.yml` in your repo:

```yaml
name: Workflow security scan
on:
  pull_request:
    paths: [".github/workflows/**"]
  push:
    branches: [main]
    paths: [".github/workflows/**"]

permissions: {}

jobs:
  lint:
    permissions:
      contents: read
      security-events: write   # SARIF upload to Security tab (remove when enforce: true)
    uses: uw-ssec/.github/.github/workflows/zizmor-lint.yml@main
    with:
      enforce: false   # set to true once all medium+ findings are resolved
```

When `enforce: false` (the default), findings are uploaded to the GitHub Security tab as SARIF but the job always passes — good for capturing a baseline without blocking PRs. Flip to `enforce: true` (and drop `security-events: write`) once all `medium+` findings are resolved or acknowledged with per-line ignores in `zizmor.yml`.

### Alternative: inline job

If you need a self-contained workflow without the reusable dependency:

```yaml
name: Workflow security scan
on:
  pull_request:
    paths: [".github/workflows/**"]
  push:
    branches: [main]
    paths: [".github/workflows/**"]

permissions: {}

jobs:
  zizmor:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write   # SARIF upload; remove when advanced-security: "false"
    steps:
      - uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4
        with:
          persist-credentials: false
      - uses: zizmorcore/zizmor-action@b1d7e1fb5de872772f31590499237e7cce841e8e # v0.5.3
        with:
          inputs: .github/workflows/
          min-severity: medium
          advanced-security: "true"   # report-only; change to "false" to enforce
          annotations: "true"
```

Copy `zizmor.yml` from this repo to the root of your repository and adjust ignore rules as needed.

---

## Quick self-check
Before merging any workflow change, verify:

- [ ] `permissions: {}` at the top of the workflow file
- [ ] Every job has an explicit `permissions:` block with only what it needs
- [ ] Every `uses:` line references a full 40-character SHA (not a tag or branch)
- [ ] Every `uses:` line has a `# tag` comment for human readability
- [ ] Every `actions/checkout` step has `persist-credentials: false`
- [ ] No `${{ github.event.* }}` values interpolated directly into `run:` steps
- [ ] No `pull_request_target` that checks out `github.event.pull_request.head.*`
- [ ] No `curl | bash` or equivalent remote script execution
- [ ] Release workflows use OIDC trusted publishing (no long-lived secrets)
- [ ] Release environment exists with required reviewers
- [ ] `zizmor` passes with `--min-severity medium`

---

## Resources

- [uw-ssec hardened CI template](.github/workflow-templates/hardened-ci.yml)
- [uw-ssec hardened PyPI release template](.github/workflow-templates/hardened-release-pypi.yml)
- [zizmor documentation](https://woodruffw.github.io/zizmor/)
- [GitHub Actions security hardening guide](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Keeping actions up to date with Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot)
- [Preventing Pwn Requests (GitHub Security Lab)](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
- [PyPI trusted publishing](https://docs.pypi.org/trusted-publishers/)
