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

### Blank white page — the no-wallet crash

**Symptom:** Build succeeds, Pages is live, but the site renders a completely blank white page for visitors without MetaMask.

**Cause:** `new ethers.providers.Web3Provider(window.ethereum)` throws a `TypeError` when `window.ethereum` is `undefined` (no wallet injected). The unhandled error crashes React before anything renders.

This commonly lives in `App.js` directly or in a `loadProvider` / `interactions.js` helper:
```javascript
// ❌ Crashes without MetaMask
const provider = new ethers.providers.Web3Provider(window.ethereum)
```

**Fix — Part 1: guard the provider instantiation**

Replace the bare call with a conditional:
```javascript
// ✅ Falls back to read-only public RPC when no wallet present
const provider = window.ethereum
  ? new ethers.providers.Web3Provider(window.ethereum)
  : new ethers.providers.JsonRpcProvider('https://rpc.sepolia.org')
```

If the provider is created in a Redux `interactions.js` file (e.g. a `loadProvider` action), apply the same guard there.

**Fix — Part 2: guard `window.ethereum` event listeners**

After creating the provider, event listeners like `chainChanged` / `accountsChanged` also crash without a wallet:
```javascript
// ❌ Crashes without MetaMask
window.ethereum.on('chainChanged', () => { window.location.reload() })

// ✅ Safe
if (window.ethereum) {
  window.ethereum.on('chainChanged', () => { window.location.reload() })
  window.ethereum.on('accountsChanged', async () => { await loadAccount(dispatch) })
}
```

**Fix — Part 3: `isLoading` stuck at `true` — spinner that never resolves**

Many DApps initialize with `isLoading: true` and rely on `loadBlockchainData` to flip it to `false` once data loads. Without MetaMask, there are two common patterns that leave the spinner stuck permanently — the user sees a spinning "Loading Data..." or a blank white page:

**Pattern A — early return before `setIsLoading(false)`:**
```javascript
// ❌ Returns before spinner is cleared
const loadBlockchainData = async () => {
  if (!window.ethereum) {
    console.error('MetaMask not found')
    return  // isLoading stays true forever
  }
  // ...
  setIsLoading(false)
}

// ✅ Clear the spinner on every exit path
const loadBlockchainData = async () => {
  if (!window.ethereum) {
    setIsLoading(false)  // ← add this
    return
  }
  // ...
  setIsLoading(false)
}
```

**Pattern B — `useEffect` guards loading behind `window.ethereum`:**
```javascript
// ❌ Without wallet the else branch never runs — isLoading stuck
useEffect(() => {
  if (isLoading && window.ethereum) {
    loadBlockchainData()
  }
}, [isLoading])

// ✅ Explicitly clear loading when no wallet present
useEffect(() => {
  if (isLoading && window.ethereum) {
    loadBlockchainData()
  } else if (isLoading && !window.ethereum) {
    setIsLoading(false)  // ← add this
  }
}, [isLoading])
```

**Wrap async load in try/catch to prevent unhandled rejections:**

If the app connects via public RPC fallback but the config only has local Hardhat chainId (31337), contract lookups like `config[chainId].token.address` will throw when `chainId` is `11155111`. Without a try/catch this becomes an unhandled rejection:
```javascript
const loadBlockchainData = async () => {
  try {
    const provider = await loadProvider(dispatch)
    const chainId = await loadNetwork(provider, dispatch)
    await loadTokens(provider, chainId, dispatch)  // throws if chainId not in config
  } catch (err) {
    console.warn('Demo mode: contracts not deployed to this network', err)
    // app renders with empty Redux state — UI visible, no data
  }
}
```

**Fix — Part 4: wire in a `DAppGuard` component**

A `DAppGuard` wrapper shows a friendly "No wallet detected — install MetaMask" banner instead of a blank page, while still rendering the app in read-only mode. Wrap `<App />` in `index.js`:

```jsx
// src/index.js
import DAppGuard from './components/DAppGuard';

root.render(
  <React.StrictMode>
    <DAppGuard>
      <App />
    </DAppGuard>
  </React.StrictMode>
);
```

**`src/components/DAppGuard.js`:**
```jsx
import React, { useState, useEffect } from 'react';

const REQUIRED_CHAIN_ID = '0xaa36a7'; // Sepolia — update to match your deployment

function DAppGuard({ children }) {
  const [hasWallet, setHasWallet] = useState(null);
  const [correctNetwork, setCorrectNetwork] = useState(true);

  useEffect(() => {
    if (!window.ethereum) { setHasWallet(false); return; }
    setHasWallet(true);
    const checkNetwork = async () => {
      const chainId = await window.ethereum.request({ method: 'eth_chainId' });
      setCorrectNetwork(chainId === REQUIRED_CHAIN_ID);
    };
    checkNetwork();
    window.ethereum.on('chainChanged', (id) => setCorrectNetwork(id === REQUIRED_CHAIN_ID));
  }, []);

  if (hasWallet === null) return null; // still detecting

  return (
    <>
      {!hasWallet && (
        <div className="alert alert-warning text-center mb-0" role="alert" style={{ borderRadius: 0 }}>
          <strong>No wallet detected.</strong>{' '}
          Install <a href="https://metamask.io/" target="_blank" rel="noopener noreferrer">MetaMask</a> to
          interact. <em>Read-only mode — live data still loads via public RPC.</em>
        </div>
      )}
      {hasWallet && !correctNetwork && (
        <div className="alert alert-danger text-center mb-0" role="alert" style={{ borderRadius: 0 }}>
          <strong>Wrong network.</strong> Switch to <strong>Sepolia Testnet</strong> to interact.
        </div>
      )}
      {children}
    </>
  );
}

export default DAppGuard;
```

