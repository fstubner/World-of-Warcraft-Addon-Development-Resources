# WoW UI Architecture

How WoW's UI system actually works.  Dense reference, not a tutorial.

## The core primitives

Everything visible in WoW's UI is built from four primitive types:

| Type | Purpose | Created via |
|---|---|---|
| **Frame** | Container; holds textures, font strings, child frames.  Receives events. | `CreateFrame(frameType, name, parent, template)` |
| **Texture** | Rendered image (file, atlas, solid color, or gradient).  Cannot receive events. | `frame:CreateTexture(name, drawLayer, template, subLevel)` |
| **FontString** | Rendered text with a FontObject assigned. | `frame:CreateFontString(name, drawLayer, template)` |
| **FontObject** | Named font resource (path + size + flags + color).  Inherited by FontStrings. | `CreateFont(name)` |

All other UI concepts (buttons, scroll frames, sliders, edit boxes) are
specialized Frame types that inherit behavior from the core Frame system.

## Frame types

`CreateFrame(frameType, ...)` accepts:

- `"Frame"` — plain container
- `"Button"` — click-receiving frame with `SetScript("OnClick", ...)`
- `"CheckButton"` — toggleable button (inherits Button)
- `"EditBox"` — text input
- `"ScrollFrame"` — scrollable viewport
- `"Slider"` — drag-to-value control
- `"StatusBar"` — fill-based bar (HP, cast, progress)
- `"Model"` / `"PlayerModel"` / `"DressUpModel"` — 3D model viewers
- `"MovieFrame"` — pre-rendered cinematic playback
- `"Minimap"` — the world minimap (only one allowed)
- `"CinematicModel"` / `"Cooldown"` / `"ArchaeologyDigSiteFrame"` — specialized

## Draw layers (Z-order)

Every Texture and FontString is drawn on a **layer**.  Within a frame, the
render order is:

```
BACKGROUND  →  BORDER  →  ARTWORK  →  OVERLAY  →  HIGHLIGHT
(behind)                                               (front)
```

Within a single layer, textures are drawn in the order they were created
UNLESS you pass a `subLevel` (integer -8 to 7, default 0).  Higher sublevels
render on top within the same layer.

```lua
local bg = frame:CreateTexture(nil, "BACKGROUND")           -- sublevel 0
local tint = frame:CreateTexture(nil, "BACKGROUND", nil, 1) -- renders ON TOP of bg
```

**HIGHLIGHT layer is special**: only rendered while the mouse is over the
frame.  Use this for engine-driven hover effects without writing OnEnter/OnLeave.

```lua
local hoverBg = button:CreateTexture(nil, "HIGHLIGHT")
hoverBg:SetAllPoints()
hoverBg:SetColorTexture(1, 1, 1, 0.1)  -- shown only on mouseover, auto
```

## Anchor system (`SetPoint`)

Frame positions are defined by **anchor relationships**, not absolute
coordinates.  An anchor says "my `anchorPoint` sits at `relativeFrame`'s
`relativePoint` with an offset."

```lua
--                (my point)  (relative frame) (their point)  (x, y offset)
fs:SetPoint("TOPLEFT",        parent,          "TOPLEFT",     10, -10)
```

Valid anchor points: `TOPLEFT`, `TOP`, `TOPRIGHT`, `LEFT`, `CENTER`, `RIGHT`,
`BOTTOMLEFT`, `BOTTOM`, `BOTTOMRIGHT`.

### Sizing via anchors (no explicit SetSize)

If you set **both** a top anchor and a bottom anchor (or left + right), the
frame sizes itself to fit the distance between them.  This is how most WoW
UI elements are positioned:

```lua
-- Width: fills parent (minus 10px margin each side)
-- Height: fixed at 20 (only TOPLEFT + TOPRIGHT anchors, no bottom)
fs:SetPoint("TOPLEFT",  parent, "TOPLEFT",  10, -20)
fs:SetPoint("TOPRIGHT", parent, "TOPRIGHT", -10, -20)
fs:SetHeight(20)
```

### Y-axis: POSITIVE IS UP

This trips everyone up.  WoW's anchor system uses math-style Y axis, NOT
screen-style.  **Positive y offset moves UP from the reference point.**

```lua
-- Place 30px UP from the parent's top-left:
child:SetPoint("TOPLEFT", parent, "TOPLEFT", 0, 30)

-- Place 30px DOWN from the parent's top-left:
child:SetPoint("TOPLEFT", parent, "TOPLEFT", 0, -30)
```

### `SetAllPoints(frame)` pitfall

`child:SetAllPoints(parent)` creates FOUR anchor points on `child` matching
the corresponding corners of `parent`.  It's equivalent to:

