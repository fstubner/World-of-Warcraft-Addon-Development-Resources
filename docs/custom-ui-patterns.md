# Custom UI Patterns

Patterns for building beautiful custom UIs in WoW.  Distilled from reading
the source of polished addons:

- **[Narcissus](https://github.com/Peterodox/Narcissus)** — character showcase
  addon famous for its cinematic polish.  Source of many patterns here.
- **[ElvUI](https://github.com/tukui-org/ElvUI)** — total UI replacement;
  shows what's possible with committed scope.
- **[Details!](https://github.com/Tercioo/Details-Damage-Meter)** — combat
  log analyzer with heavy custom chrome.
- **[WeakAuras](https://github.com/WeakAuras/WeakAuras2)** — pattern library
  for displays, animations, and textures.
- **[BagBrother / Combuctor](https://github.com/jaliborc/Combuctor)** — good
  examples of custom list/grid rendering.

## The polish gap

Between "basic addon" and "feels like a commercial product" are about six
specific techniques.  Most WoW addons implement 0–2 of them; polished ones
implement all six.

1. **Easing curves** — no transition is linear in real UI
2. **Centralized fade manager** — single OnUpdate driving all fades
3. **Smooth scroll with inertia** — lerp toward target instead of stepping
4. **Custom chrome** — replace Blizzard templates with matching theme
5. **Gradient textures** — vertex gradients instead of solid fills
6. **Nine-slice borders** — replace BackdropTemplate with own atlas

Each pattern is documented below with ready-to-copy code.

---

## Pattern 1: Penner easing library

Every animation should decelerate naturally instead of the linear ramp
`UIFrameFadeIn` provides.  Port of Robert Penner's equations (also used in
LibEasing and Narcissus/API/Easing.lua).

```lua
-- Usage: f(t, b, e, d)
--   t = elapsed time
--   b = begin value
--   e = end value (NOT change — actual target)
--   d = total duration
--   returns: interpolated value at time t

local sin, cos, pi = math.sin, math.cos, math.pi
local Easing = {}

function Easing.linear(t, b, e, d)
    return (e - b) * t / d + b
end

-- Fast start, gentle deceleration.  "Feels good" default for most UI.
function Easing.outSine(t, b, e, d)
    return (e - b) * sin(t / d * (pi / 2)) + b
end

-- Slow start, fast finish.  For elements sliding OFF screen.
function Easing.inSine(t, b, e, d)
    return -(e - b) * cos(t / d * (pi / 2)) + (e - b) + b
end

-- Slow start, fast middle, slow end.  For long transitions.
function Easing.inOutSine(t, b, e, d)
    return -(e - b) / 2 * (cos(pi * t / d) - 1) + b
end

-- Strong deceleration.  Good for "snap into place" motions.
function Easing.outQuart(t, b, e, d)
    t = t / d - 1
    return -(e - b) * (t*t*t*t - 1) + b
end

-- Slight overshoot then settle.  "Bouncy" entrance.
function Easing.outBack(t, b, e, d)
    local s = 1.70158
    t = t / d - 1
    return (e - b) * (t * t * ((s + 1) * t + s) + 1) + b
end

return Easing
```

**Default recommendation:** use `outSine` everywhere you'd use linear.  It
looks natural without being flashy.

---

## Pattern 2: Centralized fade manager

Narcissus runs ONE OnUpdate that iterates all active fades.  Compared to
`UIFrameFadeIn` (which creates separate tickers per frame):

- One OnUpdate instead of N (less GC pressure)
- Easing curves, not just linear
- Can cancel/override/stack fades cleanly
- FadeMixin injects `.FadeIn()` / `.FadeOut()` methods onto any frame

```lua
-- FadeManager.lua
local Fade = {}
local active = {}  -- [frame] = { from, to, duration, elapsed, easing, onFinish }
local driver = CreateFrame("Frame")
local running = false

local function onUpdate(_, elapsed)
    local any = false
    for frame, f in pairs(active) do
        f.elapsed = f.elapsed + elapsed
        if f.elapsed >= f.duration then
            frame:SetAlpha(f.to)
            if f.to <= 0.01 then frame:Hide() end
            if f.onFinish then f.onFinish(frame) end
            active[frame] = nil
        else
            local a = f.easing(f.elapsed, f.from, f.to, f.duration)
            frame:SetAlpha(a)
            any = true
        end
    end
    if not any then
        driver:SetScript("OnUpdate", nil)
        running = false
    end
end

function Fade:FadeFrame(frame, duration, fromAlpha, toAlpha, easing, onFinish)
    if toAlpha > fromAlpha then frame:Show() end
    frame:SetAlpha(fromAlpha)
    active[frame] = {
        from = fromAlpha, to = toAlpha,
        duration = duration, elapsed = 0,
        easing = easing or Easing.outSine,
        onFinish = onFinish,
    }
    if not running then
        driver:SetScript("OnUpdate", onUpdate)
        running = true
    end
end

function Fade:FadeIn(frame, duration, onFinish)
    self:FadeFrame(frame, duration or 0.22, 0, 1, nil, onFinish)
end

function Fade:FadeOut(frame, duration, onFinish)
    self:FadeFrame(frame, duration or 0.18, frame:GetAlpha(), 0, nil, onFinish)
end

return Fade
```

---

## Pattern 3: Smooth scroll with inertia

Narcissus's single biggest "feel" contribution.  On each mouse-wheel tick,
set a **target** scroll position; each frame, lerp the current position
toward the target at 14% of remaining distance.  Result: exponential
deceleration into the target.

```lua
-- SmoothScroll.lua
local LERP_SPEED = 0.14   -- proportion of remaining distance per normalized frame
local MIN_DELTA  = 0.5
local SHIFT_MULT = 2.5

local scrolls = {}  -- [ScrollFrame] = { target, maxScroll, animating, onScroll }
local driver = CreateFrame("Frame")
local running = false

local function onUpdate(_, elapsed)
    local timeFactor = elapsed * 60  -- normalize to 60fps for consistent speed
    local any = false

    for sf, s in pairs(scrolls) do
        if s.animating then
            local cur = sf:GetVerticalScroll()
            local diff = s.target - cur
            if math.abs(diff) < MIN_DELTA then
                sf:SetVerticalScroll(s.target)
                s.animating = false
                if s.onScroll then s.onScroll() end
            else
                local step = diff * LERP_SPEED * timeFactor
                step = (step > 0) and math.max(step, MIN_DELTA) or math.min(step, -MIN_DELTA)
                local next = math.max(0, math.min(cur + step, s.maxScroll))
                sf:SetVerticalScroll(next)
                if s.onScroll then s.onScroll() end
                any = true
            end
        end
    end

    if not any then
        driver:SetScript("OnUpdate", nil)
        running = false
    end
end

function ApplySmoothScroll(sf, stepSize, onScroll)
    local state = { target = 0, maxScroll = 0, animating = false, onScroll = onScroll }
    scrolls[sf] = state

    sf:EnableMouseWheel(true)
    -- HookScript, NOT SetScript, to preserve UIPanelScrollFrameTemplate's
    -- built-in scrollbar-tracking handler.
    sf:HookScript("OnMouseWheel", function(self, delta)
        local child = self:GetScrollChild()
        local childH = child and child:GetHeight() or 0
        state.maxScroll = math.max(0, childH - self:GetHeight())

        local mult = IsShiftKeyDown() and SHIFT_MULT or 1
        local step = -delta * stepSize * mult
        state.target = math.max(0, math.min(state.maxScroll, state.target + step))
        state.animating = true
        if not running then
            driver:SetScript("OnUpdate", onUpdate)
            running = true
        end
    end)
end
```

Apply at frame creation:

```lua
ApplySmoothScroll(myScrollFrame, ROW_HEIGHT * 3, function()
    -- Optional: called every frame during scroll.  Useful for virtual
    -- row repainting in a virtualized list.
    self:RepaintVisibleRows()
end)
```

---

## Pattern 4: Custom button chrome

Stock `UIPanelButtonTemplate` has a stone-bevel look that doesn't fit
modern themes.  Replace with a flat dark panel + thin border + tinted label:

```lua
local function CustomButton(parent, text, w, h)
    local btn = CreateFrame("Button", nil, parent)
    btn:SetSize(w or 140, h or 26)

    -- Background: vertical gradient dark → slightly-lighter
    local bg = btn:CreateTexture(nil, "BACKGROUND")
    bg:SetAllPoints()
    bg:SetColorTexture(1, 1, 1, 1)  -- must set before gradient
    bg:SetGradient("VERTICAL",
        CreateColor(0.06, 0.05, 0.02, 0.92),
        CreateColor(0.14, 0.11, 0.04, 0.96))

    -- Thin gold border (4 edge textures)
    local top = btn:CreateTexture(nil, "ARTWORK")
    top:SetPoint("TOPLEFT"); top:SetPoint("TOPRIGHT"); top:SetHeight(1)
    top:SetColorTexture(1, 0.82, 0, 0.55)

    local bot = btn:CreateTexture(nil, "ARTWORK")
    bot:SetPoint("BOTTOMLEFT"); bot:SetPoint("BOTTOMRIGHT"); bot:SetHeight(1)
    bot:SetColorTexture(1, 0.82, 0, 0.25)

    local left = btn:CreateTexture(nil, "ARTWORK")
    left:SetPoint("TOPLEFT"); left:SetPoint("BOTTOMLEFT"); left:SetWidth(1)
    left:SetColorTexture(1, 0.82, 0, 0.40)

    local right = btn:CreateTexture(nil, "ARTWORK")
    right:SetPoint("TOPRIGHT"); right:SetPoint("BOTTOMRIGHT"); right:SetWidth(1)
    right:SetColorTexture(1, 0.82, 0, 0.40)

    -- Hover glow (HIGHLIGHT layer, engine-driven)
    local hl = btn:CreateTexture(nil, "HIGHLIGHT")
    hl:SetAllPoints()
    hl:SetColorTexture(1, 0.82, 0, 0.12)
    hl:SetBlendMode("ADD")

    -- Gold label
    local lbl = btn:CreateFontString(nil, "OVERLAY", "GameFontNormal")
    lbl:SetAllPoints()
    lbl:SetText(text or "")
    lbl:SetTextColor(1, 0.82, 0.15)
    btn.lbl = lbl

    return btn
end
```

### Close button (gold ×)

```lua
local function CustomCloseButton(parent, size)
    local btn = CreateFrame("Button", nil, parent)
    btn:SetSize(size or 20, size or 20)
    local lbl = btn:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
    lbl:SetAllPoints()
    lbl:SetJustifyH("CENTER")
    lbl:SetText("×")
    lbl:SetTextColor(0.70, 0.58, 0.24)
    btn:SetScript("OnEnter", function(self) lbl:SetTextColor(1, 0.82, 0.10) end)
    btn:SetScript("OnLeave", function(self) lbl:SetTextColor(0.70, 0.58, 0.24) end)
    return btn
end
```

---

## Pattern 5: Gradient row backgrounds

Solid `SetColorTexture` is flat.  Gradients create visual depth.  For a
row in a list, a horizontal gradient (warm at the accent edge, cool at
the far edge) makes each row feel "lit from the left":

```lua
local bg = row:CreateTexture(nil, "BACKGROUND")
bg:SetAllPoints()
bg:SetColorTexture(0.08, 0.06, 0.02, 0.50)  -- fallback if gradient fails
bg:SetGradient("HORIZONTAL",
    CreateColor(0.14, 0.10, 0.03, 0.65),  -- warm amber at accent side
    CreateColor(0.04, 0.03, 0.01, 0.30))  -- near-black at far side
```

Apply to:
- List row backgrounds
- Window header bars (vertical: lighter top → darker bottom)
- Section dividers

---

## Pattern 6: Nine-slice backdrops (replace BackdropTemplate)

Blizzard's `SetBackdrop` requires the `BackdropTemplate` mixin since 9.0.1
and looks dated (stone-carved DialogBox border).  Narcissus builds its own
nine-slice atlas system.

**Concept:** one small texture file (typically 64×64) is split into 9
regions — 4 corners (fixed size), 4 edges (stretch along one axis), 1
center (stretches both axes).  When applied to a frame, the corners stay
pixel-perfect while edges and center scale with the frame.

```
┌──┬─────┬──┐
│TL│  T  │TR│    Corner size: 8px each
├──┼─────┼──┤    Edges: stretch to frame width/height - 2*corner
│L │  M  │ R│    Center: stretches both axes
├──┼─────┼──┤
│BL│  B  │BR│
└──┴─────┴──┘
```

Full implementation:

```lua
local C = 8 / 64  -- corner fraction in the atlas (8px corners, 64px atlas)
local REGIONS = {
    TL = { 0,     C,     0,     C   },  -- { left, right, top, bottom }
    TR = { 1-C,   1,     0,     C   },
    BL = { 0,     C,     1-C,   1   },
    BR = { 1-C,   1,     1-C,   1   },
    T  = { C,     1-C,   0,     C   },
    B  = { C,     1-C,   1-C,   1   },
    L  = { 0,     C,     C,     1-C },
    R  = { 1-C,   1,     C,     1-C },
}

local function ApplyNineSlice(frame, texPath, cornerSize)
    cornerSize = cornerSize or 8
    local r, g, b, a = 1, 0.82, 0, 0.8  -- gold tint

    local function tex(key, layer)
        local t = frame:CreateTexture(nil, layer or "BORDER")
        t:SetTexture(texPath)
        t:SetVertexColor(r, g, b, a)
        return t
    end

    local function setTC(t, region)
        t:SetTexCoord(region[1], region[2], region[3], region[4])
    end

    -- Dark fill behind border
    local fill = frame:CreateTexture(nil, "BACKGROUND")
    fill:SetPoint("TOPLEFT",     frame, "TOPLEFT",      6, -6)
    fill:SetPoint("BOTTOMRIGHT", frame, "BOTTOMRIGHT", -6,  6)
    fill:SetColorTexture(0.02, 0.02, 0.02, 0.94)

    -- Corners (fixed size)
    local tl = tex("tl"); tl:SetSize(cornerSize, cornerSize); tl:SetPoint("TOPLEFT");     setTC(tl, REGIONS.TL)
    local tr = tex("tr"); tr:SetSize(cornerSize, cornerSize); tr:SetPoint("TOPRIGHT");    setTC(tr, REGIONS.TR)
    local bl = tex("bl"); bl:SetSize(cornerSize, cornerSize); bl:SetPoint("BOTTOMLEFT");  setTC(bl, REGIONS.BL)
    local br = tex("br"); br:SetSize(cornerSize, cornerSize); br:SetPoint("BOTTOMRIGHT"); setTC(br, REGIONS.BR)

    -- Edges (stretch along one axis)
    local top = tex("top"); top:SetHeight(cornerSize)
    top:SetPoint("TOPLEFT",  tl, "TOPRIGHT")
    top:SetPoint("TOPRIGHT", tr, "TOPLEFT"); setTC(top, REGIONS.T)

    local bot = tex("bot"); bot:SetHeight(cornerSize)
    bot:SetPoint("BOTTOMLEFT",  bl, "BOTTOMRIGHT")
    bot:SetPoint("BOTTOMRIGHT", br, "BOTTOMLEFT"); setTC(bot, REGIONS.B)

    local left = tex("left"); left:SetWidth(cornerSize)
    left:SetPoint("TOPLEFT",    tl, "BOTTOMLEFT")
    left:SetPoint("BOTTOMLEFT", bl, "TOPLEFT"); setTC(left, REGIONS.L)

    local right = tex("right"); right:SetWidth(cornerSize)
    right:SetPoint("TOPRIGHT",    tr, "BOTTOMRIGHT")
    right:SetPoint("BOTTOMRIGHT", br, "TOPRIGHT"); setTC(right, REGIONS.R)
end
```

Generate the atlas texture once in Node.js / Python / Photoshop.  The
texture should have a soft gold edge fading to transparent in the middle.

---

## Pattern 7: Custom scrollbar styling

Strip Blizzard's chunky chrome off `UIPanelScrollFrameTemplate`:

```lua
local function StyleScrollbar(sf)
    local sb = sf.ScrollBar
    if not sb then return end

    -- Hide up/down arrow buttons
    if sb.ScrollUpButton   then sb.ScrollUpButton:Hide();   sb.ScrollUpButton:EnableMouse(false)   end
    if sb.ScrollDownButton then sb.ScrollDownButton:Hide(); sb.ScrollDownButton:EnableMouse(false) end

    -- Thin 6px scrollbar
    sb:SetWidth(6)

    -- Hide the stock track textures
    for _, region in ipairs({ sb:GetRegions() }) do
        if region:GetObjectType() == "Texture" then region:SetAlpha(0) end
    end

    -- Gold thumb
    local thumb = sb:GetThumbTexture()
    if thumb then
        thumb:SetColorTexture(1, 0.82, 0, 0.55)
        thumb:SetSize(4, 40)
    end
end
```

---

## Pattern 8: Type icons / atlas glyphs

Small icon next to each list row — instant visual category identification.
Use `Interface\Icons\*` paths (stable across expansions) or your own atlas:

```lua
local TYPE_META = {
    quest     = { label = "QUEST",     color = {1.00, 0.82, 0.10},
                  icon  = "Interface\\Icons\\INV_Misc_Book_09" },
    dialog    = { label = "DIALOG",    color = {0.40, 0.80, 1.00},
                  icon  = "Interface\\Icons\\INV_Misc_Note_01" },
    cinematic = { label = "CINEMATIC", color = {0.80, 0.40, 1.00},
                  icon  = "Interface\\Icons\\INV_Misc_Spyglass_03" },
    -- ...
}

-- On row paint:
row.typeIcon:SetTexture(meta.icon)
row.typeIcon:SetVertexColor(meta.color[1], meta.color[2], meta.color[3], 0.85)
```

**Path tip:** `Interface\\Icons\\*` paths have been stable since Vanilla.
Ability/spell icons inside them (`ability_warrior_savageblow`, etc.) rarely
change.  Safe bet for cross-expansion compatibility.

---

## Pattern 9: Virtualized list rendering

For lists longer than ~50 items, don't create a frame per row.  Instead:

1. Create a FIXED pool of ~20 row frames (enough to cover the visible area)
2. Set the scroll-child height to `total_items * row_height` (gives correct scrollbar range)
3. On scroll, repaint the pool with data from the appropriate slice

```lua
local VISIBLE_ROWS = 18
local ROW_H = 46

-- One-time setup: create pool
local rows = {}
for i = 1, VISIBLE_ROWS do
    rows[i] = CreateRow(scrollChild)  -- your row factory
end

local function Repaint()
    local scroll = scrollFrame:GetVerticalScroll()
    local firstIdx = math.floor(scroll / ROW_H) + 1

    for i = 1, VISIBLE_ROWS do
        local dataIdx = firstIdx + i - 1
        local row = rows[i]
        local item = filteredData[dataIdx]
        if item then
            PaintRow(row, item, dataIdx)
            row:SetPoint("TOPLEFT", 0, -((dataIdx - 1) * ROW_H))
            row:Show()
        else
            row:Hide()
        end
    end
end

-- Drive repaint on scroll (in addition to filter/refresh)
scrollFrame:HookScript("OnVerticalScroll", Repaint)
```

Memory: 18 frames × ~8 textures each = ~150 UI objects, regardless of
whether your list has 10 or 10,000 items.  Compare to naïve approach
which creates N frames for N items.

---

## Pattern 10: Rich text via color-escape codes

WoW FontStrings support inline color via `|cAARRGGBB...|r`:

```lua
local body = ""
body = body .. "|cff7a7566Description:|r\n"              -- muted gray label
body = body .. "|cffb0a47eThe quest body text here|r\n"  -- cream value
body = body .. "\n"
body = body .. "|cff7a7566Rewards:|r\n"
body = body .. "|cffb0a47e  " .. rewardItem .. "|r\n"
body = body .. "|cffffd066  ✓ Chosen path|r\n"           -- gold highlight

fontString:SetText(body)
```

Use for multi-section detail panels.  Avoids needing a FontString per
section — one word-wrapped FontString handles all layout.

Hex format: `|cAARRGGBB` where each is 2 hex digits (AA=alpha, 00-FF each).

---

## Putting it together: Narcissus-tier window checklist

To ship a window that feels as polished as Narcissus:

- [ ] Nine-slice backdrop instead of `BackdropTemplate`
- [ ] Fade-in on `ShowBrowser()` via centralized fade manager + `outSine`
- [ ] Smooth scroll with inertia + overscroll bounce
- [ ] Custom scrollbar (thin gold thumb, no arrow buttons)
- [ ] Custom fonts (ship a TTF, e.g. Source Sans 3 under SIL OFL)
- [ ] Custom buttons (flat + gold border, not `UIPanelButtonTemplate`)
- [ ] Custom close button (gold ×)
- [ ] Custom search box (dark bg, magnifier glyph, placeholder text)
- [ ] Gradient backgrounds on rows and headers
- [ ] Type icons on list rows
- [ ] Vignette texture behind content for depth
- [ ] Virtualized row rendering for long lists
- [ ] Hover fade (120ms `outSine` instead of engine snap)
- [ ] Detail pane crossfade (0.55 → 1 over 180ms) on selection change
- [ ] Color-escape-coded rich text in detail pane
- [ ] `GetStringHeight` to size scroll children correctly

Each is 10-50 lines of code.  Together they're the difference between
"a WoW addon" and "a polished product."

## Further reading

- `docs/ui-architecture.md` — primitives these patterns build on
- `docs/common-gotchas.md` — pitfalls that'll bite you implementing the above
- [Narcissus/NarciDB/ScrollFrame.lua](https://github.com/Peterodox/Narcissus/blob/master/NarciDB/ScrollFrame.lua) — reference smooth-scroll implementation
- [Narcissus/API/Easing.lua](https://github.com/Peterodox/Narcissus/blob/master/API/Easing.lua) — reference easing library
