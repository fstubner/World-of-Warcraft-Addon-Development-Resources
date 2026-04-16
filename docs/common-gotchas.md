# Common Gotchas

The debugging quicksand pits.  Each entry is a real bug encountered while
building addons, with the symptom, root cause, and fix.

## Lua 5.1 escape sequences

**Symptom:** String literal like `"Previously \xe2\x80\x94 Azeroth"` renders
in the game as `Previously xe2x80x94 Azeroth` — the hex escape bytes are
printed as literal text.

**Cause:** WoW uses Lua 5.1.  `\x` hex escapes were added in Lua 5.2.  In
5.1, `\x` is not a recognized escape — the backslash is discarded and the
rest is kept as literal characters.

**Fix:** Use the actual UTF-8 byte sequence in the source file, or decimal
escapes (which DO work in 5.1):
```lua
-- BROKEN:
local dash = "\xe2\x80\x94"

-- Works:
local dash = "—"                         -- UTF-8 bytes in source
local dash = string.char(226, 128, 148)  -- decimal escape
```

## TOC file encoding

**Symptom:** Your addon loads the first few Lua files listed in the TOC,
then stops.  Later entries never execute.  Error messages reference lines
far past the apparent issue.

**Cause:** UTF-8 box-drawing characters (`─`, `═`) or other non-ASCII content
in TOC comment lines.  WoW's TOC parser treats them as malformed and may
bail out of the file.

**Fix:** Use plain ASCII in TOC comments.  `#` works; use `-` or `=` for
separators, not Unicode box-drawing.

```toc
# ── BROKEN ──  (em-dash renders as garbage bytes)
# --- OK ---
```

## FontString without a font

**Symptom:** Red error box: `FontString:SetText(): Font not set`.

**Cause:** `CreateFontString(nil, "OVERLAY")` with no template argument —
the FontString has no FontObject inherited.  Then `SetText` errors because
there's nothing to render with.

**Fix:** Pass a Blizzard template name as the third argument at creation:
```lua
-- BROKEN:
local fs = frame:CreateFontString(nil, "OVERLAY")
fs:SetText("hello")   -- errors

-- Works:
local fs = frame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
fs:SetText("hello")
```

If using a custom font, either set it via `SetFontObject()` immediately
after creation, or use the template-baseline pattern (template argument
provides working font, then optionally upgrade).

## SetBackdrop on frames without BackdropTemplate

**Symptom:** `attempt to call method 'SetBackdrop' (a nil value)` when
calling `frame:SetBackdrop(...)`.

**Cause:** Since WoW 9.0.1, `SetBackdrop` only exists on frames that include
`BackdropTemplate` in their templates list.

**Fix:** Include the template:
```lua
local f = CreateFrame("Frame", "MyFrame", UIParent, "BackdropTemplate")
f:SetBackdrop({ ... })
```

Or skip `SetBackdrop` entirely and roll your own nine-slice
(see `custom-ui-patterns.md`).

## SetAllPoints collapses to zero size

**Symptom:** Frame is created but invisible.  Its parent is anchored but
has no explicit `SetWidth`/`SetHeight`.  Children with `SetAllPoints(parent)`
render at 0×0.

**Cause:** Anchor resolution.  When the parent's size comes from anchors
only (e.g., `SetPoint("TOPLEFT", ...)` + `SetPoint("BOTTOMRIGHT", ...)`),
the size is resolved lazily during layout.  Children with `SetAllPoints`
inherit via anchor chain; if the parent's size hasn't been resolved yet,
the children collapse.

**Fix:** Give the parent explicit size, OR give children explicit size
independent of the parent, OR defer frame access until after the parent
has been shown at least once.

```lua
-- SAFE:
local parent = CreateFrame("Frame", nil, UIParent)
parent:SetSize(400, 300)     -- explicit
parent:SetPoint("CENTER")

local child = CreateFrame("Frame", nil, parent)
child:SetAllPoints(parent)   -- now resolves cleanly
```

