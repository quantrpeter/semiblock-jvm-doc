# SemiBlock JVM

**SemiBlock JVM** is a visual block-based editor for authoring Java Virtual Machine (JVM) bytecode directly. It lets you build sequences of low-level JVM instructions by dragging and dropping blocks, see the textual bytecode update live, and export or embed the results.

It is part of the SemiBlock family of visual programming tools (alongside editors for MicroPython on ESP32, IoT device workflows, and other targets) and is typically accessed from within the newblock-server web application.

## What It Is

- A **statement-oriented Blockly workspace** (blocks connect top-to-bottom via previous/next connections, just like lines of code).
- **70+ instruction blocks** covering the most common JVM opcodes, grouped into 13 color-coded categories.
- A **smart `push` block** (`jvm_push`) that automatically chooses the most compact constant instruction (`iconst_*`, `bipush`, `sipush`, `ldc`, `ldc2_w`, etc.) based on type and value.
- A **Free Code** escape hatch (`freeCode`) that accepts raw multiline text so you can paste or type any bytecode the visual blocks do not yet cover.
- **Live code generation**: every workspace change immediately produces equivalent textual bytecode (shown with syntax highlighting and line numbers).
- **Custom toolbox experience**: category icons, live search/filter, labeled separators, and a "quantr" theme.
- **Persistence & embedding API**: auto-saves to browser localStorage; exposes a small `quantr.*` (or module exports) surface for loading/saving workspaces and retrieving generated code.
- **Designed for education and exploration**: great for learning the JVM instruction set, experimenting with stack/locals/control flow, or generating skeleton bytecode for further hand-tuning.

## Key Concepts

- **Blocks are instructions**: Most blocks emit exactly one line (e.g. `iconst_0\n`, `iadd\n`, `invokevirtual java/lang/Object.toString()Ljava/lang/String;\n`).
- **Linear execution model**: The generator walks the top-level chain of blocks (using a custom `scrub_` that concatenates `next` blocks). There are no "value" sockets for expressions; you explicitly manage the operand stack with push/load/dup/etc.
- **Labels are strings**: Control flow uses `jvm_label`, `jvm_goto`, `jvm_if*` etc. Labels are free-form text fields; the generator just emits `label: name` and `goto name` / `if_icmpeq name` etc. You are responsible for matching them.
- **Types are explicit in many places**: Dropdowns on arithmetic, load/store, return, array, etc. let you pick `i`/`l`/`f`/`d`/`a` (and variants).
- **Free Code is trusted**: Anything you type in a `freeCode` block is emitted verbatim (plus a trailing newline). Use it for `ldc` constants with complex strings, custom attributes, or opcodes not yet modeled as blocks.

## Main UI Areas (in the host page)

- **Blockly workspace** (left, large): the visual editor with searchable toolbox.
- **Generated Code pane** (right): highlighted, line-numbered textual bytecode produced by the generator.
- **Host application extras** (in `jvm.blade.php`): project save/load (currentProjectId), simulator/debugger panels showing frames, local variables, operand stack, plus various tool buttons. These are provided by the surrounding SemiBlock server UI, not the core blockly-jvm bundle.

## When to Use It

- Teaching or learning JVM bytecode and the operand stack model.
- Rapidly sketching method bodies or static initializers as bytecode.
- Generating a starting point that you then copy into a `.class` assembler or further tooling.
- Exploring what the compiler would emit for small constructs (by building the equivalent stack operations by hand).

## Limitations / Scope

- It generates **textual bytecode listings**, not binary `.class` files or in-memory `java.lang.Class` objects (the surrounding simulator in the host page may interpret or execute the listing).
- No high-level language constructs (no `if` expressions, loops, or Java source); those would live in a "Language" layer (icon and style support exists but the category is not present in the current toolbox).
- You must understand the JVM stack, locals, and descriptor syntax (`()V`, `Ljava/lang/String;`, etc.).
- Control flow is manual (labels + gotos / if_*); there are no structured block constructs in the current instruction set.

## Technology

- Built on **Blockly v11** (`blockly ^11.0.0`).
- Uses a custom `Blockly.Generator('byteCode')`.
- Custom `ToolboxCategory` renderer for icons + selection highlighting.
- `@blockly/field-multilineinput` for the Free Code block.
- Packaged with Webpack 5; the bundle is served from `/blockly-jvm/build-production/bundle.js` (or a dev server at `http://localhost:8080/bundle.js`).
- Icons are inline SVGs per category.
- The host page also pulls in highlight.js + line-numbers plugin for pretty-printing the output.

See the [Table of Contents](toc.md) to explore getting started, the block categories, and how code generation and embedding work.