> If your app uses Redux, `DAppGuard` should sit inside `<Provider store={store}>` so child components can still access the store while the guard banner is shown.

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

> **Full PAT setup and naming conventions** are covered in [Step 1 of the GitExporter guide](./how-to-setup-GITEXPORTER.md#step-1--create-a-personal-access-token-pat--classic). The summary below covers the scopes specific to this context.

### Step 1 — Create a classic PAT

Use a **classic** token, not fine-grained. Classic tokens are simpler and work reliably across all repos.

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Name it after the sync pair — e.g. `wilson-lawn-sync`, `trading-bot-sync`
3. **Expiration:** your choice — set a calendar reminder to rotate annually
4. **Scopes:**
   - Check **`repo`** (top-level) — required for all sync workflows
   - Also check **`workflow`** if your sync injects a `.github/workflows/` file into the public repo (e.g. a `deploy.yml` for GitHub Pages). Without it, GitHub rejects the push with: *"refusing to allow a PAT to create or update workflow files without `workflow` scope"*
5. Click **Generate token** and **copy it immediately** — GitHub will not show it again

> **Adding `workflow` scope later:** You can edit an existing token at [github.com/settings/tokens](https://github.com/settings/tokens), check `workflow`, and save. The token value does not change — no need to update the secret.

### Step 2 — Add the token as a secret in your **private** repo

**Use a separate terminal window** — not a Claude Code session or any tool that logs commands. Running `gh secret set` interactively in a clean terminal masks the token with `*****` and keeps it out of shell history and session logs.

```bash
gh secret set PUBLIC_REPO_TOKEN --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
# → prompts: "Paste your secret: " — type/paste your token, it shows as ****
# → confirms: "✓ Set Actions secret PUBLIC_REPO_TOKEN for YOUR_USERNAME/YOUR_PRIVATE_REPO"
```

> **Avoid `--body "token"` on the command line** — the token value lands in `.zsh_history` in plain text.

Or via UI: Private repo → Settings → Secrets and variables → Actions → New repository secret → `PUBLIC_REPO_TOKEN`

### Step 3 — Re-run the failed workflow

```bash
gh workflow run sync-public.yml --repo YOUR_USERNAME/YOUR_PRIVATE_REPO
```

Or via UI: Actions tab → select the failed run → Re-run jobs.

### Multiple public repos — one PAT or many?

One classic PAT can cover multiple repos — no repo selection step like fine-grained tokens. One token per sync pair keeps rotation and revocation clean: if you name it `trading-bot-sync` you know exactly what breaks when you rotate it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Jekyll build fails with Liquid syntax error | Pages source set to branch/Jekyll, not GitHub Actions | `gh api repos/OWNER/REPO/pages --method PUT --field build_type=workflow` |
| 404 on JS/CSS assets | Missing base path | CRA: add `homepage` to `package.json` · Vite: pass `--base=/REPO_NAME/` |
| `CI: false` missing | Warnings treated as errors | Add `CI: false` to build env in workflow (CRA only) |
| `Authentication failed` in sync workflow | `PUBLIC_REPO_TOKEN` secret not set or expired | See Part 4 above |
| `refusing to allow a PAT to create or update workflow files` | Token missing `workflow` scope | Edit token at github.com/settings/tokens → check `workflow` → save (token value unchanged) |
| App loads but contract calls fail | No wallet + no fallback RPC | Add `JsonRpcProvider` fallback in `contractUtils.js` |
| Build succeeds, Pages live, but **blank white page** | `new Web3Provider(window.ethereum)` throws when no wallet — unhandled crash | Guard provider instantiation + event listeners + wrap with `DAppGuard` — see Part 3 |
| Spinner ("Loading Data...") never resolves — no content appears | `isLoading` stuck `true`: early return skips `setIsLoading(false)`, or `useEffect` guards load behind `window.ethereum` | Add `setIsLoading(false)` before every early return; add `else if (!window.ethereum) setIsLoading(false)` to useEffect — see Part 3 |
| App renders empty UI with no data | Config only has local chainId (31337); public RPC returns different chainId → `config[chainId]` is `undefined` → throws | Wrap `loadBlockchainData` in try/catch — app renders UI shell in demo mode — see Part 3 |
| Build succeeds but Pages still shows old version | Pages deployment lag | Wait 1–2 min, hard refresh (`Cmd+Shift+R`) |

---

## Templates

Ready-to-copy workflow files live in [`drasticstatic/my-template`](https://github.com/drasticstatic/my-template):
- `workflow-templates/sync-public-allowlist.yml`
- `workflow-templates/sync-public-excludelist.yml`

For the full GitExporter + sync-public pipeline, see [`how-to-setup-GITEXPORTER.md`](how-to-setup-GITEXPORTER.md).

---

*Maintained by [drasticstatic](https://github.com/drasticstatic)*
