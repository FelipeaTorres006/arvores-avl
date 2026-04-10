# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

**"Data Structure Visualizations"** — an interactive educational web tool by David Galles (University of San Francisco). Each page animates a classic algorithm or data structure on an HTML5 `<canvas>`. Users can insert/delete/search values and watch the algorithm execute step by step, with play/pause/rewind controls.

The index page (`Algorithms.html`) lists all available visualizations. `template.html` is the official starter for adding new ones.

## Running

Open any `.html` file directly in a browser. No build step, no npm, no server required. If assets fail to load due to browser CORS restrictions on `file://`, use:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/AVLtree.html
```

No tests, no linter, no package manager.

## Directory Layout

```
AnimationLibrary/   — generic animation engine (never algorithm-specific)
AlgorithmLibrary/   — one JS file per algorithm; Algorithm.js is the base class
ThirdParty/         — jQuery 1.5.2 + jQuery UI 1.8.11 (only used for the speed slider)
timages/            — jQuery UI theme images
*.html              — one page per visualization + Algorithms.html index
template.html       — copy this when adding a new visualization
visualizationPageStyle.css — layout for visualization pages
visualPages.css     — layout for non-visualization pages (index, about, faq)
```

## Architecture: The Command Pipeline

This is the most important thing to understand. **Algorithm code never touches the canvas directly.** It builds a list of string commands that the animation engine later parses and plays back.

### Step 1 — Algorithm emits commands

Inside any action method (e.g. `insertElement`), the algorithm calls:

```js
this.cmd("CreateCircle", id, label, x, y);
this.cmd("Move", id, newX, newY);
this.cmd("Step");   // ← marks the boundary between animation frames
this.cmd("SetText", 0, "Inserted!");
return this.commands;  // always return at the end
```

`this.cmd(name, ...args)` appends the string `"name<;>arg1<;>arg2<;>..."` to `this.commands[]`. The `<;>` separator is how `AnimationMain.js` later splits the command apart.

`"Step"` is special: everything between two `Step` commands executes together in one animation frame. Without `Step`, all operations appear instantaneously.

### Step 2 — `implementAction` wires the action into undo history

```js
// In a callback (e.g. button click):
this.implementAction(this.insertElement.bind(this), value);
```

`implementAction(fn, val)` pushes `[fn, val]` onto `this.actionHistory`, calls `fn(val)`, and hands the returned command array to `animationManager.StartNewAnimation()`.

### Step 3 — AnimationManager parses and executes

`AnimationMain.js` holds a 30ms `setTimeout` loop (`timeout()`). Each tick calls `animationManager.update()` and `objectManager.draw()`. The manager reads commands from `this.AnimationSteps[]`, splits each on `<;>`, and dispatches to `ObjectManager` (which owns the actual canvas objects). The `"Step"` command stops the loop until the current frame's animations complete.

### Full command reference (parsed in `startNextBlock()`)

| Command | Args |
|---|---|
| `CreateCircle` | id, label, x, y |
| `CreateLabel` | id, text, x, y [, centered] |
| `CreateHighlightCircle` | id, color, x, y [, radius] |
| `CreateRectangle` | id, label, w, h [, x, y, xJustify, yJustify] |
| `Move` | id, toX, toY |
| `Connect` | id1, id2 [, color, curve, directed, label, anchorSide] |
| `Disconnect` | id1, id2 |
| `Delete` | id |
| `SetText` | id, text [, index] |
| `SetForegroundColor` | id, color |
| `SetBackgroundColor` | id, color |
| `SetHighlight` | id, bool |
| `SetEdgeHighlight` | id1, id2, bool |
| `SetAlpha` | id, float |
| `SetLayer` | id, layer |
| `AlignRight` | id, referenceId |
| `Step` | _(no args — frame boundary)_ |

Colors can be `"#RRGGBB"` or `"0xRRGGBB"` — `parseColor()` normalizes both to `#` form.

## Two Separate Undo Systems

These are distinct and easy to confuse:

