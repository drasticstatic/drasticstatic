# How to Establish a GitHub Profile README

A GitHub profile README lets you turn your `github.com/yourusername` page into a living, branded landing page — visible to every recruiter, collaborator, or curious stranger who visits your profile.

---

## Step 1 — Create the Special Repository

1. Go to **github.com → New repository**
2. Set the repository name to **exactly your GitHub username** (e.g. `drasticstatic/drasticstatic`)
3. Set it to **Public**
4. Check **Add a README file**
5. Click **Create repository**

> GitHub will show a banner: *"drasticstatic/drasticstatic is a special repository."* If you don't see this, go to **Settings → toggle the special repository switch** on the repo page sidebar.

---

## Step 2 — Edit Your README.md

Clone the repo locally or edit directly in the GitHub web editor. Everything in `README.md` renders as your public profile page. GitHub supports standard Markdown plus raw HTML for layout control.

---

## Step 3 — Add a Banner

Host a custom banner SVG on GitHub Pages (or any public URL) and embed it at the top:

```markdown
<div align="center">
  <img src="https://yourusername.github.io/banner.svg" width="100%" alt="your name"/>
</div>
```

**Quick banner SVG template** — save as `banner.svg` and host on your GitHub Pages site:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 160" width="900" height="160">
  <rect width="900" height="160" fill="#060816"/>
  <text x="450" y="80" text-anchor="middle" dominant-baseline="middle"
    font-family="monospace" font-weight="900" font-size="68" fill="#72e7ff">
    yourname
  </text>
  <text x="450" y="128" text-anchor="middle" dominant-baseline="middle"
    font-family="monospace" font-size="15" letter-spacing="4" fill="#9fb0c8">
    your tagline here
  </text>
