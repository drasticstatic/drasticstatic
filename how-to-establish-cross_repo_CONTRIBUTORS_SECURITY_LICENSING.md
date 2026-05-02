# 🛡️ How to Establish Cross-Repo Security, Licensing & Contributor Standards

> A practical guide for `drasticstatic` — covering SECURITY.md templates, the global `.github` repo, Dependabot, Private Vulnerability Reporting, and contributor recognition. Written for a polyglot stack: TypeScript/Solidity, Python, GitHub Actions.

---

## 📋 Quick-Reference Checklist

For each new repo, tick these off:

- [ ] `LICENSE` in repo root (MIT for most; check per-project)
- [ ] `CONTRIBUTING.md` in repo root
- [ ] `SECURITY.md` in repo root — OR rely on global `.github` fallback (see below)
- [ ] Dependabot enabled (alerts + security updates)
- [ ] `dependabot.yml` for version updates if the repo has package files
- [ ] Private Vulnerability Reporting enabled (one-time global setting)
- [ ] CONTRIBUTORS.md entry (in profile repo `drasticstatic/drasticstatic`)

---

## 🔒 SECURITY.md — Templates

### Standard Template (most repos)

```markdown
# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| latest  | ✅        |
| older   | ❌        |

## Reporting a Vulnerability

Please **do not** open a public GitHub Issue for security vulnerabilities.

Instead, use one of the following:
- **GitHub Private Vulnerability Reporting** (preferred) — use the "Report a vulnerability" button on the Security tab
- **Email** — [your email here]

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Any suggested fix (optional)

You will receive a response within **48 hours**. If confirmed, a CVE will be requested and you will be credited in the security advisory.

## Safe Harbor

We consider security research conducted in good faith to be authorized. We will not pursue legal action against researchers who:
- Report vulnerabilities through the above channels
- Avoid accessing or modifying user data
- Do not disrupt service availability
- Keep findings confidential until we've had time to address them
```

### Robust Template (with legal disclaimer — for public-facing or sensitive repos)

```markdown
# Security Policy

## About This Project

[Brief description — what the project does and what data/systems it touches.]

## Reporting a Vulnerability

**Do not report security vulnerabilities through public GitHub Issues, discussions, or pull requests.**

Use one of these private channels:
1. **GitHub Private Vulnerability Reporting** — preferred. Click "Report a vulnerability" on the Security tab.
2. **Email** — [your email]

Please include:
- A clear description of the vulnerability
- Steps to reproduce (proof-of-concept if possible)
- Affected versions or components
- Potential impact assessment
- Your suggested fix or mitigation (optional but appreciated)

**Response commitment:** Acknowledgement within 48 hours. Resolution timeline communicated within 7 days.

## Responsible Disclosure

We follow a **coordinated disclosure** model:
- We ask for **90 days** before public disclosure to allow for a fix
- We will credit you by name (or pseudonym if preferred) in the security advisory
- We will request a CVE on your behalf for qualifying vulnerabilities

## Safe Harbor

This project considers good-faith security research to be **authorized and protected**. Researchers acting in good faith who:
- Report through official channels above
- Avoid accessing, modifying, or exfiltrating user data
- Do not degrade service availability (no DoS testing)
- Keep findings confidential during the disclosure window

...will not face legal action from this project's maintainers. We will not refer researchers to law enforcement for good-faith research.

> **Note:** This safe harbor applies only to the maintainers of this project and does not bind third-party hosting providers or other parties.

## Recognition

Confirmed vulnerability reporters will be:
- Named (or credited as preferred) in the GitHub Security Advisory
- Listed in [CONTRIBUTORS.md](https://github.com/drasticstatic/drasticstatic/blob/main/CONTRIBUTORS.md)
- Offered a LinkedIn recommendation for significant findings

We do not currently offer monetary rewards.
```

---

## 🌐 The Global `.github` Repo — One Repo, All Fallbacks

GitHub has a special repo: **`[username]/.github`** (i.e. `drasticstatic/.github`).

Any file placed here acts as a **fallback** for all repos on the account that don't have their own version. This means:

| File | Where GitHub looks first | Then falls back to |
|------|--------------------------|-------------------|
| `SECURITY.md` | Repo root | `drasticstatic/.github/SECURITY.md` |
| `CONTRIBUTING.md` | Repo root | `drasticstatic/.github/CONTRIBUTING.md` |
| `CODE_OF_CONDUCT.md` | Repo root | `drasticstatic/.github/CODE_OF_CONDUCT.md` |
| Issue templates | `.github/ISSUE_TEMPLATE/` | `drasticstatic/.github/ISSUE_TEMPLATE/` |
| PR templates | `.github/PULL_REQUEST_TEMPLATE.md` | `drasticstatic/.github/PULL_REQUEST_TEMPLATE.md` |

### How to set it up

1. Create repo `drasticstatic/.github` on GitHub (public)
2. Clone it locally: `git clone https://github.com/drasticstatic/.github.git`
3. Add your default files:
   ```
   .github/
   ├── SECURITY.md
   ├── CONTRIBUTING.md
   ├── CODE_OF_CONDUCT.md
   └── ISSUE_TEMPLATE/
       └── bug_report.md
   ```
4. Commit and push — all repos without their own versions now inherit these

> **Tip for this stack:** `dev-recruitment-safeguards` has its own `SECURITY.md`/`CONTRIBUTING.md` (repo-specific content). The `.github` repo is the right fallback for repos that don't need custom versions.

---

