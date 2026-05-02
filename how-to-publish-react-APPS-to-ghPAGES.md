# How to Publish React Apps to GitHub Pages

> Covers **Create React App (CRA)** and **Vite** — including Web3/Infura apps, the `PUBLIC_REPO_TOKEN` PAT for sync-public workflows, and common failure modes.

---

## Two build systems — two slightly different setups

| | CRA (`react-scripts`) | Vite |
|---|---|---|
| Build output | `build/` | `dist/` (or `build/` if configured) |
| Base path config | `"homepage"` in `package.json` | `--base` flag at build time |
| Artifact path in workflow | `./build` | `./dist` |
| Free public RPC fallback | via `contractUtils.js` | same |

---

## Part 1 — CRA (Create React App)

### Step 1 — Set `homepage` in `package.json`

```json
{
  "homepage": "https://YOUR_USERNAME.github.io/YOUR_REPO_NAME"
}
```

Without this, asset paths are absolute and everything 404s on Pages.

### Step 2 — Add the GitHub Actions workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy to GitHub Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci --legacy-peer-deps

      - run: npm run build
        env:
          REACT_APP_INFURA_KEY: ${{ secrets.INFURA_API_KEY }}   # omit if not Web3
          CI: false   # prevents warnings being treated as errors

      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./build   # CRA outputs to build/

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> `CI: false` is important — CRA treats lint warnings as errors by default, which breaks builds on GitHub's runner.

### Step 3 — Set Pages source to GitHub Actions

GitHub defaults to Jekyll (branch-based). You must switch it:

**Via GitHub CLI (recommended):**
```bash
gh api repos/YOUR_USERNAME/YOUR_REPO/pages --method PUT --field build_type=workflow
```

**Via UI:** Repo → Settings → Pages → Source → **GitHub Actions**

> If you skip this step, GitHub's Jekyll builder runs instead and crashes on JSX/Liquid syntax conflicts.

---

## Part 2 — Vite

### Step 1 — No `homepage` needed

Vite doesn't read `package.json`'s `homepage`. Base path is passed at build time with `--base`.

### Step 2 — Add the GitHub Actions workflow

```yaml
name: Build and Deploy to GitHub Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - run: npm run build -- --base=/YOUR_REPO_NAME/
        env:
          VITE_INFURA_KEY: ${{ secrets.INFURA_API_KEY }}   # omit if not Web3

      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist   # Vite outputs to dist/ by default

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> The `--base=/YOUR_REPO_NAME/` must match your repo name exactly, including the trailing slash. Without it, JS/CSS assets load from the root and 404.

### Step 3 — Set Pages source to GitHub Actions

Same as CRA — must be switched from Jekyll to GitHub Actions:

```bash
gh api repos/YOUR_USERNAME/YOUR_REPO/pages --method PUT --field build_type=workflow
```

---

## Part 3 — Web3 Apps (Infura / Public RPC Fallback)

For DApps that read on-chain data, set up a read-only provider fallback so the app loads even without MetaMask:

**`src/utils/contractUtils.js`:**
```javascript
import { BrowserProvider, JsonRpcProvider, Contract } from 'ethers';

const RPC_URL = import.meta.env.VITE_INFURA_KEY          // Vite
  ?? process.env.REACT_APP_INFURA_KEY                     // CRA
  ? `https://sepolia.infura.io/v3/${import.meta.env.VITE_INFURA_KEY ?? process.env.REACT_APP_INFURA_KEY}`
  : 'https://rpc.sepolia.org';  // free public fallback — no key needed

export const getProvider = () =>
  window.ethereum ? new BrowserProvider(window.ethereum) : new JsonRpcProvider(RPC_URL);

export const getContract = (address, abi) =>
  new Contract(address, abi, getProvider());

export const truncateAddress = (address) =>
  address ? `${address.slice(0, 6)}...${address.slice(-4)}` : '';
```

### Adding `INFURA_API_KEY` as a repo secret

```bash
gh secret set INFURA_API_KEY --repo YOUR_USERNAME/YOUR_REPO
# Paste your key when prompted
```

Or via UI: Repo → Settings → Secrets and variables → Actions → New repository secret → `INFURA_API_KEY`

> The app works without this secret — it falls back to the free public RPC. The Infura key just gives you a private, rate-limit-free endpoint.

---

## Part 4 — `PUBLIC_REPO_TOKEN` for sync-public.yml

If your repo uses `sync-public.yml` to mirror to a public preview repo, the workflow needs a Personal Access Token to push cross-repo. This is **separate** from GitHub Pages — it's only needed for the sync pipeline.

### Step 1 — Create a Fine-grained PAT

1. Go to [github.com/settings/tokens?type=beta](https://github.com/settings/tokens?type=beta)
2. Click **Generate new token**
3. Name it something like `sync-public-token`
4. Set **Expiration** (90 days recommended — add a calendar reminder to rotate)
5. Under **Repository access** → select **Only select repositories** → pick your **public** preview repo(s) only
6. Under **Permissions** → **Contents** → **Read and Write**
7. Click **Generate token** and **copy it immediately** — GitHub won't show it again

### Step 2 — Add the token as a secret in your **private** repo

```bash
# Interactively — paste your token when prompted
gh secret set PUBLIC_REPO_TOKEN --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
```

Or via UI: Private repo → Settings → Secrets and variables → Actions → New repository secret → `PUBLIC_REPO_TOKEN`

### Step 3 — Re-run the failed workflow

```bash
gh workflow run sync-public.yml --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
```

Or via UI: Actions tab → select the failed run → Re-run jobs.

### Multiple public repos — one PAT or many?

One PAT can cover multiple repos if you select all of them under "Repository access" in Step 1. Simpler to maintain: one secret, one rotation reminder.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Jekyll build fails with Liquid syntax error | Pages source set to branch/Jekyll, not GitHub Actions | `gh api repos/OWNER/REPO/pages --method PUT --field build_type=workflow` |
| 404 on JS/CSS assets | Missing base path | CRA: add `homepage` to `package.json` · Vite: pass `--base=/REPO_NAME/` |
| `CI: false` missing | Warnings treated as errors | Add `CI: false` to build env in workflow (CRA only) |
| `Authentication failed` in sync workflow | `PUBLIC_REPO_TOKEN` secret not set or expired | See Part 4 above |
| App loads but contract calls fail | No wallet + no fallback RPC | Add `JsonRpcProvider` fallback in `contractUtils.js` |
| Build succeeds but Pages still shows old version | Pages deployment lag | Wait 1–2 min, hard refresh (`Cmd+Shift+R`) |

---

## Templates

Ready-to-copy workflow files live in [`drasticstatic/my-template`](https://github.com/drasticstatic/my-template):
- `workflow-templates/sync-public-allowlist.yml`
- `workflow-templates/sync-public-excludelist.yml`

For the full GitExporter + sync-public pipeline, see [`how-to-setup-GITEXPORTER.md`](how-to-setup-GITEXPORTER.md).

---

*Maintained by [drasticstatic](https://github.com/drasticstatic)*
