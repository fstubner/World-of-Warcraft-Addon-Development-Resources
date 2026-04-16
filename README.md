# WoW Addon Development Resources

A working reference for building WoW addons targeting **Midnight 12.0** retail.
Built from hard-won lessons, source-code reading, and verified API spelunking.

Sections are written for quick skim-and-return reading — not tutorials.  If
you're looking for "how do I start an addon," most tutorials on the internet
are fine.  This repo is for "I need to know the exact current API" and
"how do addons like Narcissus get that polished feel."

## Contents

| Path | Purpose |
|---|---|
| [`docs/api-reference.md`](docs/api-reference.md) | Canonical sources for the Midnight 12.0 API.  Where to look, how docs are structured, what's in the generated docs vs what isn't. |
| [`docs/ui-architecture.md`](docs/ui-architecture.md) | How WoW's UI system actually works: frames, layers, templates, SetPoint semantics, the scroll system, fonts, textures. |
| [`docs/custom-ui-patterns.md`](docs/custom-ui-patterns.md) | Patterns for building beautiful custom UIs.  Extracted from reading Narcissus, BagBrother, ElvUI, and others.  Easing, fade managers, nine-slice, smooth scroll, custom chrome. |
| [`docs/addon-starter-checklist.md`](docs/addon-starter-checklist.md) | Minimum viable addon: TOC, file layout, SavedVariables, event registration, slash commands.  Each step with the WoW idioms behind it. |
| [`docs/common-gotchas.md`](docs/common-gotchas.md) | The debugging quicksand pits.  Lua 5.1 limitations, TOC encoding, `\xHH` escapes, SetBackdrop + BackdropTemplate, FontString-without-font, deprecated APIs. |
| [`docs/testing.md`](docs/testing.md) | How to actually test an addon.  `/reload` cycle, BugSack/BugGrabber, `/run`, `/dump`, `/fstack`, `/eventtrace`, luacheck config for the WoW API. |
| [`scripts/wow-api`](scripts/wow-api) | Command-line tool that verifies API calls against Blizzard's live generated docs.  Download-once, grep-forever. |
| `api-cache/` | Local snapshot of Blizzard's `Blizzard_APIDocumentationGenerated` for the live branch.  **Not committed** (~30 MB of regenerable docs).  First invocation of `scripts/wow-api` downloads it automatically (~15s on a decent connection). |

## Quick start

```bash
# Verify a function exists in Midnight 12.0
scripts/wow-api GetQuestObjectives

# Browse an entire namespace
scripts/wow-api -l C_QuestLog

# Refresh the snapshot after a Blizzard patch
scripts/wow-api -u
```

## The authoritative sources

When in doubt, trust these in this order:

1. **[Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source)** live branch — Blizzard's own FrameXML and auto-generated API docs.  The closest thing to "ground truth."
2. **[warcraft.wiki.gg](https://warcraft.wiki.gg)** — community-maintained wiki.  Usually correct on current patches, sometimes stale.  Good for legacy globals not in the generated docs.
3. **[townlong-yak.com](https://www.townlong-yak.com/framexml/live/)** — browsable FrameXML with cross-references.
4. **Training data in LLMs** — wrong more often than you expect.  Always verify.

When the wiki and the generated docs disagree, the generated docs win: they're
produced directly from the game client at each Blizzard build.

## Targeted game version

Everything here is written for **Midnight 12.0** retail (interface `120000`).
Classic, Classic Era, Cataclysm Classic, and earlier expansions often have
different API namespaces.  This repo does not cover them.
