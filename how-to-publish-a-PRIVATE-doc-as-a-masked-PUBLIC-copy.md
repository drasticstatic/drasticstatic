# 🙈 How to publish a private doc as a masked public copy

> **Pattern:** keep one **private source of truth** (e.g. an org's `.github-private/CONTACTS.md`)
> and generate a **public copy** with private sections stripped and personal emails masked, so you
> never maintain two diverging files. First built for PIR® (`psychedelicsinrecovery`), October 2026.

## 🧩 The pieces
| Piece | Where | What it does |
|---|---|---|
| 🔒 Private source | e.g. `.github-private/CONTACTS.md` | Everything, including the members-only parts |
| 🪬 Private markers | inside the source | `<!-- private:start -->` … `<!-- private:end -->` wrap anything that must never go public. Start each block with a `> 🔒🪬 PRIVATE — …` blockquote so humans reading the private file can see the boundary too. |
| 🧹 Generator | `scripts/publish-public-contacts.py` | Strips private blocks, masks any email **not** on your org's domain as `***@domain`, adds an "auto-generated" banner, and **refuses to publish** if markers are unbalanced |
| 🤖 Workflow | `.github/workflows/publish-public-contacts.yml` | On push, runs the generator and commits the result to the public repo |
| 🔑 Secret | `PUBLIC_SYNC_TOKEN` | Fine-grained PAT, **Contents read/write on the public repo only**. Without it the workflow skips, and you run the script locally instead. |

📦 **Reusable template:** [`my-template/workflow-templates/publish-masked-public-copy.py`](https://github.com/drasticstatic/my-template/tree/main/workflow-templates)
and `.yml`. Copy both, set `ORG_DOMAIN`, and adjust the paths.

## 🧭 Steps
1. Write the private doc. Wrap private parts in markers; leave public parts bare.
2. Show missing public values as `🙈`, so readers know something exists but is members-only.
3. Run `python3 scripts/publish-public-contacts.py /tmp/preview.md` and **grep the preview** for
   anything that must not leak (full names, personal emails, vault and DNS details).
4. Commit the public copy (or let the workflow do it once the secret exists).
5. Link the public copy from the org profile README, and link the private copy from the public one.
   It will 404 for non-members, so say so, and offer an access-request link.

## ⚠️ Gotchas learned the hard way
- **Don't write the literal marker strings inside a private block's explanatory text.** The
  generator sees them as markers. Describe them in words instead.
- Mask by **domain allowlist**, not by guessing which emails are personal.
- Public tables should never name who can recover credentials. That's a gift to social engineers.

## ❓ Why not my-template's `sync-public-allowlist.yml`?
That workflow mirrors whole **files** from a private repo to a public one. This pattern publishes
**part of one file**. Use the allowlist sync for whole-repo public lanes (changelogs, portals), and
this for mixed public/private documents.
