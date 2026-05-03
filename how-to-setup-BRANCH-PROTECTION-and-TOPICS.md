# How to Set Up Branch Protection and GitHub Topics

> Covers **repository rulesets** for branch protection and **topic tagging** for discoverability — applied via `gh api` for fast bulk setup across many repos.

---

## Branch Protection

GitHub's **rulesets API** is the modern replacement for the older branch protection rules API. Rulesets support:
- `deletion` — prevents the branch from being deleted
- `non_fast_forward` — prevents force pushes

### Two protection modes

| Mode | Rules | When to use |
|------|-------|-------------|
| **Deletion-only** | `deletion` only | Sync-target public repos — `sync-public.yml` force-pushes on every run |
| **Full** | `deletion` + `non_fast_forward` | All other repos — no force push needed |

### Free plan note

Rulesets on **private repos** require GitHub Pro or higher. If you're on a free plan, the API returns:

```
Upgrade to GitHub Pro or make this repository public to enable this feature.
```

Public repos: rulesets work on all plans.

---

## Applying Branch Protection

### Deletion-only (sync-target public repos)

```bash
echo '{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
  "rules": [{"type": "deletion"}]
}' | gh api repos/OWNER/REPO/rulesets --method POST --input -
```

### Full protection (no force push)

```bash
echo '{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
  "rules": [{"type": "deletion"}, {"type": "non_fast_forward"}]
}' | gh api repos/OWNER/REPO/rulesets --method POST --input -
```

`~DEFAULT_BRANCH` is a GitHub magic ref that always targets the repo's default branch — no need to hardcode `main`.

### Bulk apply to multiple repos

```bash
# Deletion-only — sync-target repos
sync_targets=("my-public-preview" "another-public-mirror")

for repo in "${sync_targets[@]}"; do
  echo '{
    "name": "protect-main",
    "target": "branch",
    "enforcement": "active",
    "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
    "rules": [{"type": "deletion"}]
  }' | gh api repos/OWNER/$repo/rulesets --method POST --input - 2>&1 | \
    jq -r '"'"$repo"': " + (.name // .message)'
done
```

```bash
# Full protection — all other public repos
full_protect=("my-repo-a" "my-repo-b" "my-repo-c")

for repo in "${full_protect[@]}"; do
  echo '{
    "name": "protect-main",
    "target": "branch",
    "enforcement": "active",
    "conditions": {"ref_name": {"include": ["~DEFAULT_BRANCH"], "exclude": []}},
    "rules": [{"type": "deletion"}, {"type": "non_fast_forward"}]
  }' | gh api repos/OWNER/$repo/rulesets --method POST --input - 2>&1 | \
    jq -r '"'"$repo"': " + (.name // .message)'
done
```

### Check existing rulesets

```bash
gh api repos/OWNER/REPO/rulesets | jq '.[] | {id, name, enforcement}'
```

### Delete a ruleset

```bash
gh api repos/OWNER/REPO/rulesets/RULESET_ID --method DELETE
```

---

## GitHub Topics

Topics tag your repo for GitHub search and discoverability. Max 20 per repo.

### Apply topics to a single repo

```bash
gh api repos/OWNER/REPO/topics --method PUT \
  --input - <<< '{"names": ["react", "typescript", "vite", "github-pages"]}'
```

### Bulk apply with a helper function

```bash
apply_topics() {
  local repo=$1
  shift
  local names_json=$(printf '"%s",' "$@" | sed 's/,$//')
  echo "{\"names\": [$names_json]}" | \
    gh api repos/OWNER/$repo/topics --method PUT --input - | \
    jq -r '.names | join(", ")'
}

apply_topics my-react-app react typescript vite github-pages
apply_topics my-solidity-project defi solidity ethereum hardhat
```

### Read current topics

```bash
gh api repos/OWNER/REPO/topics | jq '.names'
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Upgrade to GitHub Pro` | Rulesets not available on free plan for private repos | Make repo public or upgrade plan |
| `Invalid property /rules` | Passing rules as a string, not a JSON array | Use `--input -` with a heredoc or echo — never `--field` for nested objects |
| `~DEFAULT_BRANCH` doesn't match | Repo has no default branch set | Push a commit to `main` first; or replace with `"refs/heads/main"` |
| Topics not appearing in search | GitHub search index lag | Wait a few minutes and search again |
| PUT replaces all topics | Topics API is replace-not-append | Read current topics first, merge arrays, then PUT the combined list |

---

## Which repos need which protection

**Sync-target public repos** — these are force-pushed by `sync-public.yml` on every run. Apply **deletion-only**:

```
*-public-preview
*-public
```

**Everything else** — apply **full protection** (deletion + no force push).

---

*Maintained by [drasticstatic](https://github.com/drasticstatic)*
