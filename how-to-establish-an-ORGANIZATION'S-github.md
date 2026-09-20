# GitHub organization setup — how the special repos actually work

Written after actually building this out for `psychedelicsinrecovery` and `theholyearthfoundation`
(2026-09-20) — everything below was verified live, not just researched. Distinct from personal-account
GitHub conventions (this repo, `drasticstatic/drasticstatic`, is itself the personal-profile special
repo — orgs work differently, covered here).

## The three special repos per organization

| Repo (exact name) | Visibility | Purpose |
|---|---|---|
| `.github` | **Public** | Org profile page (`profile/README.md`) + org-wide default community health files (`SECURITY.md`, `CONTRIBUTING.md` at repo root) |
| `.github-private` | Private | Same profile-README mechanism, but only rendered for logged-in org members |
| `<org-login>.github.io` | Public | The org's root GitHub Pages site — must match the org's login exactly, case-sensitive |

## `.github` — org profile + community health defaults

This one repo does two unrelated jobs at once — easy to conflate, worth keeping separate in your head:

1. **Org overview page.** GitHub renders `profile/README.md` (not the repo's own root `README.md`)
   on the organization's public overview page. Getting the path wrong (root instead of `profile/`) is
   the single most common mistake — a root `README.md` here just describes the `.github` repo itself
   to someone who opens it directly, it does **not** show up on the org's public page.
2. **Org-wide default community health files.** `SECURITY.md` and `CONTRIBUTING.md` placed at this
   repo's **root** (not inside `profile/`) become the fallback for any repo in the org that doesn't
   have its own copy of that file. A repo-level `SECURITY.md` always wins over this default if one
   exists — this is a fallback, not an override.

**The org must be public** (or the repo visible to whoever you want seeing the profile) for the
profile page to render for logged-out visitors. A private org's `.github` profile only shows to
members, same as `.github-private` always does.

## `.github-private` — members-only profile

Same `profile/README.md` mechanism as above, but this repo is private, so only logged-in members of
the organization see it rendered on the org page. Useful for internal coordination notes, status,
or anything not ready for public view yet — this is where "where do things actually stand" content
belongs while a young org is still finding its shape.

## LICENSE — no org-wide fallback, by design

Unlike `SECURITY.md`/`CONTRIBUTING.md`, there is **no equivalent org-wide fallback for licensing**.
Every repository that needs one gets its own `LICENSE` (or `LICENSE.md`) file at its own root.
GitHub detects it automatically and shows a license badge in that repo's sidebar — this is
per-repository by design, since an org can mix public open-source work with private proprietary
code, and licensing has to bind to the specific codebase it covers, not the organization as a whole.

## `<org>.github.io` — the org's root page and routing anchor

Creating a repo named **exactly** the org's login (case-sensitive) plus `.github.io` gives the org a
real homepage at the clean root domain (`https://<org>.github.io`) instead of a generic 404. Verified
live: **GitHub Pages auto-enables itself** for a repo with this exact name — no manual "enable Pages"
step needed (confirmed by a 409 "already enabled" response when explicitly trying to turn it on via
the API right after repo creation).

What it actually controls:

- **The root landing page** for the whole org's GitHub Pages presence.
- **Routing for every other project's Pages site.** Any other repo in the org with Pages enabled
  deploys under `<org>.github.io/<repo-name>/` automatically — a subpath, not its own domain — unless
  that specific repo has its own custom domain configured.
- **Custom domain inheritance.** If a custom domain is applied to this specific repo (via a `CNAME`
  file at its root, or the Pages settings UI), that becomes the org's default base domain. Any other
  project repo without its own separate custom domain inherits routing under that domain rather than
  falling back to `<org>.github.io/<repo-name>/`.

Minimum viable content: `index.html` and `404.html` at the repo root. No build step required unless
you want one (Jekyll, a static site generator, etc.) — plain HTML in the root is enough to go live.

## What actually got built (as a working example)

Both orgs now have all three repos live:

```
psychedelicsinrecovery/.github                                  (public, profile + community health)
psychedelicsinrecovery/.github-private                          (private, members-only profile)
psychedelicsinrecovery/psychedelicsinrecovery.github.io          (public, org root page)

theholyearthfoundation/.github                                  (public, profile + community health)
theholyearthfoundation/.github-private                          (private, members-only profile)
theholyearthfoundation/theholyearthfoundation.github.io          (public, org root page)
```

All content in these six repos is a **first draft** — written from real context where it existed
(deep for PIR, given the amount of context this session already had; thinner for Holy Earth
Foundation, sourced from `~/the-holy-earth-foundation/README.md` and its two project READMEs, and
explicitly flagged in the profile itself as needing Kenney's review before being treated as final).
Neither profile should be read as institutionally final — both say so in their own text.

## Practical sequence, if setting this up for a future org

```bash
gh repo create <org>/.github --public --source=. --push          # after building profile/README.md, SECURITY.md, CONTRIBUTING.md locally
gh repo create <org>/.github-private --private --source=. --push  # after building README.md locally
gh repo create <org>/<org>.github.io --public --source=. --push   # after building index.html, 404.html locally
```

Pages enablement is automatic for the `.github.io` repo — no extra API call needed, verified above.