## ScrollFrame child with height = 1

**Symptom:** Text added to a ScrollFrame's child only renders the first
line; everything past that is clipped.

**Cause:** The scroll child was created with `SetSize(width, 1)` and the
FontString inside uses `SetAllPoints()`.  The FontString inherits the
1-pixel height of the container and clips all but the first line.

**Fix:** Either size the container to fit content after text is set, or
use partial anchoring so the FontString's height is flexible:

```lua
-- Fix 1: size container after text
bodyText:SetText(longString)
bodyContainer:SetHeight(bodyText:GetStringHeight() + 8)

-- Fix 2: flexible height via partial anchors
bodyText:SetPoint("TOPLEFT",  bodyContainer, "TOPLEFT",  0, 0)
bodyText:SetPoint("TOPRIGHT", bodyContainer, "TOPRIGHT", 0, 0)
-- No bottom anchor = height grows with content
```

## SetScript vs HookScript

**Symptom:** Using `SetScript("OnMouseWheel", ...)` on a
`UIPanelScrollFrameTemplate` breaks the scrollbar — it no longer tracks
the scroll position.

**Cause:** `SetScript` REPLACES the existing handler.
`UIPanelScrollFrameTemplate` has its own `OnMouseWheel` that updates the
scrollbar child — replacing it kills that wiring.

**Fix:** Use `HookScript` to run alongside the existing handler:
```lua
-- BROKEN (kills the scrollbar):
sf:SetScript("OnMouseWheel", function(self, delta) ... end)

-- Works (runs after the built-in):
sf:HookScript("OnMouseWheel", function(self, delta) ... end)
```

## Deprecated API: SelectQuestLogEntry

**Symptom:** Code calling `SelectQuestLogEntry(idx)` either fails silently
or errors in Midnight 12.0.

**Cause:** Deprecated.  Reward-query APIs accept `questID` directly now.

**Fix:** Drop the select-then-query dance:
```lua
-- OLD:
SelectQuestLogEntry(idx)
local money = GetQuestLogRewardMoney()

-- NEW:
local money = GetQuestLogRewardMoney(questID)
```

## C_QuestLog.GetQuestClassification doesn't exist

**Symptom:** `attempt to call nil value (method 'GetQuestClassification')`.

**Cause:** The function exists, but in `C_QuestInfoSystem`, not `C_QuestLog`.
Common confusion since other quest APIs live in `C_QuestLog`.

**Fix:**
```lua
-- WRONG:
C_QuestLog.GetQuestClassification(questID)

-- RIGHT:
C_QuestInfoSystem.GetQuestClassification(questID)
```

**General rule:** when in doubt, run `scripts/wow-api GetFoo` in this repo
to find which namespace a function lives in.

## Enum availability at ADDON_LOADED

**Symptom:** Concern that `Enum.QuestClassification` might be nil early in
the load cycle.

**Answer:** In Midnight 12.0, all `Enum.*` tables are populated at client
startup, BEFORE `ADDON_LOADED` fires.  No guard needed:

```lua
-- Unnecessary:
if Enum and Enum.QuestClassification then
    STORY_CLASSIFICATIONS[Enum.QuestClassification.Campaign] = true
end

-- Sufficient (and clearer):
STORY_CLASSIFICATIONS[Enum.QuestClassification.Campaign] = true
```

## Defensive nil-guards on AddEvent-normalized fields

**Symptom:** Code full of `e.title or ""`, `e.zone or ""` etc.

**Cause:** Historic defense against legacy save data.  If your `AddEvent`
(or equivalent write path) normalizes nil inputs to empty strings, these
guards are redundant and hide real bugs.

