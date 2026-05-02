# How to Set Up GitExporter + sync-public.yml

> **Goal:** Share selected parts of a private repo publicly — preserving full commit history — without exposing source code, secrets, or proprietary logic.

---

## The Problem

You have a private repo with months of commit history. You want a public-facing version that shows:
- A polished README
- Selected docs or examples
- Proof you've been working on the project over time

But you don't want to expose your source code, strategy files, API keys, or anything private.

**Two tools solve this — use them together:**

| Tool | When to use |
|------|-------------|
| **gitexporter** (local CLI) | One-time or on-demand local preview — useful before CI is wired up |
| **sync-public.yml** (GitHub Actions) | Automated — runs on every push, keeps public repo continuously up to date |

---

## Tool 1 — gitexporter (Local CLI)

### What it does

gitexporter traverses your **entire commit tree**. For every historical commit, it selectively keeps only the allowed files and recreates the commit in a new target repo. The result: a real Git repo with real history — not a ZIP export.

### Install + run

No installation needed:

```bash
npx gitexporter gitexporter.config.json
```

### Config file

Create `gitexporter.config.json` at your repo root:

```json
{
  "forceReCreateRepo": true,
  "sourceRepoPath": ".",
  "targetRepoPath": "../YOUR_REPO-public-preview",
  "ignoredPaths": [
    "CLAUDE.md",
    ".augmentignore",
    ".gitignore",
    ".github/",
    "AGENT-SYNC/",
    "specs/",
    "logs/",
    "strategies/",
    "gitexporter.config.json",
    "src/",
    "scripts/"
  ]
}
```

> **Add `gitexporter.config.json` to your `.gitignore`** — it reveals your private directory structure.

### Push the result to GitHub

After running, gitexporter creates a fully initialized local Git repo:

```bash
cd ../YOUR_REPO-public-preview
gh repo create YOUR_USERNAME/YOUR_REPO-public-preview --public
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO-public-preview.git
git push origin main
```

---

## Tool 2 — sync-public.yml (GitHub Actions)

This GitHub Action runs automatically on every push to `main`. It rewrites history using `git filter-repo` and force-pushes the filtered result to your public repo.

### Two models — pick one

#### Allowlist model (strict — private by default)
Everything is private unless explicitly listed as public. Best when most of your repo is sensitive.

```yaml
# .github/workflows/sync-public.yml
name: 🌐 Sync Public Preview

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - run: pip install git-filter-repo

      - name: Validate root path classification
        shell: bash
        run: |
          set -euo pipefail
          public_allowlist=("README.md" "LICENSE" "docs")
          private_allowlist=(".augmentignore" ".claude" ".github" ".gitignore" "CLAUDE.md" "AGENT-SYNC" "specs")

          is_listed() {
            local entry="$1"; shift
            for item in "$@"; do [[ "$item" == "$entry" ]] && return 0; done
            return 1
          }

          unclassified=()
          while IFS= read -r entry; do
            [[ "$entry" == ".git" ]] && continue
            is_listed "$entry" "${public_allowlist[@]}" && continue
            is_listed "$entry" "${private_allowlist[@]}" && continue
            unclassified+=("$entry")
          done < <(find . -maxdepth 1 -mindepth 1 | sed 's|^\./||' | sort)

          if ((${#unclassified[@]} > 0)); then
            echo "Unclassified paths — classify before proceeding:"
            printf ' - %s\n' "${unclassified[@]}"
            exit 1
          fi

      - name: Filter private content
        run: |
          git filter-repo \
            --path "CLAUDE.md" \
            --path ".augmentignore" \
            --path ".gitignore" \
            --path ".github/" \
            --path "AGENT-SYNC/" \
            --path "specs/" \
            --invert-paths \
            --filename-callback 'return None if filename is None or filename.startswith(b".claude/") else filename' \
            --force

      - name: Push to public repo
        env:
          TOKEN: ${{ secrets.PUBLIC_REPO_TOKEN }}
        run: |
          git config user.name "Public Mirror Bot"
          git config user.email "actions@github.com"
          git remote add origin https://x-access-token:${TOKEN}@github.com/YOUR_USERNAME/YOUR_PUBLIC_REPO.git
          git push origin main --force
```

#### Exclude list model (open — public by default)
Everything is public unless explicitly listed as private. Best when most of your repo is shareable with only a few things to hide.

```yaml
      - name: Strip private content
        run: |
          git filter-repo \
            --path "CLAUDE.md" \
            --path ".augmentignore" \
            --path ".github/" \
            --path "AGENT-SYNC/" \
            --path "secrets/" \
            --path "strategies/" \
            --invert-paths \
            --filename-callback 'return None if filename is None or filename.startswith(b".claude/") else filename' \
            --force
```