## 🤖 Dependabot — Automated Dependency Security

### Enable globally (recommended)

Go to **GitHub Settings → Code security → Dependabot** and enable:
- ✅ Dependabot alerts
- ✅ Dependabot security updates
- ✅ Grouped security updates

This covers all repos without needing per-repo configuration.

### `dependabot.yml` — Version Update Automation

For repos with package files, add `.github/dependabot.yml` to get automated version bump PRs:

```yaml
# .github/dependabot.yml
# Covers Christopher's polyglot stack:
# - npm (TypeScript, Hardhat/Solidity via npm)
# - pip (Python — trading-assistant scripts, data tools)
# - github-actions (workflow dependencies)
# Not applicable: Pine Script (no package manager), Solidity (managed via npm/Hardhat)

version: 2
updates:

  # npm — TypeScript, React, Hardhat, web3 packages
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    groups:
      security-updates:
        applies-to: security-updates
        patterns: ["*"]
      minor-and-patch:
        applies-to: version-updates
        update-types: ["minor", "patch"]
    open-pull-requests-limit: 5

  # pip — Python scripts (trading-assistant, data analysis)
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5

  # GitHub Actions — keep workflow actions pinned to latest
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
```

**Notes for this stack:**
- **Solidity:** Dependencies are managed via npm (Hardhat, OpenZeppelin) — the `npm` block covers it
- **Pine Script:** No package manager exists — not applicable, not a Dependabot ecosystem
- **Cargo (Rust):** Not currently in stack — add `package-ecosystem: "cargo"` if Rust is introduced
- **Multi-directory repos:** If `package.json` is in a subdirectory, change `directory: "/"` to `directory: "/contracts"` etc.

---

## 🔕 Private Vulnerability Reporting — Global Enable

GitHub allows enabling Private Vulnerability Reporting (PVR) for **all public repos** at once.

### Enable globally

1. Go to **GitHub Settings → Code security**
2. Find **"Private vulnerability reporting"**
3. Toggle: **Enable for all public repositories**

This adds a **"Report a vulnerability"** button to the Security tab of every public repo. Reports go directly to you as a private draft advisory — no public exposure until you choose to publish.

> For personal accounts, this is the recommended approach — no need to enable per-repo.

---

## 🏆 Hall of Fame / CONTRIBUTORS.md

The **`drasticstatic/drasticstatic`** repo (your GitHub profile README repo) is the natural home for a cross-repo `CONTRIBUTORS.md`. This ensures it's discoverable at `github.com/drasticstatic` and linked from security policies.

### Template

```markdown
# 🏆 Contributors & Security Researchers

Thank you to everyone who has contributed to or secured `drasticstatic` projects.

## Security Researchers

| Name / Handle | Finding | Repo | Date |
|--------------|---------|------|------|
| *(open for first entry)* | — | — | — |

## Code Contributors

See individual repo contributor graphs for code contributions.

## How to Get Listed

- Report a confirmed security vulnerability (see [SECURITY.md](https://github.com/drasticstatic/.github/blob/main/SECURITY.md))
- Make a significant code or documentation contribution to any public repo

*Recognition is at maintainer discretion. Pseudonyms honored on request.*
```

---

## 💼 Bug Bounty Without Budget — Recognition-Only Program

A formal bug bounty doesn't require money. A **recognition-only program** still incentivizes responsible disclosure by offering:

| Reward | What it means |
|--------|--------------|
| **CONTRIBUTORS.md listing** | Permanent public credit in profile repo |
| **LinkedIn recommendation** | For significant/critical findings — verifiable professional recognition |
| **CVE credit** | Your name in the NVD/CVE database — career-relevant for security researchers |
| **GitHub Security Advisory credit** | Attributed in the published advisory |
| **Safe harbor** | You won't be sued for good-faith research (the most important "reward") |

### Key language to include in SECURITY.md

```
We do not offer monetary rewards. For confirmed vulnerabilities, we offer:
- Credit in the published GitHub Security Advisory
- Listing in CONTRIBUTORS.md
- A LinkedIn recommendation for high-severity findings (at maintainer discretion)
- CVE assignment for qualifying vulnerabilities
```

---

## 🗂️ Per-Repo vs. Global — Decision Matrix

| Scenario | Recommendation |
|----------|---------------|
| New repo, no special security needs | Rely on `.github` repo fallbacks |
| Repo with user data or smart contract funds | Add repo-specific `SECURITY.md` with threat model |
| Educational/informational repo (like `dev-recruitment-safeguards`) | Custom `SECURITY.md` explaining what files are safe |
| Profile/profile-README repo | Add `CONTRIBUTORS.md` here |
| All repos | Enable Dependabot globally; add `dependabot.yml` to repos with package files |

---

## 🔗 Related Files

| File | Location | Purpose |
|------|----------|---------|
| Pointer (AGENT-SYNC) | `trading-assistant/AGENT-SYNC/how-to-establish-cross_repo_CONTRIBUTORS_SECURITY_LICENSING.md` | Cross-repo reference |
| `dev-recruitment-safeguards` example | `/Users/christopherwilson/dappu/dev-recruitment-safeguards/SECURITY.md` | Repo-specific security policy |
| Global profile repo | `/Users/christopherwilson/drasticstatic/` | Home for CONTRIBUTORS.md |
| GitHub Pages / favicon host | `/Users/christopherwilson/drasticstatic.github.io/` | Global assets (favicon, banner SVG) |

---

*Created by Fortuna — trading-assistant session May 2, 2026*