```lua
child:SetPoint("TOPLEFT",     parent, "TOPLEFT",     0, 0)
child:SetPoint("TOPRIGHT",    parent, "TOPRIGHT",    0, 0)
child:SetPoint("BOTTOMLEFT",  parent, "BOTTOMLEFT",  0, 0)
child:SetPoint("BOTTOMRIGHT", parent, "BOTTOMRIGHT", 0, 0)
```

**This requires `parent` to have a resolvable size.**  If parent is itself
anchored only (no explicit `SetWidth`/`SetHeight` and no conflicting anchors),
it may have zero dimensions at the time children are positioned, causing
children anchored to parent edges to collapse to zero width/height as well.

If you see a frame rendering but all its children invisible, this is usually
why.  Fix: give the parent explicit dimensions, or give children explicit
dimensions independent of the parent.

## Templates

Templates are Lua/XML definitions that can be applied to frames at creation
time to inject child widgets, scripts, and default configuration.

### Built-in templates worth knowing

| Template | Provides | Notes |
|---|---|---|
| `"BackdropTemplate"` | `SetBackdrop`, `SetBackdropColor`, `SetBackdropBorderColor` methods | Since 9.0.1, frames NEED this to use backdrops |
| `"UIPanelButtonTemplate"` | Stock Blizzard grey button | Stone-bevel look, doesn't fit modern themes |
| `"UIPanelCloseButton"` | Red X close button (top-right of frames) | Matches Blizzard frame chrome |
| `"UIPanelScrollFrameTemplate"` | ScrollFrame with built-in ScrollBar child | Chunky Blizzard chrome; can be restyled |
| `"InputBoxTemplate"` | EditBox with gold rivets | For plain text inputs |
| `"SearchBoxTemplate"` | Styled EditBox with magnifier | Blue tint — doesn't always fit custom themes |
| `"SliderTemplate"` | Slider with thumb | Used for scrollbars |
| `"GameTooltipTemplate"` | Tooltip frame with standard layout | For custom tooltip windows |

Example:
```lua
local btn = CreateFrame("Button", "MyButton", parent, "UIPanelButtonTemplate")
btn:SetSize(120, 26)
btn:SetText("Click Me")
btn:SetPoint("CENTER")
btn:SetScript("OnClick", function() print("clicked") end)
```

### Multiple templates

Templates can be combined by comma-separating them:

```lua
local f = CreateFrame("Frame", "MyFrame", UIParent, "BackdropTemplate,TooltipBorderedFrameTemplate")
```

## FontObjects and FontStrings

FontStrings display text.  They MUST have a font assigned before
`SetText` is called — otherwise WoW throws `FontString:SetText(): Font not set`.

Three ways to assign a font, in order of reliability:

### 1. Template at creation (most reliable)
```lua
local fs = parent:CreateFontString(nil, "OVERLAY", "GameFontNormal")
```
The template name refers to a named FontObject (a Blizzard global).  The
FontString inherits the font the moment it's created — atomic, can't fail.

### 2. SetFontObject after creation
```lua
local fs = parent:CreateFontString(nil, "OVERLAY")
fs:SetFontObject("GameFontNormalLarge")   -- string name
-- OR
fs:SetFontObject(MyCustomFontObject)       -- actual FontObject
```
Works as long as the named object exists and has a font set.  If you pass
a custom FontObject whose `SetFont` failed, the FontString has no font and
`SetText` will error.

### 3. Direct SetFont
```lua
fs:SetFont("Interface\\AddOns\\MyAddon\\fonts\\Inter.ttf", 13, "OUTLINE")
```
Bypasses FontObjects entirely.  Useful for one-off styling.  Path must use
backslashes (WoW interface paths) and be a valid .ttf / .otf file inside
the addon folder or `Fonts/` in the WoW install.

### Built-in Blizzard FontObjects

The most commonly used:

| Name | Size | Weight | Purpose |
|---|---|---|---|
| `GameFontNormal` | 12pt | regular | Default body text |
| `GameFontNormalSmall` | 10pt | regular | Labels, metadata |
| `GameFontNormalLarge` | 14pt | regular | Section headings |
| `GameFontNormalHuge` | 16pt | regular | Window titles |
| `GameFontHighlight` | 12pt | regular | White (not yellow) body text |
| `GameFontHighlightSmall` | 10pt | regular | White label text |
| `GameFontDisable` | 12pt | regular | Grey (disabled) text |
| `GameFontRed` / `Green` | 12pt | regular | Colored variants |
| `NumberFont_Shadow_Small` | tabular digits | Aligned number displays |

All available in `.../FrameXML/Fonts.xml` and `SharedFonts.xml`.

### Custom font objects