**Fix:** Normalize at the write boundary, trust the shape thereafter:
```lua
function Addon:AddEvent(type, title, zone, detail)
    table.insert(DB.events, {
        type   = type,
        title  = title  or "",
        zone   = zone   or GetRealZoneText() or "",
        detail = detail or "",
        ts     = time(),
    })
end

-- Elsewhere, trust the fields:
row.titleLabel:SetText(e.title)   -- no `or ""` needed
```

## Event order: PLAYER_LOGIN vs ADDON_LOADED

**Symptom:** `UnitName("player")` returns `nil` or `"Unknown"` at
`ADDON_LOADED`.

**Cause:** `ADDON_LOADED` fires before the player object is valid.
SavedVariables are ready; unit APIs aren't.

**Fix:** Defer unit-dependent initialization to `PLAYER_LOGIN`:
```lua
function Addon:OnAddonLoaded()
    -- OK: init SavedVariables, register events, create static tables
end

function Addon:OnPlayerLogin()
    -- OK: UnitName, GetRealmName, C_Map.GetBestMapForUnit("player")
end
```

## Ticker leak on /reload

**Symptom:** After several `/reload` cycles, an addon's background task
runs multiple times — one copy per reload.

**Cause:** `/reload` re-runs Lua code but DOESN'T clean up previously-created
`C_Timer.NewTicker` instances.  Each reload spawns a new ticker without
cancelling the old one.

**Fix:** Store the ticker handle and cancel before creating:
```lua
if Addon._exportTicker then Addon._exportTicker:Cancel() end
Addon._exportTicker = C_Timer.NewTicker(300, function() ... end)
```

Same applies to `UIFrameFadeIn` and any other long-running callback
registered through `SetScript("OnUpdate", ...)`.

## Registering an event name that doesn't exist

**Symptom:** Your `OnEvent` handler never fires for a given event.  No
errors, just silence.

**Cause:** `RegisterEvent` silently ignores unknown or misspelled events.
No runtime feedback.

**Fix:** Verify event names via `/eventtrace` in-game, or grep
`Blizzard_APIDocumentationGenerated/*Documentation.lua` for
`Events =` and look for the exact name.

## Frame that works, then breaks after /reload

**Symptom:** Frame displays correctly on first launch, looks broken after
`/reload`.

**Likely cause:** You're creating a `C_Timer.NewTicker` or similar during
frame build, and its state accumulates across reloads.  Or you're mutating
a module-local variable that persists in the Lua environment.

Use `/reload` MANY times during development.  If something only fails after
multiple reloads, you have a state-leak bug.

## Local files can't be reached via HTTP

**Concern:** "Can my Lua code read a file on disk?"

**Answer:** No.  WoW's Lua sandbox has zero file I/O.  No `io.open`, no
sockets, no HTTP.  The ONLY channels in/out are:

- `SavedVariables` (written on logout, read on login)
- The DataBroker protocol between addons (via LibDataBroker)
- User input (slash commands, chat, mouse)

For external communication (e.g. with a companion app), write to
SavedVariables and have the external process watch the file.

## /eventtrace output overwhelming

**Observation:** `/eventtrace` shows hundreds of events per second,
especially `COMBAT_LOG_EVENT_UNFILTERED`.

**Fix:** Click the filter field at the top of the EventTrace window.  You
can whitelist or blacklist event names.  Also see the "Ignore" button on
events you never care about — it persists across sessions.

## Error messages ref: FrameXML line numbers

**Observation:** Error traces show paths like `Interface/FrameXML/...` and
line numbers referring to Blizzard code, not your addon.

**Cause:** Blizzard's own UI code is the last stack frame before the error
bubbles up.  Your addon triggered the error indirectly, e.g. by passing
bad data to a Blizzard API.

**Fix:** Look further up the stack trace for the first line inside your
own addon's files.  That's where the actual bug is.  Use BugSack/BugGrabber
for better stack traces — the default WoW error popup shows only the top
frame.

## Further reading

- `docs/testing.md` — how to catch all these at dev time instead of in-game
- `docs/api-reference.md` — where to look up APIs to avoid wiki staleness
