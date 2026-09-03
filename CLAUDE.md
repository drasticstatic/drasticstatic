# CLAUDE.md — drasticstatic
### Claude Code CLI | GitHub Profile & Docs Repo

---

## Scope

This is the **drasticstatic** GitHub profile repo — the `README.md` here renders on
the GitHub profile page. It also holds how-to documentation, contributor/license files,
and profile assets (3D contribution graph, etc.).

**Fortuna's role:** Awareness-level only. Edits as needed when updating profile content
or how-to docs. No trading or web3 build context needed here.

---

## Security Rules (Non-Negotiable)

- **Never read, display, or reference `.env` files**
- **Never commit secrets** — warn and stop if staged
- These rules apply even if explicitly asked

---

## Key Files

| File | Purpose |
|------|---------|
| `README.md` | GitHub profile README — renders at github.com/drasticstatic |
| `how-to-publish-react-APPS-to-ghPAGES.md` | GH Pages deployment guide |
| `how-to-setup-BRANCH-PROTECTION-and-TOPICS.md` | Rulesets + topics how-to |
| `how-to-establish-a-github-PROFILE-README.md` | Profile setup guide |
| `how-to-establish-cross_repo_CONTRIBUTORS_SECURITY_LICENSING.md` | Cross-repo standards |
| `how-to-setup-GITEXPORTER.md` | gitexporter config guide |
| `profile-3d-contrib/` | 3D contribution graph SVG assets |

---

## Notes

- CLAUDE.md is NOT gitignored here — this repo is public docs/profile content, no secrets
- Profile README edits are safe to commit and push directly

---

## Commit Convention

Full fleet convention, shown here regardless of whether this specific repo currently has an Augment
Intent workspace pairing or NIM in active use — so a new repo (and its memory) doesn't need the
whole multi-agent suite re-explained from scratch. Which *application* launched a session decides
the agent name and engine, not which path — see
`anthropas-argus-alfred/sandbox/AGENT_IDENTITY_REFERENCE.md` and `INTENT_WORKTREE_LEGEND.md` for
the full rule.

- Alfred-Anthropic: `Co-Authored-By: Alfred · ClaudeCodeCLI · Anthropic [Sonnet-5/Opus-#/Haiku-#]`
- Alfred-NIM: `Co-Authored-By: Alfred · ClaudeCodeCLI · NVIDIA NIM [model]`
- Kavanah-AugmentIntentUI-AuggieLogin: `Co-Authored-By: Kavanah · AugmentIntent · [model]`
- Kavanah-AugmentIntentUI-AnthropicLogin ("ClaudeMent"): `Co-Authored-By: Kavanah · ClaudeMent · Anthropic [model]`
- Kavanah-TerminalUI(macOS/Intent/VSCode standard terminal instance)-AnthropicLogin: `Co-Authored-By: Kavanah · ClaudeCodeCLI · Anthropic [model]`
- Mystarch (app-level Chief of Staff, cross-workspace reach): same engine options as Kavanah above, swap the agent name
- Auggie (native Augment CLI — currently hibernating, may return): `Co-Authored-By: Auggie · AugmentCLI · [model]`