```lua
local font = CreateFont("MyAddon_Title")
font:SetFont("Interface\\AddOns\\MyAddon\\fonts\\Inter-SemiBold.ttf", 18, "")
font:SetTextColor(1, 0.82, 0)
font:SetShadowColor(0, 0, 0, 0.8)
font:SetShadowOffset(1, -1)

-- Now any FontString can inherit from it:
myFontString:SetFontObject(font)
```

**Gotcha:** if `SetFont` fails (wrong path, invalid TTF), the FontObject
exists but has no font.  Any FontString inheriting from it then errors on
`SetText`.  Always test your font path or set Frizqt as a baseline:

```lua
font:SetFont("Fonts\\FRIZQT__.TTF", 18, "")  -- guaranteed baseline
font:SetFont("Interface\\AddOns\\MyAddon\\fonts\\Inter.ttf", 18, "")  -- custom
```

## Textures

### SetTexture vs SetColorTexture

```lua
-- Use a texture file (TGA, BLP, or PNG/JPG in modern WoW):
t:SetTexture("Interface\\AddOns\\MyAddon\\textures\\button.tga")

-- Fill with a solid color:
t:SetColorTexture(1, 0.82, 0, 0.9)  -- RGBA, all 0-1
```

SetColorTexture is NOT a shortcut for SetTexture with a solid image — it
creates a procedural solid-color texture internally.  Efficient and
common for UI backgrounds.

### Gradients (Midnight 12.0)

Since Dragonflight, `SetGradient` takes a direction + two CreateColor objects:

```lua
t:SetColorTexture(1, 1, 1, 1)  -- must set a texture first
t:SetGradient("VERTICAL",
    CreateColor(0.1, 0.1, 0.1, 0.9),  -- top
    CreateColor(0.3, 0.2, 0.05, 1.0)) -- bottom
-- OR:
t:SetGradient("HORIZONTAL", leftColor, rightColor)
```

Pre-10.0 used `SetGradientAlpha` with different semantics — deprecated.

### TexCoord — using sprite atlases

One texture file can contain multiple images.  `SetTexCoord(left, right, top, bottom)`
selects a region:

```lua
t:SetTexture("Interface\\AddOns\\MyAddon\\textures\\atlas.tga")
-- Use only the bottom-right quadrant:
t:SetTexCoord(0.5, 1, 0.5, 1)
```

TexCoord values are normalized (0-1).  Excellent for button state changes
(normal/hover/pressed on rows of a single texture file).

### Masks

`SetMask("path")` clips the texture to the alpha of another image.  Used
for circular portraits, rounded corners, etc.

```lua
portrait:SetMask("Interface\\CharacterFrame\\TempPortraitAlphaMask")
```

### Blend modes

`SetBlendMode("MODE")` controls how the texture combines with what's behind it:

| Mode | Effect |
|---|---|
| `"DISABLE"` | Opaque, replaces (default for textures without alpha) |
| `"BLEND"` | Standard alpha blending (default for RGBA textures) |
| `"ADD"` | Additive — brighter than backdrop.  Used for glows. |
| `"MOD"` | Multiply — darker than backdrop.  Used for shadows. |
| `"ALPHAKEY"` | 1-bit transparency cutout |

## The scroll system

A `ScrollFrame` is a viewport that displays a portion of a larger child
frame (the "scroll child").  You control the visible portion with
`SetVerticalScroll(y)`.

```
┌─────────── ScrollFrame (viewport) ──────────┐
│  ┌────── Scroll Child (larger) ─────────┐   │
│  │ Line 1                                │   │ ← visible
│  │ Line 2                                │   │ ← visible
│  │ Line 3                                │   │ ← visible
├──┤ Line 4                                ├───┤ ← clipped (hidden)
│  │ Line 5                                │   │
│  │ ...                                   │   │
│  └───────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

### The built-in template

```lua
local sf = CreateFrame("ScrollFrame", "MyScroll", parent, "UIPanelScrollFrameTemplate")
sf:SetPoint("TOPLEFT",     parent, "TOPLEFT",     10, -30)
sf:SetPoint("BOTTOMRIGHT", parent, "BOTTOMRIGHT", -30, 10)  -- leaves room for scrollbar

local child = CreateFrame("Frame", nil, sf)
child:SetWidth(400)    -- IMPORTANT: must have explicit width
child:SetHeight(800)   -- height drives the scroll range
sf:SetScrollChild(child)

