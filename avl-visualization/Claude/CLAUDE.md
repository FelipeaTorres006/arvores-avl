# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Project

No build step required. Open any `.html` file directly in a browser (e.g., `AVLtree.html`, `BST.html`). If assets fail to load due to browser security restrictions, serve locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/AVLtree.html
```

There are no tests, no linting, and no package manager.

## Architecture Overview

This is a client-side algorithm visualization framework. Every visualization page is self-contained: a single `.html` file loads the shared libraries and one algorithm-specific JS file.

### Two-Layer Library Structure

**`AnimationLibrary/`** — the rendering and animation engine (never algorithm-specific):
- `AnimationMain.js` — global canvas setup, `initCanvas()`, play/pause/step/skip controls, the `timeout()` render loop (30ms interval), and the `returnSubmit()` input helper
- `ObjectManager.js` — owns all drawn objects; draws the canvas each frame
- `AnimatedObject.js` — base for all visual primitives (circle, rectangle, label, line, linked list node, B-tree node, highlight circle)
- `UndoFunctions.js` / `CustomEvents.js` — undo stack support and event bus

**`AlgorithmLibrary/`** — one JS file per algorithm:
- `Algorithm.js` — base class; defines `implementAction`, `undo`, `addControls` helpers, and `cmd()`. Also exports DOM helpers: `addControlToAlgorithmBar`, `addCheckboxToAlgorithmBar`, `addRadioButtonGroupToAlgorithmBar`, `addLabelToAlgorithmBar`
- Each algorithm (e.g., `AVL.js`, `BST.js`, `RedBlack.js`) extends `Algorithm` using ES5 prototype inheritance

### Prototype Inheritance Pattern

All algorithm files follow this pattern:
```js
function MyAlg(am, w, h) { this.init(am, w, h); }
MyAlg.prototype = new Algorithm();
MyAlg.prototype.constructor = MyAlg;
MyAlg.superclass = Algorithm.prototype;
```

### Command Pattern for Animation

Algorithm actions never manipulate the canvas directly. Instead they build a command array and hand it to the animation manager:

```js
MyAlg.prototype.insertElement = function(val) {
    this.commands = [];                          // always reset first
    var id = this.nextIndex++;
    this.cmd("CreateCircle", id, val, x, y);
    this.cmd("Move", id, newX, newY);
    this.cmd("Step");                            // marks one animation frame boundary
    return this.commands;
}
```

`implementAction(fn, val)` stores `[fn, val]` in `this.actionHistory`, calls `fn(val)`, and passes the returned command array to `animationManager.StartNewAnimation()`. Undo works by calling `reset()` then replaying all history entries except the last.

### Standard HTML Page Structure

Every `.html` page must have:
- `<table id="AlgorithmSpecificControls">` — filled by the algorithm's `addControls()`
- `<canvas id="canvas">` — the drawing surface
- `<table id="GeneralAnimationControls">` — filled by `AnimationMain.js`
- Script tags loading the full `AnimationLibrary/` stack, then `Algorithm.js`, then the specific algorithm JS
- `<body onload="init();">` where `init()` calls `initCanvas()` and instantiates the algorithm

### Adding a New Algorithm

1. Copy `AlgorithmLibrary/MyAlgorithm.js` as your starting template — it contains detailed inline documentation of every pattern to follow.
2. Create an HTML page by copying an existing one (e.g., `AVLtree.html`) and replacing the algorithm script tag.
3. The `nextIndex` counter is your object ID allocator — increment it for every new canvas object.
4. Push all UI controls into `this.controls[]` so they are automatically disabled/enabled during animations.
