---
name: "Library/SBPele/SBPele"
tags: meta/library
---
This is a pure [Space Lua](https://silverbullet.md) library that adds [Pele](https://pele.run/) diagram rendering to SilverBullet.

[Pele](https://pele.run/) is a fast, dependency-free Mermaid-compatible renderer that produces themeable SVG. Write ordinary `pele` fenced blocks, or use the `sbpele` Space Lua API to build diagrams from your space's data.

# Installation

Install this page through SilverBullet's **Library Manager** using:

`github:jasinco/SBPele/SBPele.md`

Pele is loaded from jsDelivr by default. For a self-hosted/offline setup, download the browser module and set `config.set("sbpele", { bundlePath = "Library/SBPele/pele.js" })` to a local copy.

# Features

## Static diagrams

Write a fenced `pele` block and move your cursor out of it to live-preview:

    ```pele
    flowchart TD
        Start --> Stop
    ```

`mermaid` fences are also registered as an alias, so existing Mermaid blocks continue to work when this library is installed.

## Dynamic diagrams

The `sbpele` API generates diagrams at runtime from the object index. They update when the underlying page data changes:

```space-lua
${sbpele.diagram("flowchart LR\n  A[\"Hello\"] --> B[\"World\"]")}

${sbpele.diagram(sbpele.relationGraph{
  pages = query[[from index.pages("adr")]],
  relations = {"dependsOn", "related"},
})}
```

## API

### `sbpele.diagram(input, opts)`

Renders a raw Mermaid-compatible string or a diagram IR table. `opts` may contain `maxWidth` in pixels plus `width`, `height`, or `maxWidth` CSS lengths for the SVG.

### `sbpele.toSource(input)`

Serializes an IR table to Mermaid source without rendering it.

### `sbpele.relationGraph(spec)`

Builds a flowchart from a `pages` list. It supports `relations`, `groupBy`, `direction`, `color`, and `max`, matching the corresponding fields in the source pages. Wiki-link frontmatter values are used as edges when `relations` is omitted.

### `sbpele.timeline(spec)`

Builds a timeline from pages with a date field. Options are `dateField`, `groupBy`, `labelField`, `title`, and `max`.

### `sbpele.distribution(spec)`

Builds a pie chart counting pages by a categorical frontmatter field. Options are `pages`, `by`, `emptyLabel`, `title`, and `showData`.

# Configuration

```lua
config.set("sbpele", {
  -- jsDelivr browser module, or a local Space path when bundlePath exists.
  cdnUrl = "https://cdn.jsdelivr.net/npm/pele@1/dist/pele.js",
  bundlePath = "Library/SBPele/pele.js",
})
```

Pele follows the host theme through CSS variables and uses a high-contrast palette by default so labels, borders, and edges remain visible. The rendered widget adds the `pele` class; customize it with variables such as `--pele-bg`, `--pele-fg`, `--pele-line`, `--pele-surface`, and `--pele-accent`.

# Implementation

```space-lua
sbpele = sbpele or {}

local DEFAULTS = {
  cdnUrl = "https://cdn.jsdelivr.net/npm/pele@1/dist/pele.js",
  bundlePath = "Library/SBPele/pele.js",
}

local function safeId(raw)
  local id = string.gsub(tostring(raw), "[^%w_]", "_")
  if not string.match(id, "^[%a_]") then id = "n_" .. id end
  return id
end

local WRAP = {
  rect = {"[", "]"}, rounded = {"(", ")"}, stadium = {"([", "])",},
  circle = {"((", "))"}, diamond = {"{", "}"}, hexagon = {"{{", "}}"},
}
local EDGE = {
  solid = {arrow = "-->", plain = "---"},
  dotted = {arrow = "-.->", plain = "-.-"},
  thick = {arrow = "==>", plain = "==="},
}

local function label(s)
  return '"' .. string.gsub(tostring(s), '"', "#quot;") .. '"'
end

local function knownIds(elements, out)
  for _, e in ipairs(elements or {}) do
    if e.kind == "node" or e.kind == "subgraph" then
      out[safeId(e.id)] = true
      if e.kind == "subgraph" then knownIds(e.elements, out) end
    end
  end
end

local function emit(elements, known, indent, lines)
  for _, e in ipairs(elements or {}) do
    if e.kind == "node" then
      local w = WRAP[e.shape or "rect"] or WRAP.rect
      lines[#lines + 1] = indent .. safeId(e.id) .. w[1] .. label(e.label or e.id) .. w[2]
    elseif e.kind == "edge" then
      local from, to = safeId(e.from), safeId(e.to)
      if not known[from] then error('edge references missing node "' .. from .. '"') end
      local op = EDGE[e.style or "solid"] or EDGE.solid
      local connector = e.arrow == false and op.plain or op.arrow
      local text = e.label and ("|" .. label(e.label) .. "|") or ""
      lines[#lines + 1] = indent .. from .. " " .. connector .. text .. " " .. to
    elseif e.kind == "subgraph" then
      lines[#lines + 1] = indent .. "subgraph " .. safeId(e.id) .. (e.label and (" [" .. label(e.label) .. "]") or "")
      emit(e.elements, known, indent .. "  ", lines)
      lines[#lines + 1] = indent .. "end"
    elseif e.kind == "classDef" then
      local keys = {}
      for k in pairs(e.style or {}) do keys[#keys + 1] = k end
      table.sort(keys)
      local props = {}
      for _, k in ipairs(keys) do props[#props + 1] = k .. ":" .. e.style[k] end
      lines[#lines + 1] = indent .. "classDef " .. e.name .. " " .. table.concat(props, ",") .. ";"
    elseif e.kind == "click" then
      -- Pele exposes Mermaid links in RenderResult.links; the widget handler below
      -- turns internal links into SilverBullet navigation.
      lines[#lines + 1] = indent .. "click " .. safeId(e.id) .. " \"" .. tostring(e.target) .. "\""
    elseif e.kind == "raw" then
      lines[#lines + 1] = indent .. e.text
    else
      error("unknown element kind: " .. tostring(e.kind))
    end
  end
end

function sbpele.serialize(d)
  local lines = {}
  if d.type == "flowchart" then
    lines[#lines + 1] = "flowchart " .. (d.direction or "TD")
  else
    lines[#lines + 1] = d.title and (d.type .. " title " .. d.title) or d.type
  end
  local known = {}
  knownIds(d.elements, known)
  emit(d.elements, known, "  ", lines)
  return table.concat(lines, "\n")
end

function sbpele.toSource(input)
  return type(input) == "string" and input or sbpele.serialize(input)
end

local function cfg()
  local c = config.get("sbpele", {})
  return { cdnUrl = c.cdnUrl or DEFAULTS.cdnUrl, bundlePath = c.bundlePath or DEFAULTS.bundlePath }
end

local function importPele()
  local c = cfg()
  local m = space.fileExists(c.bundlePath) and js.importFromSpace(c.bundlePath) or js.import(c.cdnUrl)
  if m and m.default and not m.render then return m.default end
  return m
end

local function withSize(svg, width, height, maxWidth)
  if not (width or height or maxWidth) then return svg end
  local style = ""
  if width then style = style .. "width:" .. width .. ";" end
  if height then style = style .. "height:" .. height .. ";" end
  if maxWidth then style = style .. "max-width:" .. maxWidth .. ";" end
  local replacement, n = string.gsub(svg, '(<svg[^>]-style=")([^"]*)"', "%1%2;" .. string.gsub(style, "%%", "%%%%") .. '"', 1)
  if n == 0 then replacement = string.gsub(svg, "(<svg)", '%1 style="' .. string.gsub(style, "%%", "%%%%") .. '"', 1) end
  return replacement
end

-- Use a deliberately high-contrast palette so diagrams remain readable on both
-- SilverBullet light and dark themes. Pele inherits these variables through the
-- wrapper rather than relying on its relatively subtle defaults.
local function peleThemeStyle()
  local dark = false
  pcall(function() dark = js.window.document.documentElement.dataset.theme == "dark" end)
  if dark then
    return "--pele-bg:#111827;--pele-fg:#f9fafb;--pele-fg-on-light:#111827;--pele-fg-on-dark:#ffffff;--pele-muted:#d1d5db;--pele-line:#f9fafb;--pele-surface:#374151;--pele-surface-alt:#4b5563;--pele-border:#e5e7eb;--pele-accent:#93c5fd;"
  end
  return "--pele-bg:#ffffff;--pele-fg:#111827;--pele-fg-on-light:#111827;--pele-fg-on-dark:#ffffff;--pele-muted:#374151;--pele-line:#111827;--pele-surface:#f3f4f6;--pele-surface-alt:#e5e7eb;--pele-border:#374151;--pele-accent:#1d4ed8;"
end

function sbpele.diagram(input, opts)
  opts = opts or {}
  local width, height, maxWidth = opts.width, opts.height, opts.maxWidth
  if type(input) == "table" then
    width, height, maxWidth = width or input.width, height or input.height, maxWidth or input.maxWidth
  end
  local source = sbpele.toSource(input)
  local svg, links
  local ok, err = pcall(function()
    local result = importPele().render(source, { responsive = true, maxWidth = tonumber(opts.maxWidth), idPrefix = "sbpele-" .. tostring(os.time()) })
    svg, links = result.svg, result.links
  end)
  if not ok or not svg then
    local msg = string.gsub(tostring(err), "&", "&amp;")
    msg = string.gsub(msg, "<", "&lt;")
    return widget.html("<pre class='sbpele-error'>Pele error: " .. msg .. "</pre>")
  end
  svg = withSize(svg, width, height, maxWidth)
  -- Put the palette directly on the SVG as well as the wrapper. Pele's SVG
  -- contains its own fallback variables, so this guarantees contrast even if
  -- SilverBullet normalizes widget wrapper elements.
  local theme = peleThemeStyle()
  local themed, n = string.gsub(svg, '(<svg[^>]-style=")([^"]*)"', "%1%2;" .. theme .. '"', 1)
  if n > 0 then svg = themed end
  return widget.new {
    html = '<div class="pele sbpele-diagram" style="' .. peleThemeStyle() .. '">' .. svg .. "</div>",
    display = "block", markdown = "```pele\n" .. source .. "\n```",
    events = { click = function(e)
      local ok2, el = pcall(function() return e.data.target.closest("a") end)
      if ok2 and el then
        local href = el.getAttribute("href")
        if href and href ~= "" and not string.match(href, "^https?://") then editor.navigate(href) end
      end
    end },
  }
end

function sbpele.relationGraph(spec)
  spec = spec or {}; local max = spec.max or 60; local groupBy = spec.groupBy
  local function deref(v)
    if type(v) == "table" then v = v[1] end
    if type(v) ~= "string" then return nil end
    return string.gsub(string.gsub(v, "^%[%[", ""), "%]%]$", "")
  end
  local pages = {}
  for _, p in ipairs(spec.pages or {}) do pages[#pages + 1] = p end
  table.sort(pages, function(a,b) return string.lower(a.name or "") < string.lower(b.name or "") end)
  local ids, shown = {}, 0
  for _, p in ipairs(pages) do if shown < max then shown = shown + 1; ids[p.name] = "n" .. shown end end
  local function short(n) return string.match(n, "[^/]+$") or n end
  local elements = {}
  for _, p in ipairs(pages) do
    if ids[p.name] then elements[#elements+1] = {kind="node", id=ids[p.name], label=short(p.name), shape="rounded"}; elements[#elements+1] = {kind="click", id=ids[p.name], target=p.name} end
  end
  for _, p in ipairs(pages) do
    local from = ids[p.name]
    if from then
      local fields = spec.relations or {}
      if not spec.relations then for k in pairs(p) do if k ~= groupBy then fields[#fields+1] = k end end end
      for _, field in ipairs(fields) do
        local value = p[field]; if type(value) == "string" then value={value} end
        if type(value) == "table" then for _, v in ipairs(value) do local to=ids[deref(v)]; if to and to ~= from then elements[#elements+1]={kind="edge",from=from,to=to,label=field,style=field=="supersededBy" and "dotted" or "solid"} end end end
      end
    end
  end
  if shown < #pages then elements[#elements+1]={kind="raw",text="%% showing "..shown.." of "..#pages.." nodes"} end
  return {type="flowchart", direction=spec.direction or "LR", elements=elements, width=spec.width, height=spec.height, maxWidth=spec.maxWidth}
end

function sbpele.timeline(spec)
  spec=spec or {}; local field=spec.dateField or "date"; local pages={}
  for _,p in ipairs(spec.pages or {}) do if type(p[field])=="string" and string.match(p[field],"^%d%d%d%d%-%d%d%-%d%d") then pages[#pages+1]=p end end
  table.sort(pages,function(a,b)return a[field]<b[field] end); local elements={}
  if spec.title then elements[#elements+1]={kind="raw",text="title "..spec.title} end
  local shown=math.min(#pages,spec.max or 100)
  for i=1,shown do local p=pages[i]; local name=spec.labelField and p[spec.labelField] or (string.match(p.name,"[^/]+$") or p.name); elements[#elements+1]={kind="raw",text=p[field].." : "..string.gsub(tostring(name),":"," ")} end
  if shown<#pages then elements[#elements+1]={kind="raw",text="%% showing "..shown.." of "..#pages} end
  return {type="timeline",elements=elements,width=spec.width,height=spec.height,maxWidth=spec.maxWidth}
end

function sbpele.distribution(spec)
  spec=spec or {}; local counts, order={},{}; for _,p in ipairs(spec.pages or {}) do local v=p[spec.by]; if (v==nil or v=="") then v=spec.emptyLabel end; if v~=nil then v=tostring(v); if not counts[v] then counts[v]=0; order[#order+1]=v end; counts[v]=counts[v]+1 end end
  table.sort(order,function(a,b)return counts[a]>counts[b] or (counts[a]==counts[b] and a<b) end); local elements={}; if spec.title then elements[#elements+1]={kind="raw",text="title "..spec.title} end
  for _,v in ipairs(order) do elements[#elements+1]={kind="raw",text='"'..string.gsub(v,'"',"#quot;")..'" : '..counts[v]} end
  return {type=spec.showData and "pie showData" or "pie",elements=elements,width=spec.width,height=spec.height,maxWidth=spec.maxWidth}
end

codeWidget.define { language="pele", render=function(body) return sbpele.diagram(body) end }
codeWidget.define { language="mermaid", render=function(body) return sbpele.diagram(body) end }
```
