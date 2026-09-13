[![license](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=for-the-badge)](LICENSE)
[![conventional commits](https://img.shields.io/badge/Conventional%20Commits-1.0.0-yellow.svg?style=for-the-badge)](https://conventionalcommits.org)

<p align="center">🔁 Declarative GitHub repo configuration, as a <code>gh</code> extension.</p>

Ship one JSON manifest per org (or per user), commit it to that org's
`.github` repo, and run `gh reposync --apply` to bring every listed
repo's description, topics, security settings, and branch protection
in line. `--check` reports drift without mutating anything.

## ✨ Features

- 🧾 **Per-repo manifest** — each managed repo carries its own
  `.github/repos.json` describing its desired state; run the tool
  from inside the repo and it picks the file up automatically
- 🔍 **Drift detection** — `gh reposync --check` diffs actual vs
  desired and exits non-zero when anything is off, so CI can gate PRs
  that forget to update the manifest
- 🛡️ **Covers the boring bits** — repo settings, description, topics,
  Dependabot / secret scanning / push protection, branch protection
- 🌐 **Cross-owner repos** — per-repo `"owner"` override lets a single
  manifest manage personal repos alongside an org's (e.g. the
  org's `example-org/*` plus your personal `you/*` mixed into one
  manifest)

## 📦 Install

```bash
gh ext install retr0h/gh-reposync
```

Updates land via `gh ext upgrade gh-reposync`.

### 📋 Requirements

- [`gh`](https://cli.github.com/) authenticated (`gh auth login`)
  with `repo` + `admin:org` scopes (the latter only if you manage
  org-scoped branch-protection restrictions)
- [`jq`](https://jqlang.github.io/jq/)
- bash 4+

## 🚀 Usage

```bash
gh reposync --check                    # Diff, exit 1 on drift
gh reposync --apply                    # Apply desired config everywhere
gh reposync --apply --repo service-a   # One repo only
gh reposync --config path/to/m.json \  # Explicit manifest path
            --check
```

### 📂 Config discovery

First match wins:

1. `--config PATH` on the command line
2. `$GH_REPOSYNC_CONFIG` environment variable
3. `./repos.json` in the current directory
4. `./.github/repos.json`

The expected workflow: keep your manifest in `example-org/.github/repos.json`
(GitHub's conventional location for org-wide defaults), `cd` into
that repo, and run `gh reposync --apply`.

## 📝 Manifest schema

See [`repos.example.json`](repos.example.json) for a
complete example. The schema in brief:

```jsonc
{
  "org": "example-org",               // Default owner for repos[]

  "settings": { ... },                // Applied to every repo (gh api PATCH /repos/{owner}/{repo})
  "security": { ... },                // Dependabot + secret-scanning toggles
  "branch_protection": { ... },       // Applied to each repo's configured branch

  "repos": [
    {
      "name": "service-a",            // Required
      "description": "…",             // Optional — skipped if omitted
      "topics": ["x", "y"],           // Optional — sorted, deduped, exact-match
      "owner": "some-user"            // Optional — override the default `org`
    }
  ]
}
```

### ⚠️ User-owned repos

Branch protection `restrictions` (team and user push allowlists) and
`block_creations`, which only works with them, exist on organization repos
alone. GitHub refuses them on a user-owned repo with HTTP 422, and the whole
branch protection request fails.

`gh-reposync` asks GitHub what owns each repo rather than comparing it with
`org`, so a personal manifest whose `org` is your own username works too. On a
user-owned repo it sends `restrictions: null` and `block_creations: false`,
says it skipped them, and checks `block_creations` against `false`. Every other
setting and protection is applied the same way to both.

## ⚙️ What gets synced

| Area | Source of truth | GitHub API |
|---|---|---|
| Repo settings | `.settings.*` | `PATCH /repos/{owner}/{repo}` |
| Description | `.repos[].description` | `PATCH /repos/{owner}/{repo}` |
| Topics | `.repos[].topics` | `PUT /repos/{owner}/{repo}/topics` |
| Dependabot alerts | `.security.vulnerability_alerts` | `PUT/DELETE /repos/{owner}/{repo}/vulnerability-alerts` |
| Dependabot updates | `.security.dependabot_security_updates` | `PATCH /repos/{owner}/{repo}` |
| Secret scanning | `.security.secret_scanning` | `PATCH /repos/{owner}/{repo}` |
| Push protection | `.security.secret_scanning_push_protection` | `PATCH /repos/{owner}/{repo}` |
| Branch protection | `.branch_protection.*` | `PUT /repos/{owner}/{repo}/branches/{branch}/protection` |

Anything not listed — webhooks, environments, actions settings,
collaborators, team access — is **not** managed by this tool. By
design, to keep the manifest focused and the bash code readable.

## 🔐 Required token scopes

```
repo
admin:org        # only if you're setting branch-protection restrictions
```

## 🗺️ Roadmap

- [x] 📦 `gh` extension packaging
- [x] 🔍 `--check` drift detection with non-zero exit
- [x] 🛡️ Branch protection, security features, topics, description, settings
- [ ] 🧪 Real test suite (currently shellcheck only)
- [ ] 🪝 Actions workflow template: `check` on PR, `apply` on merge

## 📄 License

The [MIT][] License.

[MIT]: LICENSE
