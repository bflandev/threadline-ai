---
name: excalidraw-generator
description: >
  Use this skill when the user wants to produce an Excalidraw flow diagram from a use case or set of use cases. Triggers include 'generate an excalidraw file', 'make a flow diagram for this use case', 'visualise this use case', 'create a swim lane diagram', 'export this to excalidraw'. Also use automatically in the agentic pipeline after screen-inventory and before stakeholder review. Output is always a valid .excalidraw JSON file written to disk. Do NOT use for Figma files, wireframes, or high-fidelity design - this skill produces rough flow diagrams only.
---

# Excalidraw Generator

Produces valid `.excalidraw` JSON files from use cases. Each file contains swim lanes, the happy path, extension branches, rough screen mockups, and a legend.

## Overview

Excalidraw files are plain JSON with a `.excalidraw` extension. The file is opened by dragging it onto the canvas at excalidraw.com. The format is well-suited to AI generation because it is declarative: every element has an explicit position, size, and style.

Always write the file to disk. Do not output the JSON in the chat.

## File structure

Every Excalidraw file has this top-level structure:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [...],
  "appState": {
    "gridSize": null,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

All content lives in the `elements` array. Every element requires these base fields:

```json
{
  "id": "<unique 16-char alphanumeric string>",
  "type": "<element type>",
  "x": 0, "y": 0, "width": 0, "height": 0,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "roundness": null,
  "seed": <random integer>,
  "version": 1,
  "versionNonce": <random integer>,
  "isDeleted": false,
  "boundElements": null,
  "updated": 1,
  "link": null,
  "locked": false
}
```

## Element types

### Rectangle

```json
{
  "type": "rectangle",
  "roundness": { "type": 3 }
}
```

Set `roughness: 1` for the hand-drawn aesthetic. Set `roughness: 0` for lane backgrounds.

### Text

```json
{
  "type": "text",
  "text": "Label text\nwith newlines",
  "fontSize": 13,
  "fontFamily": 1,
  "textAlign": "center",
  "verticalAlign": "top",
  "roundness": null
}
```

Every rectangle that needs a label requires a separate text element positioned inside it. Text elements do not inherit the parent rectangle's position — position them explicitly.

### Arrow

```json
{
  "type": "arrow",
  "points": [[0, 0], [dx, dy]],
  "startArrowhead": null,
  "endArrowhead": "arrow",
  "startBinding": null,
  "endBinding": null,
  "roundness": { "type": 2 }
}
```

Arrow `x` and `y` are the start point. `points` defines the path relative to the start. `[dx, dy]` is the displacement to the end point — calculate this from the target element's position.

## Layout conventions

Use a consistent layout across all use case diagrams so diagrams can be read as a set.

### Canvas dimensions

- Start content at x=40, y=20
- Title text at y=22, font size 22
- Subtitle (actors list) at y=54, font size 12, color `#888888`
- Swim lanes begin at y=90
- Lane height: 185px per lane
- Step boxes: width=155, height=68

### Swim lanes

One lane per actor. Lane backgrounds use `roughness: 0` and `roundness: null`.

| Actor type | Background colour | Border colour |
|---|---|---|
| Human actor (primary) | `#ddeeff` | `#aaccee` |
| System | `#f0f0f0` | `#cccccc` |
| Human actor (secondary) | `#eeffee` | `#aaddaa` |

Lane labels: text element at x=28, vertically centred in the lane, font size 13.

### Step boxes

- White background `#ffffff`, dark border `#1e1e1e`, roughness 1, rounded corners
- Position in the lane of the actor who owns that step
- X positions: space steps 220px apart starting at x=160
- Vertical centre: mid-point of the owning actor's lane

### Happy path arrows

- `strokeStyle: "solid"`, color `#1e1e1e`
- Diagonal when crossing lanes (actor → system or system → actor)
- Horizontal when staying in the same lane

### Extension branches

- `strokeStyle: "dashed"`
- Error/validation extensions: color `#c8860a` (amber)
- Alternate success extensions: color `#1a7a36` (green)
- Extensions hang below the system lane, y = system lane bottom + 60px
- Extension boxes: width=185, height=78
- Error extension box fill: `#fff3cd`, border: `#c8860a`
- Alternate success box fill: `#d4edda`, border: `#1a7a36`

### Screen mockups

Rough UI sketches above the top actor lane (y = -120 relative to lane top).

Structure each mockup as:
1. Outer frame rectangle: 165×135, fill `#fafafa`, border `#666666`, roughness 1
2. Title bar rectangle: 165×22, fill `#e0e0e0`, border `#666666`, font size 10
3. Content row rectangles: 148×16, fill `#eeeeee`, border `#cccccc`, font size 9, stacked with 22px gap
4. Optional badge rectangle: 100×18, fill and border match the semantic meaning (blue=action, amber=warning, green=success)

Connect mockups to their steps with dotted arrows (`strokeStyle: "dotted"`, color `#aaaaaa`).

### Legend

Place in the bottom-left corner, below all extension boxes:

```
──►  happy path
- -►  extension (alt/error)
····►  screen reference
```

Three text elements, y spaced 18px apart, color `#333333`, font size 11.

## Generation procedure

1. Parse the use case: extract actors, steps, and extensions
2. Determine lane count from actor list
3. Calculate canvas dimensions: width = (step count × 220) + 160, height = (lane count × 185) + 90 + extension space
4. Generate elements in this order: title → subtitle → lane backgrounds → lane labels → step boxes → happy path arrows → extension boxes → extension arrows → screen mockups → mockup arrows → legend
5. Assign unique IDs (16-char alphanumeric) to every element
6. Assign random seed and versionNonce integers to every element
7. Write the complete JSON to a `.excalidraw` file named `UC-XX-[slug].excalidraw`

## Script template

Use Python to generate the JSON. This avoids manual coordinate errors:

```python
import json, random

def make_id():
    return ''.join(random.choices('abcdefghijklmnopqrstuvwxyz0123456789', k=16))

def base_element(**kwargs):
    el = {
        "id": make_id(), "angle": 0, "fillStyle": "solid",
        "strokeWidth": 2, "strokeStyle": "solid", "roughness": 1,
        "opacity": 100, "groupIds": [], "frameId": None,
        "seed": random.randint(1, 99999), "version": 1,
        "versionNonce": random.randint(1, 99999),
        "isDeleted": False, "boundElements": None,
        "updated": 1, "link": None, "locked": False
    }
    el.update(kwargs)
    return el
```

Build rectangle, text, and arrow helper functions on top of this base. Calculate all positions programmatically from the step count and lane count — do not hardcode coordinates.

## Output

Save to: `/mnt/user-data/outputs/UC-XX-[title-slug].excalidraw`

Confirm: "File written: UC-XX-[title-slug].excalidraw — open at excalidraw.com by dragging the file onto the canvas."
