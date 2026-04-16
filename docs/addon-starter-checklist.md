# Addon Starter Checklist (Midnight 12.0)

Minimum viable addon with everything wired correctly.  Each step shows the
WoW idiom behind it — not just "do this" but "here's why."

## File layout

```
Interface/AddOns/MyAddon/
├── MyAddon.toc            Declares the addon; lists files to load
├── MyAddon.lua            Entry point
├── modules/
│   ├── Core.lua           Your namespace + public API
│   └── UI.lua             Frames and rendering
└── libs/                  Embedded libraries (LibStub-based)
    └── LibDataBroker-1.1/
        └── LibDataBroker-1.1.lua
```

Folder name MUST match the TOC filename minus `.toc`.  `MyAddon/MyAddon.toc`,
not `MyAddon/myaddon.toc` or similar.

## The TOC file

```toc
## Interface: 120000
## Title: My Addon
## Notes: One-line description shown in the AddOn List
## Author: Your Name
## Version: 0.1.0
## Category: Miscellaneous
## IconTexture: Interface\AddOns\MyAddon\textures\icon

## SavedVariables: MyAddonDB
## SavedVariablesPerCharacter: MyAddonCharDB

# Comments start with a single #.  Blank lines are ignored.

# Embedded libraries (load first)
libs/LibDataBroker-1.1/LibDataBroker-1.1.lua

# UI infrastructure (load before anything that creates frames)
modules/UI.lua

# Core
MyAddon.lua
modules/Core.lua
```

### TOC directive reference

| Directive | Purpose |
|---|---|
| `## Interface` | Target WoW interface version.  `120000` = Midnight 12.0.  If this doesn't match the client, WoW marks the addon "Out of Date" and hides it unless "Load out of date AddOns" is checked |
| `## Title` | Display name in the AddOn List |
| `## Notes` | Tooltip description |
| `## Author` | Your name (shown in tooltip) |
| `## Version` | Your addon version (shown in tooltip) |
| `## Category` | Groups addons in the list.  Common: `Action Bars`, `Auctions`, `Buffs`, `Chat`, `Combat`, `Map`, `Miscellaneous`, `Professions`, `Quest & Leveling`, `Raid`, `UI`, `Tooltip` |
| `## IconTexture` | 64×64 icon shown next to the addon in the list |
| `## SavedVariables` | Account-wide persistent table (one file in WTF/Account/<NAME>/SavedVariables/) |
| `## SavedVariablesPerCharacter` | Per-character table (separate file per char) |
| `## Dependencies` | Comma-separated addon names that MUST load first (hard dep) |
| `## OptionalDeps` | Load-order hint (soft dep; addon still loads if missing) |
| `## LoadWith` | Load with specific addons loaded |
| `## DefaultState` | `enabled` (default) or `disabled` |

### TOC gotchas

- **ASCII only in comments.**  WoW's TOC parser chokes on UTF-8 box-drawing
  characters (`─`) and similar — anything past the malformed comment line
  may fail to load.  Stick to `#` + ASCII.
- **Paths use forward slash or backslash.**  Both work, but most addons use
  forward slashes for Lua paths and backslashes for Interface paths.
- **Blank lines and comment lines are ignored.**  Safe to organize freely.

## Entry point (MyAddon.lua)

```lua
-- MyAddon.lua
local ADDON_NAME = "MyAddon"
MyAddon = {}   -- your global namespace

-- Event dispatcher: a single hidden frame handles all subscribed events
local frame = CreateFrame("Frame", "MyAddonMainFrame", UIParent)
frame:RegisterEvent("ADDON_LOADED")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_LOGOUT")

frame:SetScript("OnEvent", function(_, event, ...)
    if event == "ADDON_LOADED" then
        local name = ...
        if name == ADDON_NAME then MyAddon:OnAddonLoaded() end
    elseif event == "PLAYER_LOGIN" then
        MyAddon:OnPlayerLogin()
    elseif event == "PLAYER_LOGOUT" then
        MyAddon:OnPlayerLogout()
    end
end)

function MyAddon:OnAddonLoaded()
    -- SavedVariables are NOW populated.  Init defaults if first run.
    if not MyAddonDB then
        MyAddonDB = {
            version = 1,
            settings = { autoRun = false },
            entries = {},
        }
    end
    if not MyAddonCharDB then
        MyAddonCharDB = { character = "", realm = "" }
    end

    -- Boot sub-modules in dependency order
    MyAddon.UI:Init()
end

function MyAddon:OnPlayerLogin()
    -- Character info is NOW valid
    MyAddonCharDB.character = UnitName("player")
    MyAddonCharDB.realm     = GetRealmName()
end

function MyAddon:OnPlayerLogout()
    -- Clean up tickers, flush state to SavedVariables
end

-- Slash commands
SLASH_MYADDON1 = "/myaddon"
SLASH_MYADDON2 = "/ma"
SlashCmdList["MYADDON"] = function(msg)
    msg = (msg or ""):lower():gsub("^%s+", ""):gsub("%s+$", "")
    if msg == "" then
        MyAddon.UI:ToggleBrowser()
    elseif msg == "help" then
        print("/myaddon         open main window")
        print("/myaddon clear   reset saved data")
    elseif msg == "clear" then
        MyAddonDB.entries = {}
        print("Cleared.")
    else
        print("Unknown command. Try /myaddon help")
    end
end
```

## Event timing

```
WoW Launches
    │
    ├─ Addon files LOADED (Lua executes top-level)
    │
    ├─ ADDON_LOADED fires (SavedVariables are ready)
    │
    ├─ ... later ...
    │
    ├─ PLAYER_ENTERING_WORLD (UI framework ready)
    │
    ├─ PLAYER_LOGIN (player object valid; UnitName/GetRealmName work)
    │
    ├─ ... gameplay ...
    │
    └─ PLAYER_LOGOUT (right before client exit; SavedVariables flush to disk)
```

