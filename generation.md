# Code Generation & Embedding

This page explains how blocks become textual JVM bytecode, the public API surface you can use from the host page or when embedding the editor, and how the bundle is built and served.

## The Generator Object

The core is a single `Blockly.Generator` instance:

```js
// src/generators/byteCodeGenerator.js
const byteCodeGenerator = new Blockly.Generator('byteCode');
```

It overrides `scrub_` so that statement blocks are simply concatenated in linear order (the `next` connection):

```js
byteCodeGenerator.scrub_ = function(block, code, thisOnly) {
    const nextBlock = block.nextConnection && block.nextConnection.targetBlock();
    if (nextBlock && !thisOnly) {
        return code + '\n' + byteCodeGenerator.blockToCode(nextBlock);
    }
    return code + '\n';
};
```

Every `forBlock['jvm_*']` (and `freeCode`) handler returns a string that ends with `\n` (or the generator adds one). The final `getCode()` trims the result.

## Smart Constant Emission (`jvm_push`)

The `jvm_push` block is implemented with a large switch on type + value that chooses the most compact opcode:

- `int` 0..5 → `iconst_0` … `iconst_5`
- `int` in byte range → `bipush`
- `int` in short range → `sipush`
- `long`/`double` → `ldc2_w` (or the 0/1 const forms)
- `float` 0/1/2 → `fconst_*`
- everything else → `ldc` (or `ldc_w`)

Individual `iconst_*`, `bipush`, `sipush`, `ldc*` blocks still exist if you want to force a particular encoding.

## Load/Store Compact Forms

Inside the load and store handlers the generator checks the index:

```js
if (index <= 3) {
    return `${type}load_${index}\n`;
}
return `${type}load ${index}\n`;
```

The same pattern applies to stores. This is purely a code-generation convenience; the blocks themselves only store the numeric index.

## Full forBlock Map

`byteCodeGenerator.forBlock` contains entries for every block type defined in `jvmBlock.js`:

- All the `iconst_*` / `lconst_*` / `fconst_*` / `dconst_*` / `aconst_null` constants
- `bipush`, `sipush`, `ldc`, `ldc_w`, `ldc2_w`
- `pop*`, `dup*`, `swap`
- `jvm_load` / `jvm_store` / `jvm_iinc`
- All arithmetic, bitwise, and conversion blocks (they read the TYPE dropdown and emit the two-letter prefix + opcode)
- All the `if*`, `goto`, `label`, `return` control blocks (they read OP / LABEL / TYPE fields)
- Object, field, static, cast, instanceof, new
- All four invoke* variants (they assemble the full `invokevirtual Class.method(Desc)Ret` line)
- Array new/aload/astore/length
- Monitor and athrow
- `freeCode` (verbatim passthrough)

If you add a new block you must add both the JSON definition and a matching `forBlock` entry, otherwise `blockToCode` will emit a warning comment.

## Public API Surface (`quantr.*` and module exports)

`src/index.js` exports (and also attaches to `window.quantr` for classic script usage):

```js
export function clearWorkspace()
export function saveWS()          // returns the compact JSON string
export function loadWS(json)      // loads into the workspace
export function getCode()         // textual bytecode (trimmed)
export function getWS()           // alias of saveWS
export function getBlockCode(block) // code for a single block (rarely used)
export const byteCodeGenerator
```

Inside the host page these are used by the "Copy", "Run", and project save/load buttons. The two change listeners keep the generated view and localStorage in sync:

```js
workspace.addChangeListener((e) => {
    if (e.isUiEvent) return;
    save(ws);
    const code = getCode();
    document.getElementById('generatedCode').innerHTML = ...;
});
```

## Serialization

`src/serialization.js` only contains the classic Blockly workspace save/restore helpers (`save(ws)` / `load(json)`). All persistence in the current editor is via `localStorage.setItem('mainWorkspace', ...)`.

The host blade template adds another layer on top (`currentProjectId`, `currentProjectName`, `toBinary` helper that base64-encodes the workspace JSON for the server).

## Styling & Custom Renderer

- Theme: `Blockly.Themes.quantr` (grid, flyout, toolbox colours, cursor).
- Custom `CustomCategory` (registered over the default) injects per-category SVG icons (from `src/icons/*.js`) and does the selection highlight + shadeColor animation.
- `CustomConstantProvider` overrides notch/tab/corner sizes (though the runtime uses the 'zelos' renderer).
- CSS lives in `src/index.css` (flex layout, `#blocklyDiv`, toolbox search box positioning, output pane) and is augmented by the host `public/css/jvm.css`.

## Building & Serving

From the `blockly-jvm` package directory:

```bash
npm install
npm run build          # or the production variant
```

Webpack outputs to `public/blockly-jvm/build-production/bundle.js` (the path the blade template expects).

During development the surrounding `newblock-server` usually runs a separate webpack-dev-server on port 8080 that serves the un-minified bundle with HMR.

The blade template has a small loader that tries the dev URL first and falls back to the production bundle on error.

## Embedding in Your Own Page

Minimal requirements:

1. A `<div id="blocklyDiv" style="height:600px"></div>`
2. The bundle script (or ES module import).
3. After load, the workspace is already injected; you only need to wire your own buttons to `quantr.getCode()`, `quantr.clearWorkspace()`, etc.
4. If you want to drive the generated view yourself, listen to `workspace.addChangeListener` and call `quantr.getCode()`.

Example (classic script):

```html
<script src="/blockly-jvm/build-production/bundle.js"></script>
<script>
  document.getElementById('copyBtn').onclick = () => {
    const code = window.quantr.getCode();
    navigator.clipboard.writeText(code);
  };
</script>
```

## Adding New Instructions

1. Add a JSON block definition in `src/blocks/jvmBlock.js` (use one of the existing category arrays).
2. Add a `forBlock['your_new_block']` handler in `byteCodeGenerator.js` that returns the opcode line(s) + `\n`.
3. Add the block to the appropriate category object in `src/toolbox.js`.
4. (Optional) Add an icon mapping in `CustomCategory.createIconDom_` if you introduce a new category.
5. Rebuild and test.

Because the generator is a plain text emitter, you can also just tell users to drop a `freeCode` block for brand-new or exotic opcodes while you implement the visual block.

## Further Reading

- [Overview](overview.md) – what the tool is for and its scope.
- [Getting Started](getting-started.md) – first program, search, labels, saving.
- [Block Categories & Reference](blocks.md) – every category and the exact bytecode each block emits.
- Source: `newblock-server/blockly-jvm/src/` (especially `byteCodeGenerator.js`, `jvmBlock.js`, `toolbox.js`, `index.js`).