**1. Algorithm-level undo** (`Algorithm.prototype.undo` in `Algorithm.js`):
Triggered by the "Skip Back" button. Replays the entire `actionHistory` from scratch (calling `reset()` first, then re-running every stored `[fn, val]` pair except the last). This is O(n) in the number of past actions but gives correct state for any algorithm automatically.

**2. Animation-level undo** (`UndoFunctions.js`):
Triggered by "Step Back". Fine-grained: each command in `startNextBlock()` pushes a corresponding `UndoBlock` subclass (e.g. `UndoMove`, `UndoCreate`, `UndoConnect`, `UndoSetText`). Stepping backward replays these in reverse to unwind exactly one animation frame.

## Object ID Management

Every canvas object needs a unique integer ID. The convention is:

```js
this.nextIndex = 0;  // initialized in init()

var id = this.nextIndex++;  // allocate a new ID
this.cmd("CreateCircle", id, label, x, y);
```

`nextIndex` is monotonically increasing. Algorithms must track which IDs they've allocated (e.g. storing them in tree node structs). Object 0 is conventionally reserved for the status/explanatory text label at the top-left of the canvas.

## Inheritance and UI Wiring

All algorithms use ES5 prototype inheritance:

```js
function AVL(am, w, h) { this.init(am, w, h); }
AVL.prototype = new Algorithm();
AVL.prototype.constructor = AVL;
AVL.superclass = Algorithm.prototype;
```

`Algorithm.prototype.init` sets up the animation manager listeners and the `actionHistory` array. Every subclass must call it via `superclass.init.call(this, am, w, h)`.

**UI controls** are added in `addControls()` using helpers from `Algorithm.js`:
- `addControlToAlgorithmBar("Button", "Insert")` — returns the `<input>` element
- `addControlToAlgorithmBar("Text", "")` — returns a text input
- `addCheckboxToAlgorithmBar("Label")` — returns a checkbox
- `addRadioButtonGroupToAlgorithmBar(["A","B"], "groupName")` — returns array of radio inputs

Push every control into `this.controls[]` so `disableUI`/`enableUI` can disable them during animations.

**Each callback must go through `implementAction`**, never manipulate data structures directly:

```js
AVL.prototype.insertCallback = function() {
    var val = this.insertField.value;
    if (val != "") {
        this.insertField.value = "";
        this.implementAction(this.insertElement.bind(this), val);
    }
}
```

## Events

`AnimationManager` and `AnimationMain` emit these events (via `CustomEvents.js`):
- `AnimationStarted` / `AnimationEnded` / `AnimationWaiting` — drive button enable/disable state
- `AnimationUndoUnavailable` — fired when history is empty
- `CanvasSizeChanged` — fired when user resizes canvas; algorithms should override `sizeChanged(w, h)` to reposition elements

## Cookies

Animation speed, canvas width/height, and control panel position are persisted across page loads via `getCookie`/`setCookie` (defined in `AnimationMain.js`).

## AVL Tree Specifics (`AlgorithmLibrary/AVL.js`)

`AVLNode(data, graphicID, heightLabelID, x, y)` stores:
- `graphicID` — ID of the circle on canvas
- `heightLabelID` — ID of the small label showing the node's height
- `left`, `right`, `parent` — tree pointers
- `height` — integer height of subtree

The four rotations are: `singleRotateRight`, `singleRotateLeft`, `doubleRotateRight`, `doubleRotateLeft`. Each one updates both the JS tree pointers AND emits the corresponding `Connect`/`Disconnect` commands to animate the edge changes. After rotating, `resizeTree()` repositions all nodes to maintain proper visual spacing.

## Adding a New Algorithm

1. Copy `AlgorithmLibrary/MyAlgorithm.js` — it has detailed inline comments explaining every convention.
2. Copy `template.html`, replace the last `<script>` tag with the path to your new JS file.
3. Override `init`, `addControls`, `reset`, `disableUI`, `enableUI`.
4. Implement the `init()` global function at the bottom of your JS file that calls `initCanvas()` and instantiates your class.
5. Add a link to your page in `Algorithms.html`.