**Don't create frames at file-load time** that depend on player data.
Defer to `PLAYER_LOGIN` or later.

## SavedVariables

Persistent Lua tables.  Declared in TOC with `## SavedVariables: MyAddonDB`
and `## SavedVariablesPerCharacter: MyAddonCharDB`.

On disk at:
```
WTF/Account/<ACCOUNT_NAME>/SavedVariables/MyAddon.lua              # account-wide
WTF/Account/<ACCOUNT_NAME>/<SERVER>/<CHARACTER>/SavedVariables/MyAddon.lua  # per-char
```

WoW reads these on startup and flushes them on `PLAYER_LOGOUT` / client exit.
Changes during play are NOT persisted until logout (or `/reload`).

```lua
-- Reading:  tables are globals after ADDON_LOADED
print(MyAddonDB.version)

-- Writing: just mutate them
MyAddonDB.entries[#MyAddonDB.entries + 1] = { ts = time(), note = "hello" }

-- Migration pattern: bump version and migrate on ADDON_LOADED
if not MyAddonDB.version or MyAddonDB.version < 2 then
    MigrateToV2(MyAddonDB)
    MyAddonDB.version = 2
end
```

**Cap table sizes.**  SavedVariables are loaded into memory on every login.
Large tables (10K+ entries) slow down startup.  Evict old data:

```lua
while #MyAddonDB.entries > 2000 do
    table.remove(MyAddonDB.entries, 1)
end
```

## A minimum viable UI

```lua
-- modules/UI.lua
MyAddon.UI = {}

local mainFrame

function MyAddon.UI:Init()
    -- Deferred frame creation
end

function MyAddon.UI:ToggleBrowser()
    if not mainFrame then mainFrame = self:_build() end
    if mainFrame:IsShown() then mainFrame:Hide()
    else mainFrame:Show() end
end

function MyAddon.UI:_build()
    local f = CreateFrame("Frame", "MyAddonMainFrame", UIParent, "BackdropTemplate")
    f:SetSize(600, 400)
    f:SetPoint("CENTER")
    f:SetFrameStrata("HIGH")
    f:SetBackdrop({
        bgFile = "Interface/Tooltips/UI-Tooltip-Background",
        edgeFile = "Interface/DialogFrame/UI-DialogBox-Border",
        tile = true, tileSize = 32, edgeSize = 32,
        insets = { left = 11, right = 12, top = 12, bottom = 11 },
    })
    f:SetBackdropColor(0.02, 0.02, 0.02, 0.94)
    f:SetMovable(true)
    f:EnableMouse(true)
    f:RegisterForDrag("LeftButton")
    f:SetScript("OnDragStart", f.StartMoving)
    f:SetScript("OnDragStop",  f.StopMovingOrSizing)

    -- Title
    local title = f:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
    title:SetPoint("TOP", 0, -16)
    title:SetText("My Addon")

    -- Close button
    local close = CreateFrame("Button", nil, f, "UIPanelCloseButton")
    close:SetPoint("TOPRIGHT", -4, -4)

    -- Register for Escape key
    tinsert(UISpecialFrames, "MyAddonMainFrame")

    return f
end
```

## Debug print helper

```lua
function MyAddon:Print(msg)
    DEFAULT_CHAT_FRAME:AddMessage("|cff7ec8e3[MyAddon]|r " .. tostring(msg))
end
```

Use `DEFAULT_CHAT_FRAME:AddMessage` instead of bare `print()`.  The latter
goes to whatever chat frame currently has focus — not reliable.

## Minimap button (via LibDBIcon)

Many addons include a minimap button for quick access.  Pattern:

```lua
local dataobj = {
    type  = "launcher",
    label = "MyAddon",
    icon  = "Interface\\AddOns\\MyAddon\\textures\\icon",
    OnClick = function(self, btn)
        if btn == "LeftButton" then MyAddon.UI:ToggleBrowser()
        elseif btn == "RightButton" then ShowMenu(self) end
    end,
    OnTooltipShow = function(tt)
        tt:AddLine("|cff7ec8e3MyAddon|r")
        tt:AddLine("Left-click: open", 1, 1, 1)
    end,
}

local LDB     = LibStub("LibDataBroker-1.1")
local LDBIcon = LibStub("LibDBIcon-1.0")
local obj     = LDB:NewDataObject("MyAddon", dataobj)
LDBIcon:Register("MyAddon", obj, MyAddonDB.minimapButton or {})
```

## Packaging for CurseForge

Add `.pkgmeta` at the repo root to tell the CurseForge BigWigs packager
what to include:

```yaml
package-as: MyAddon

ignore:
  - .claude
  - .git
  - .github
  - node_modules
  - scripts
  - "*.md"

externals:
  libs/LibDataBroker-1.1:
    url: https://repos.curseforge.com/wow/libdatabroker-1-1
    tag: latest
  libs/LibDBIcon-1.0:
    url: https://repos.curseforge.com/wow/libdbicon-1-0
    tag: latest
```

The packager auto-builds on new GitHub tags via GitHub Actions.  See
[BigWigsMods/packager](https://github.com/BigWigsMods/packager) for full syntax.

## Further reading

- `docs/ui-architecture.md` — how frames/layers/templates work
- `docs/custom-ui-patterns.md` — beyond minimum viable UI
- `docs/testing.md` — how to iterate on an addon without tearing your hair out
- `docs/common-gotchas.md` — the debugging pits