</svg>
```

---

## Step 4 — Add a Typing Animation

[readme-typing-svg](https://github.com/DenverCoder1/readme-typing-svg) generates an animated SVG that cycles through lines of text:

```markdown
[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&size=18&pause=1400&color=72E7FF&center=true&vCenter=true&width=520&lines=Line+One+Here;Line+Two+Here;Line+Three+Here)](https://git.io/typing-svg)
```

- Replace spaces with `+` in each line
- Separate lines with `;`
- Customize `color`, `size`, `width` in the URL parameters
- Use `%7C` for `|` characters

---

## Step 5 — Add Shields.io Badges

[shields.io](https://shields.io) generates flat badge images on the fly from a URL pattern:

```
https://img.shields.io/badge/LABEL-COLOR?style=flat&logo=LOGO&logoColor=white
```

**Examples:**

```markdown
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2671E5?style=flat&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
```

**Logo names** come from [simpleicons.org](https://simpleicons.org) — search for a brand and use its slug as the `logo=` value.

**Spacing between badges** — badges on the same line render inline. Add `&nbsp;` between them for a small gap, or put each on its own line for a tighter pack.

### When a logo is missing or removed from simpleicons

Some logos (LinkedIn, VS Code, MetaMask, OpenAI) have been removed from simpleicons due to trademark enforcement. You can embed any SVG directly as a base64 data URI:

```
logo=data:image/svg%2Bxml;base64,YOUR_BASE64_HERE
```

To encode your own SVG:
```bash
echo -n '<svg xmlns="http://www.w3.org/2000/svg" ...your svg here...</svg>' | base64 | tr -d '\n'
```

Paste the output after `base64,` in the badge URL. Note: `+` must be encoded as `%2B` in the URL.

---

## Step 6 — Add a Visitor Counter

[hits.sh](https://hits.sh) supports custom logos and colors (unlike the more common komarev counter):

```markdown
[![Visitors](https://hits.sh/github.com/yourusername.svg?style=flat&color=72e7ff&label=Views)](https://github.com/yourusername)
```

Customize `color` (hex without `#`) and `label` text in the URL.

---

## Step 7 — Add a 3D Contribution Graph (Nightly Auto-Update)

This uses the [github-profile-3d-contrib](https://github.com/yoshi389111/github-profile-3d-contrib) GitHub Action to generate an animated SVG of your contribution graph every night.

**1. Create `.github/workflows/3d-profile.yml`:**

```yaml
name: 3D Contribution Profile

on:
  schedule:
    - cron: '0 2 * * *'   # regenerates nightly at 2 AM UTC
  workflow_dispatch:        # also triggerable manually from Actions tab

permissions:
  contents: write           # required — without this the push will 403

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: yoshi389111/github-profile-3d-contrib@0.7.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          USERNAME: ${{ github.repository_owner }}

      - name: Commit & push 3D graph
        run: |
          git config --global user.email "action@github.com"
          git config --global user.name "GitHub Action"
          git add -A .
          git commit -m "chore: update 3D contribution graph" || true
          git push
```

> **Common gotcha:** `permissions: contents: write` must be present or the action will fail with a 403 push error. The `GITHUB_TOKEN` secret is provided automatically — no setup needed.

**2. Trigger it manually the first time:**
- Go to your repo → **Actions** tab → **3D Contribution Profile** → **Run workflow**
- Wait ~60 seconds — it will commit `profile-3d-contrib/` to your repo

**3. Embed it in your README:**

```markdown
<img src="profile-3d-contrib/profile-night-rainbow.svg" width="100%" alt="3D contribution graph"/>
```

Available themes: `profile-night-rainbow.svg`, `profile-green-animate.svg`, `profile-season-animate.svg`, and more — check the generated folder after first run.

---

## Step 8 — Enable GitHub Sponsors (Optional)

**1. Create `.github/FUNDING.yml`:**

```yaml
github: [yourusername]
```

This adds a **Sponsor** button to your repo automatically.

**2. Add a sponsor badge to your README:**

```markdown
[![Sponsor](https://img.shields.io/badge/Sponsor_%E2%9D%A4-FF5BBD?style=flat&logo=githubsponsors&logoColor=FF0000)](https://github.com/sponsors/yourusername)
```

---

## Full Starter Template

```markdown
<div align="center">

<img src="https://yourusername.github.io/banner.svg" width="100%" alt="your name"/>
<br/>

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&size=18&pause=1400&color=72E7FF&center=true&vCenter=true&width=520&lines=Your+First+Line;Your+Second+Line;Your+Third+Line)](https://git.io/typing-svg)

[![Sponsor](https://img.shields.io/badge/Sponsor_%E2%9D%A4-FF5BBD?style=flat&logo=githubsponsors&logoColor=FF0000)](https://github.com/sponsors/yourusername)
&nbsp;
[![Visitors](https://hits.sh/github.com/yourusername.svg?style=flat&color=72e7ff&label=Views)](https://github.com/yourusername)
&nbsp;
[![Portfolio](https://img.shields.io/badge/Portfolio-155724?style=flat&logo=github&logoColor=white)](https://yourusername.github.io)

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://linkedin.com/in/yourprofile)
&nbsp;
[![X](https://img.shields.io/badge/X-000000?style=flat&logo=x&logoColor=white)](https://x.com/yourhandle)

</div>

---

Brief bio or mission statement here.

- 🔧 **Project One** — what it is
- 📊 **Project Two** — what it is

---

<div align="center">
<img src="profile-3d-contrib/profile-night-rainbow.svg" width="100%" alt="3D contribution graph"/>
</div>

---

**Languages**

![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
```

---

## Tips & Gotchas

| Issue | Fix |
|-------|-----|
| Profile README not showing | Go to repo Settings and enable the "special repository" toggle |
| 3D workflow push fails (403) | Add `permissions: contents: write` to the workflow YAML |
| Logo not rendering on badge | Check [simpleicons.org](https://simpleicons.org) — the slug may have changed or been removed; use base64 SVG instead |
| Visitor counter shows "inaccessible" | Some counter services block GitHub's image proxy — switch to hits.sh |
| Spaces in badge labels | Replace with `_` (renders as a space) or `+` in the URL |
| `\|` pipe in typing SVG lines | Encode as `%7C` in the URL |

---

## Resources

- [shields.io](https://shields.io) — badge generator
- [simpleicons.org](https://simpleicons.org) — brand logo slugs
- [readme-typing-svg](https://github.com/DenverCoder1/readme-typing-svg) — animated typing lines
- [github-profile-3d-contrib](https://github.com/yoshi389111/github-profile-3d-contrib) — 3D contribution graph action
- [hits.sh](https://hits.sh) — visitor counter with logo support
- [badges.pages.dev](https://badges.pages.dev) — badge preview and discovery tool

---

*Built and maintained by [drasticstatic](https://github.com/drasticstatic) — feel free to fork, adapt, and share.*
