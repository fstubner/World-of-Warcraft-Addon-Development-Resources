# WoW API Reference Guide (Midnight 12.0)

This is a pointer document, not a function-by-function reference.  Blizzard
publishes 575 auto-generated doc files; nobody writes them by hand and you
shouldn't try.  This page tells you **where the ground truth lives** and
**how to read it**.

## The authoritative source

Blizzard's game client ships with a generator that produces Lua files
documenting every `C_*` namespaced function.  Since 2019, these files have
been mirrored to GitHub by the community (Gethe) and updated with every
PTR and live build:

**[Gethe/wow-ui-source/tree/live/Interface/AddOns/Blizzard_APIDocumentationGenerated](https://github.com/Gethe/wow-ui-source/tree/live/Interface/AddOns/Blizzard_APIDocumentationGenerated)**

As of writing there are **575 documentation files** there, one per
subsystem: `QuestLogDocumentation.lua`, `MapDocumentation.lua`,
`GossipInfoDocumentation.lua`, and so on.

### Structure of a generated doc file

Each file follows the same Lua-table structure:

```lua
local QuestLog =
{
    Name      = "QuestLog",
    Type      = "System",
    Namespace = "C_QuestLog",
    Environment = "All",

    Functions = {
        {
            Name = "GetQuestObjectives",
            Type = "Function",
            Arguments = {
                { Name = "questID", Type = "number", Nilable = false },
            },
            Returns = {
                { Name = "objectives", Type = "table", InnerType = "QuestObjectiveInfo", Nilable = false },
            },
        },
        ...
    },

    Events = { ... },

    Tables = {
        {
            Name = "QuestObjectiveInfo",
            Type = "Structure",
            Fields = { ... },
        },
        ...
    },
};
```

Key fields:

| Field | Meaning |
|---|---|
| `Namespace` | The Lua global prefix — `C_QuestLog`, `C_Map`, etc. |
| `Functions[].Name` | Callable as `C_QuestLog.<Name>(args...)` |
| `Arguments[].Nilable` | Whether the parameter can be `nil` — if `true`, it's optional |
| `Returns[].Nilable` | Whether the return can be `nil` — if `true`, you MUST nil-check |
| `Returns[].InnerType` | For `table` returns: the structure name of each element (cross-reference Tables section) |
| `Events[].Name` | Register with `frame:RegisterEvent("<Name>")` |
| `Tables[]` | Structure definitions for complex return types |

### What's NOT in the generated docs

The generated docs only cover **modern `C_*` namespaced APIs**.  Legacy
globals from pre-2018 WoW are not included.  These still work but are
documented only on the community wiki:

- Frame construction: `CreateFrame`, `CreateFont`, `UIParent`, `GameTooltip`
- Legacy player info: `UnitName`, `UnitGUID`, `UnitClassification`, `GetRealmName`
- Legacy quest rewards: `GetQuestLogRewardMoney`, `GetQuestLogRewardXP`,
  `GetNumQuestLogRewards`, `GetQuestLogRewardInfo` — all accept `questID`
  as an optional parameter in modern WoW
- Combat log: `CombatLogGetCurrentEventInfo`, `COMBATLOG_OBJECT_*` constants
- Map/world: `GetRealZoneText`, `GetSubZoneText`, `GetInstanceInfo`
- Achievements: `GetAchievementInfo`, `GetNumCompletedAchievements`
- FontObjects: `GameFontNormal`, `GameFontNormalSmall`, `GameFontNormalLarge`
- And more...

For these, use:

1. **[warcraft.wiki.gg](https://warcraft.wiki.gg)** — community-maintained.
   URL pattern: `https://warcraft.wiki.gg/wiki/API_<FunctionName>`
2. **[townlong-yak.com/framexml/live/](https://www.townlong-yak.com/framexml/live/)**
   — browsable FrameXML with cross-references; great for seeing how Blizzard's
   own UI code uses an API.

### When wiki and generated docs disagree

**Generated docs win.**  They come directly from the game binary and update
per-build.  The wiki is maintained by humans and lags behind patches.

Example from the Chronicle project: `C_QuestLog.GetQuestClassification`
appears on some old wiki pages but was actually never in that namespace.
The generated docs correctly show it as `C_QuestInfoSystem.GetQuestClassification`.
Wiki says it's deprecated; generated docs are source of truth.

## The wow-api verification script

`scripts/wow-api` in this repo is a thin wrapper over "grep the generated docs."

```bash
# Does this function exist?  If yes, where?
scripts/wow-api GetQuestObjectives

# Show the full signature:
scripts/wow-api -v GetQuestClassification

# List everything in a namespace:
scripts/wow-api -l C_QuestLog

# Check every API call in my current addon:
cd /path/to/addon
scripts/wow-api

# Refresh the cache after a patch:
scripts/wow-api -u
```

**Rule of thumb before writing any new API call:**
```bash
scripts/wow-api <FunctionName>
```
If it doesn't come back with a match, don't write the code yet.  The API
either doesn't exist, was renamed, or is a legacy global (check wiki).

## Enum tables

Many API returns use enum values.  These are defined on the `Enum` global:

```lua
Enum.QuestClassification.Campaign      -- 1
Enum.QuestClassification.Legendary     -- 2
Enum.QuestClassification.Important     -- 3
Enum.QuestClassification.Calling       -- 4
Enum.QuestClassification.Meta          -- 5
Enum.QuestClassification.Recurring     -- 6
Enum.QuestClassification.Questline     -- 7
Enum.QuestClassification.Normal        -- 8
```

These are populated at client startup and are available in `ADDON_LOADED`.
**Do NOT guard with `if Enum and Enum.QuestClassification` — it's always there
in Midnight 12.0.**  If it weren't, your addon has a much bigger problem.

Enum definitions live in `.../Blizzard_APIDocumentationGenerated/*Constants*.lua`
and `.../Blizzard_APIDocumentation/*Constants*.lua`.  The wiki also has a
searchable list: https://warcraft.wiki.gg/wiki/Enum

## Event reference

Events are registered with `frame:RegisterEvent("EVENT_NAME")` and fire on
`OnEvent`.  Each event carries payload arguments documented in the
`Events` section of the relevant doc file.

High-signal events for story/quest addons:

| Event | Payload | Notes |
|---|---|---|
| `PLAYER_LOGIN` | — | Player object is ready; character info is valid |
| `PLAYER_ENTERING_WORLD` | `isInitialLogin`, `isReloadingUi` | First valid moment for world-state APIs |
| `QUEST_ACCEPTED` | `questID` | Fires before the quest appears in the log |
| `QUEST_TURNED_IN` | `questID`, `xpReward`, `moneyReward` | Fires on NPC turn-in |
| `GOSSIP_SHOW` | — | NPC dialog frame opens; use `C_GossipInfo.GetText/Options` |
| `GOSSIP_OPTION_SELECTED` | `optionID` | Player picked a dialog option |
| `CINEMATIC_START` / `STOP` | `canBeCancelled` on start | In-engine cutscenes |
| `PLAY_MOVIE` | `movieID` | Pre-rendered FMV cinematics (can be replayed) |
| `ACHIEVEMENT_EARNED` | `achievementID` | New achievement |
| `BOSS_KILL` | `encounterID`, `encounterName` | After a recognized boss dies |
| `COMBAT_LOG_EVENT_UNFILTERED` | — | Use `CombatLogGetCurrentEventInfo()` |
| `ZONE_CHANGED_NEW_AREA` | — | Major zone transitions |
| `ADDON_LOADED` | `name` | Fires once per addon; SavedVariables are ready |

Run `/eventtrace` in-game to see which events fire in real-time.  Invaluable
for discovering "what event should I hook for X?"

## Deprecated API patterns to avoid

These come up repeatedly:

### `SelectQuestLogEntry(idx)` — REPLACED
Modern reward APIs accept `questID` directly:
```lua
-- OLD (still works but deprecated):
local idx = GetQuestLogIndexByID(questID)
SelectQuestLogEntry(idx)
local money = GetQuestLogRewardMoney()

-- NEW:
local money = GetQuestLogRewardMoney(questID)
```

### `GetQuestLogTitle(idx)` — REPLACED
```lua
-- OLD:
local title = GetQuestLogTitle(idx)

-- NEW:
local title = C_QuestLog.GetTitleForQuestID(questID)
```

### `UIDropDownMenu_*` — DEPRECATED
The old dropdown menu system is soft-deprecated.  Modern replacement is
`MenuUtil`.  See `Interface/SharedXML/MenuUtil.lua` in wow-ui-source.

### `BackdropTemplate` — STILL WORKS but discouraged
Since 9.0.1 (Shadowlands), `SetBackdrop` only works on frames mixed with
`"BackdropTemplate"`.  Most modern addons skip this and roll their own
nine-slice backdrops instead (see `custom-ui-patterns.md`).

### `\x` hex escapes in Lua string literals — NOT SUPPORTED
WoW uses Lua 5.1, which does NOT support `"\xe2\x80\x94"` (hex escape).
Use the actual UTF-8 character in the source or decimal escapes:
```lua
-- BROKEN in Lua 5.1:
local dash = "\xe2\x80\x94"

-- Works:
local dash = "—"         -- UTF-8 bytes directly
local dash = string.char(226, 128, 148)  -- decimal escape
```

## Further reading

- `docs/ui-architecture.md` — how frames/layers/templates actually work
- `docs/custom-ui-patterns.md` — building beautiful custom UIs
- `docs/common-gotchas.md` — debugging quicksand pits