### Setup steps

#### Step 1 — Create a Personal Access Token (PAT) — classic

Use a **classic** token, not fine-grained. Classic tokens are simpler and work reliably across all repos.

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. **Note:** give it a descriptive name per sync pair — e.g. `wilson-lawn-sync`, `trading-bot-sync` (one token per private→public pair keeps rotation and revocation clean)
3. **Expiration:** your choice — no expiration is convenient but set a calendar reminder to rotate annually if you go that route
4. **Scope:** check **`repo`** (the top-level checkbox). If your sync workflow injects a `.github/workflows/` file into the public repo (e.g. a `deploy.yml` for GitHub Pages), also check **`workflow`** — GitHub blocks pushing workflow files without it. You can add this scope later by editing the token in place; the token value does not change.
5. Click **Generate token** and **copy it immediately** — GitHub will not show it again

> **Why classic over fine-grained?** Fine-grained tokens require selecting repos individually and have more complex permission surfaces. Classic `repo` scope is the established pattern for CI/CD cross-repo push workflows and is what the GitHub Actions ecosystem expects.

> **One token per sync pair** — if you reuse the same token across multiple repos, revoking or rotating it silently breaks all of them. Naming tokens after their purpose (e.g. `divorce-custody-sync`) makes this immediately obvious in the token list.

#### Step 2 — Add the token as a secret in your **private** repo

**Use a separate terminal window** — not a Claude Code session or any tool that logs commands. Running `gh secret set` interactively in a clean terminal masks the token with `*****` and keeps it out of shell history and any session logs.

```bash
gh secret set PUBLIC_REPO_TOKEN --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
# → prompts: "Paste your secret: " — type/paste your token, it shows as ****
# → confirms: "✓ Set Actions secret PUBLIC_REPO_TOKEN for YOUR_USERNAME/YOUR_PRIVATE_REPO"
```

> **Avoid `--body "token"` on the command line** — the token value lands in your `.zsh_history` in plain text. The interactive prompt is safer.

Verify it was set:
```bash
gh secret list --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
# Should show: PUBLIC_REPO_TOKEN   Updated YYYY-MM-DD
```

Or via UI: Private repo → Settings → Secrets and variables → Actions → New repository secret → Name: `PUBLIC_REPO_TOKEN`

> **If your sync injects a GitHub Pages deploy workflow:** See [Part 4 of the GitHub Pages guide](./how-to-publish-react-APPS-to-ghPAGES.md#part-4--public_repo_token-for-sync-publicyml) for the full PAT setup in that context, including when `workflow` scope is needed and how to add it without regenerating your token.

#### Step 3 — Create the public repo

```bash
gh repo create YOUR_USERNAME/YOUR_PUBLIC_REPO --public
```

#### Step 4 — Push your workflow and trigger it

```bash
git add .github/workflows/sync-public.yml
git commit -m "feat: add sync-public workflow"
git push origin main
```

The Action will run automatically. Check the **Actions** tab to see it.

---

## Key Concept: gitexporter vs sync-public.yml

| | gitexporter | sync-public.yml |
|---|---|---|
| **Trigger** | Manual (`npx gitexporter`) | Automatic (every push) |
| **History** | Full commit tree preserved | Full history via filter-repo |
| **Setup** | One JSON file | PAT + repo secret + YAML workflow |
| **Use case** | Bootstrap / preview | Production continuous sync |

**Recommended flow:** Use `gitexporter` to verify your config looks right locally, then set up `sync-public.yml` for ongoing automation.

---

## Security Notes

- `gitexporter.config.json` itself reveals your private directory structure — add it to `.gitignore`
- `sync-public.yml` and `git filter-repo` physically remove files from history — users cannot roll back to old commits to find hidden code
- The PAT should have the minimum permissions needed (Contents: Read/Write on the target public repo only)
- If you accidentally commit a secret, use [`git filter-repo`](https://github.com/newren/git-filter-repo) to scrub it before running sync

---

## Badge for your public README

Let visitors know the repo is a filtered mirror:

```markdown
[![Synced via GitExporter](https://img.shields.io/badge/synced-GitExporter-blue?style=flat-square)](https://github.com/open-condo-software/gitexporter)
```

---

## Templates

Ready-to-use config files live in [`drasticstatic/my-template`](https://github.com/drasticstatic/my-template):
- `gitexporter.config.json`
- `.github/workflows/sync-public-allowlist.yml`
- `.github/workflows/sync-public-excludelist.yml`

---

*Maintained by [drasticstatic](https://github.com/drasticstatic)*