-- Add content to child...
local fs = child:CreateFontString(nil, "OVERLAY", "GameFontNormal")
fs:SetPoint("TOPLEFT", 0, 0)
fs:SetText("Lots of text...")
```

### The child-size trap

**The scroll child must have explicit size.**  If `child:SetHeight(1)` and
you add multi-line text via `SetAllPoints`, the text inherits the 1px height
and gets clipped.  Always size the child based on actual content:

```lua
-- After setting text:
child:SetHeight(textFS:GetStringHeight() + 16)
```

### Mouse wheel scrolling

`UIPanelScrollFrameTemplate` handles mouse wheel automatically.  If you
override `OnMouseWheel` with `SetScript`, you DESTROY the template's
handler and the scrollbar stops tracking.  Use `HookScript` instead:

```lua
sf:HookScript("OnMouseWheel", function(self, delta)
    -- your smooth-scroll lerp logic
end)
```

### Custom scrollbars

Blizzard's default scrollbar is chunky.  To replace its look without breaking
the template, style the existing scrollbar child:

```lua
local sb = sf.ScrollBar  -- Midnight 12.0 exposes this directly
sb:SetWidth(4)
sb.ScrollUpButton:Hide()
sb.ScrollDownButton:Hide()
for _, r in ipairs({ sb:GetRegions() }) do
    if r:GetObjectType() == "Texture" then r:SetAlpha(0) end  -- hide track
end
sb:GetThumbTexture():SetColorTexture(1, 0.82, 0, 0.55)  -- gold thumb
```

See `docs/custom-ui-patterns.md` for smooth-scroll (inertia) and custom
scrollbar full implementations.

## Events

Frames can register for events via `frame:RegisterEvent("EVENT_NAME")`.
The frame needs an `OnEvent` script:

```lua
local f = CreateFrame("Frame")
f:RegisterEvent("PLAYER_ENTERING_WORLD")
f:RegisterEvent("QUEST_ACCEPTED")
f:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_ENTERING_WORLD" then
        -- ...
    elseif event == "QUEST_ACCEPTED" then
        local questID = ...
        -- ...
    end
end)
```

### Event dispatch patterns

For addons with many events, a table-dispatch pattern is cleaner:

```lua
local Events = {
    PLAYER_ENTERING_WORLD = function(self) print("ready") end,
    QUEST_ACCEPTED = function(self, questID) print("quest:", questID) end,
}

local f = CreateFrame("Frame")
for evt in pairs(Events) do f:RegisterEvent(evt) end
f:SetScript("OnEvent", function(self, event, ...)
    Events[event](self, ...)
end)
```

### Unregistering

```lua
frame:UnregisterEvent("EVENT_NAME")
frame:UnregisterAllEvents()
```

## Slash commands

```lua
SLASH_MYADDON1 = "/myaddon"
SLASH_MYADDON2 = "/ma"               -- optional alias
SlashCmdList["MYADDON"] = function(msg, editbox)
    -- msg is the raw string after the slash command
    print("You said:", msg)
end
```

## Frame strata and level

Frames render in strata order.  Within a strata, higher `FrameLevel` renders on top.

```
BACKGROUND < LOW < MEDIUM (default) < HIGH < DIALOG < FULLSCREEN < FULLSCREEN_DIALOG < TOOLTIP
```

```lua
frame:SetFrameStrata("DIALOG")    -- always above most UI
frame:SetFrameLevel(5)             -- within DIALOG, render above level 4
```

## Portrait frames

```lua
local portrait = frame:CreateTexture(nil, "OVERLAY")
portrait:SetSize(40, 40)
portrait:SetPoint("TOPLEFT", 10, -10)
SetPortraitTexture(portrait, "player")  -- or any unit token
portrait:SetMask("Interface\\CharacterFrame\\TempPortraitAlphaMask")  -- circular crop
```

## Movable frames

```lua
frame:SetMovable(true)
frame:EnableMouse(true)
frame:RegisterForDrag("LeftButton")
frame:SetScript("OnDragStart", frame.StartMoving)
frame:SetScript("OnDragStop",  frame.StopMovingOrSizing)
```

## Closing with Escape

```lua
tinsert(UISpecialFrames, "MyFrameName")  -- string NAME, not the frame itself
```

The frame must have a name (the 2nd arg to `CreateFrame`).  Pressing Escape
while focused on the main UI will close any frame in `UISpecialFrames`.

## Animations

`CreateAnimationGroup()` on any Frame.  Each group can contain multiple
animations that run in sequence or parallel.

```lua
local ag = frame:CreateAnimationGroup()
ag:SetLooping("BOUNCE")  -- or "REPEAT" or "NONE"

local fade = ag:CreateAnimation("Alpha")
fade:SetFromAlpha(1.0)
fade:SetToAlpha(0.3)
fade:SetDuration(1.5)
fade:SetSmoothing("IN_OUT")

ag:Play()
```

Animation types: `Alpha`, `Scale`, `Translation`, `Rotation`, `Path`.

For smooth tweening between arbitrary values, see the custom-tween pattern
in `custom-ui-patterns.md`.

## Further reading

- `custom-ui-patterns.md` — patterns from Narcissus, ElvUI, and others for polished custom UIs
- `common-gotchas.md` — traps that will bite you
- `api-reference.md` — where to look up functions
