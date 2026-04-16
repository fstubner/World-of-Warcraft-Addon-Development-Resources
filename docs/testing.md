# Testing a WoW Addon

WoW addon testing is genuinely harder than most ecosystems because the
entire "API" is a runtime-only capability that can't be replicated outside
the game client.  This document covers the practical workflow.

## The core iteration loop

**`/reload`** is the fundamental dev cycle.  Edit your .lua → in-game,
type `/reload` (or `/rl`).  WoW re-reads all your addon's files and
rebuilds their Lua state in ~2–5 seconds.  SavedVariables persist across
the reload.

```
edit → /reload → observe → edit → /reload → ...
```

**Full client restart** is required when:
- TOC file changes (adds/removes/reorders files)
- You suspect state corruption that `/reload` can't clear
- Testing the full ADDON_LOADED → PLAYER_LOGIN sequence

Full restart takes ~30s; use sparingly.

## Development sync (directory junction)

On Windows, symlink your development directory into WoW's AddOns folder
once, so every edit is instantly visible without a copy step:

```cmd
rem Run as administrator (needed for Program Files paths):
mklink /J "C:\Games\World of Warcraft\_retail_\Interface\AddOns\MyAddon" "H:\dev\MyAddon"
```

Or in PowerShell:
```powershell
New-Item -ItemType Junction -Path "C:\Games\World of Warcraft\_retail_\Interface\AddOns\MyAddon" -Target "H:\dev\MyAddon"
```

WoW sees the junction as a regular directory.  On macOS, use `ln -s` for
symlinks (WoW respects them the same way).

## Error visibility (critical first setup)

Blizzard's default error handling SILENTLY SWALLOWS Lua errors unless you
opt in.  Every serious addon dev installs:

- **[BugSack](https://www.curseforge.com/wow/addons/bugsack)** + **[BugGrabber](https://www.curseforge.com/wow/addons/bug-grabber)**
  — captures every Lua error with full stack trace, aggregates them, shows
  a minimap icon when something errors.  Without this, a typo in an
  `OnClick` handler just means "the button doesn't work" with no
  explanation.

- Alternative: `/console scriptErrors 1` enables Blizzard's built-in error
  popup (ugly but free).

**Install BugSack BEFORE any in-game testing.**  The 30-second install saves
hours of "why isn't this working?" guesswork.

## Runtime inspection

### `/run <lua>` — inline execution

Runs arbitrary Lua.  Great for poking live state:

```
/run print(#MyAddonDB.entries)
/run for i,e in ipairs(MyAddonDB.entries) do print(e.type, e.title) end
/run MyAddon.UI:ToggleBrowser()
```

### `/dump <expr>` — pretty-print

```
/dump MyAddonDB
/dump MyAddonDB.entries[1]
/dump C_QuestLog.GetQuestObjectives(12345)
```

Great for discovering actual API return shapes when the docs are ambiguous.

### `/fstack` — frame stack inspector

Mouse over any UI element, hold `Ctrl+Shift`, and WoW shows the full frame
hierarchy under the cursor.  Invaluable for "why isn't my click handler
firing?" debugging (usually because another frame is eating the clicks).

### `/eventtrace` — live event log

Opens a window streaming every event the game fires.  Click "Filter" to
whitelist specific events.  Essential when you're wondering "does
GOSSIP_OPTION_SELECTED actually fire on that NPC?" — just `/eventtrace`
and click through the gossip.

### DevTool addon

[DevTool](https://www.curseforge.com/wow/addons/devtool) opens a treeview
browser on any table.  `/dev MyAddonDB` gives you a clickable GUI explorer
of the entire save state — much faster than chaining `/dump` calls.

## Static analysis (out-of-game)

You can catch some bugs without running WoW:

### `luac -p` (syntax check)

Lua's bytecode compiler with parse-only flag.  Catches syntax errors
immediately:
```bash
luac -p modules/UI.lua   # exits nonzero on syntax error
```

Wire into a pre-commit hook.  Catches unclosed strings, missing `end`, etc.

### luacheck (full linting)

Install: `luarocks install luacheck`.  Usage: `luacheck --config .luacheckrc .`

Catches unused variables, shadowed locals, undeclared globals.

Sample `.luacheckrc` for a WoW addon:

```lua
-- .luacheckrc
std = "lua51"

-- Your addon globals
globals = {
    "MyAddon",
    "MyAddonDB",
    "MyAddonCharDB",
    "SLASH_MYADDON1", "SLASH_MYADDON2",
    "SlashCmdList",
}

-- WoW engine / Blizzard API globals (read-only from addon perspective)
read_globals = {
    -- Lua stdlib
    "string", "table", "math", "bit", "date", "time",

    -- WoW core
    "CreateFrame", "UIParent", "UISpecialFrames",
    "GameTooltip", "WorldMapFrame",
    "DEFAULT_CHAT_FRAME", "BreakUpLargeNumbers",
    "UIFrameFadeIn", "UIFrameFadeOut",
    "CreateColor", "CreateFont",

    -- WoW namespaces
    "C_QuestLog", "C_QuestInfoSystem", "C_GossipInfo",
    "C_Map", "C_Timer", "C_CampaignInfo",
    "UiMapPoint",
    "Enum",

    -- Unit / player
    "UnitName", "UnitGUID", "UnitClassification", "UnitReaction",
    "GetRealmName", "GetRealZoneText", "GetSubZoneText",
    "GetInstanceInfo", "IsEncounterInProgress",
    "GetAchievementInfo",
    "GetQuestLogRewardMoney", "GetQuestLogRewardXP",
    "GetNumQuestLogRewards", "GetQuestLogRewardInfo",

    -- Combat
    "CombatLogGetCurrentEventInfo", "COMBATLOG_OBJECT_TYPE_NPC",

    -- Libraries
    "LibStub",

    -- Globals
    "_", "_G",
}

self = false            -- don't warn about implicit self in methods
max_line_length = false -- WoW Lua style is wide

exclude_files = {
    "libs/",
    "node_modules/",
}
```

### lua-language-server

VS Code extension with Lua IntelliSense.  Point it at a WoW API types
definition (several community-maintained repos exist, e.g.
[wow-ui-source](https://github.com/Gethe/wow-ui-source)) and get
autocomplete, hover docs, and catch-at-edit-time warnings like
"C_QuestLog.GetTitleFor doesn't exist, did you mean GetTitleForQuestID?"

Closest thing to IDE support for WoW addon dev.

## Unit testing (the hard part)

If you want automated `assert`-style tests:

### busted + mocked WoW globals

Extract pure-logic functions from your frame-handling code.  Mock the
WoW globals in your test setup.  Run with `busted`.

Testable surfaces in a typical addon:
- Pure string formatters (`_fmtMoney`, `_renderBody`, etc.)
- Filter predicates (`IsStoryQuest`, tag lookups)
- Math helpers (easing functions, elapsed-time formatters)

Anything touching `CreateFrame` / `SetScript` / `SetPoint` is effectively
untestable without a full WoW API mock.  Projects like
[wow-mock](https://github.com/bkader/wow-mock) exist but are incomplete.

Example test:
```lua
-- spec/format_spec.lua
_G.date = function(fmt, ts) return "Mock date" end
_G.time = function() return 1000000 end

describe("_fmtMoney", function()
    it("formats gold/silver/copper", function()
        assert.equal("75g 30s", _fmtMoney(753000))
    end)
    it("returns nil for zero", function()
        assert.is_nil(_fmtMoney(0))
    end)
end)
```

## Manual test matrix

A 10-minute ritual after any nontrivial change:

- [ ] `/reload` — no errors in BugSack
- [ ] `/<addon>` — main UI opens
- [ ] Click through each major feature / filter / tab
- [ ] Type in any search box — debounce works, results update
- [ ] Scroll long lists — no duplicate or missing rows
- [ ] Open and close repeatedly — no frame accumulation
- [ ] `/reload` again — state persists correctly
- [ ] Quit and relaunch WoW — SavedVariables restored cleanly
- [ ] Target an NPC / accept a quest / kill a mob — events fire, entries added

## API verification before writing code

This repo's `scripts/wow-api` tool should be your first check:

```bash
scripts/wow-api GetQuestObjectives
```

If it returns "not found," don't write the code yet.  Either the API
doesn't exist, was renamed, or is a legacy global (check
`https://warcraft.wiki.gg/wiki/API_<FunctionName>` in that case).

Running `scripts/wow-api` with no arguments verifies every `C_*` call in
your current repo against Blizzard's live docs.  Good pre-commit check.

## CI

GitHub Actions example (`.github/workflows/lint.yml`):

```yaml
name: Lint
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm install
      - run: scripts/wow-api          # bulk-verify all C_* calls
```

For CurseForge automatic packaging, add
[BigWigsMods/packager](https://github.com/BigWigsMods/packager):

```yaml
name: Release
on:
  push:
    tags: [ 'v*' ]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: BigWigsMods/packager@v2
        with:
          args: -g retail
```

## Further reading

- `docs/common-gotchas.md` — bugs to watch for during testing
- `docs/api-reference.md` — where ground-truth lives
