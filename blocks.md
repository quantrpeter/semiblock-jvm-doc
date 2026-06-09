# Block Categories & Reference

This page documents the 13 toolbox categories and the most important special blocks (`jvm_push` and `freeCode`). Each entry shows the block's message, typical emitted bytecode, and notes.

All blocks are **statement blocks** (`previousStatement` / `nextStatement`). There are no expression/value blocks; you build the operand stack explicitly.

## Free Code (top-level category)

- `freeCode` — multiline text field (registered via `@blockly/field-multilineinput`).
  - Emitted: the exact text you typed, plus a trailing `\n`.
  - Use for: raw opcodes, complex `ldc` constants with spaces or quotes, directives, or anything the visual set does not cover yet.
  - Colour: `#6fc3fb` (also used for the planned Language category).

## Stack Operations (colour #72a4d6)

Constants (one per opcode):

- `iconst_0` … `iconst_5`, `lconst_0`/`lconst_1`, `fconst_0`…`fconst_2`, `dconst_0`/`dconst_1`, `aconst_null`

Explicit small/medium constants:

- `bipush <value>` (-128..127)
- `sipush <value>` (-32768..32767)
- `ldc <value>` (string or int/float constant pool entry)

Smart optimizer (recommended):

- `jvm_push` TYPE=(int|long|float|double|null) VALUE=...
  - Emits the shortest correct form:
    - int 0..5 → `iconst_*`
    - int -128..127 → `bipush`
    - int -32768..32767 → `sipush`
    - else → `ldc` (or `ldc2_w` for long/double)
  - Special cases for 0.0/1.0/2.0 float/double and null.

Stack manipulation:

- `pop`, `pop2`, `dup`, `dup_x1`, `dup_x2`, `dup2`, `swap`

## Local Variables (colour #66d8b2)

- `jvm_load` TYPE=(i|l|f|d|a) INDEX=0..255 → `iload 3` (or the compact `iload_3` form is chosen inside the generator for 0-3)
- `jvm_store` TYPE=... INDEX=... → `istore`, `astore`, etc.
- `jvm_iinc` INDEX=... by CONST=... → `iinc 1 1`

The generator in `byteCodeGenerator.js` has special handling to emit the `_0`..`_3` compact forms for loads/stores when the index is 0-3.

## Arithmetic (colour #CF63CF)

- `jvm_add` / `sub` / `mul` / `div` / `rem` / `neg` with TYPE dropdown (i|l|f|d)
  - Emits e.g. `iadd`, `fdiv`, `dneg`, `lrem`

## Bitwise Operations (colour #c1aaf1)

- `jvm_shl` / `shr` / `ushr` / `and` / `or` / `xor` (only i and l supported in the dropdowns)
  - Emits `ishl`, `lushr`, `iand`, etc.

## Comparisons (colour #9966FF)

Long / float / double comparisons that push -1/0/1 onto the stack:

- `lcmp`, `fcmpl`, `fcmpg`, `dcmpl`, `dcmpg`

These are distinct from the `if*` control-flow blocks below.

## Control Flow (colour #FFAB19)

All control flow is label-based (labels are plain text fields; you must keep them consistent).

- `jvm_if_icmp` OP=(eq|ne|lt|le|gt|ge) LABEL → `if_icmpeq foo`
- `jvm_if` OP=... LABEL → `ifeq`, `ifne`, `iflt`, ...
- `jvm_if_acmp` OP=(eq|ne) LABEL → `if_acmpeq`
- `jvm_ifnull` / `ifnonnull` LABEL
- `jvm_goto` LABEL
- `jvm_label` NAME → `name:`
- `jvm_return` TYPE=(i|l|f|d|a|void) → `ireturn`, `areturn`, `return`, ...

## Object Operations (colour #4C97FF)

- `jvm_new` CLASS → `new java/lang/StringBuilder`
- `jvm_getfield` / `putfield` CLASS FIELD DESC → `getfield com/example/Foo.value I`
- `jvm_getstatic` / `putstatic` CLASS FIELD DESC
- `jvm_checkcast` TYPE
- `jvm_instanceof` TYPE

## Method Invocation (colour #59C059)

- `jvm_invokevirtual` CLASS METHOD DESC → `invokevirtual java/lang/String.length()I`
- `jvm_invokespecial` CLASS METHOD DESC (use for `<init>` and private/super calls)
- `jvm_invokestatic` CLASS METHOD DESC
- `jvm_invokeinterface` INTERFACE METHOD DESC

All three fields are free-text inputs. You are responsible for correct internal names and JVM descriptors.

## Type Conversions (colour #FF6680)

All the primitive widening/narrowing conversions:

`i2l`, `i2f`, `i2d`, `l2i`, `l2f`, `l2d`, `f2i`, `f2l`, `f2d`, `d2i`, `d2l`, `d2f`, `i2b`, `i2c`, `i2s`

## Array Operations (colour #FF8C1A)

- `jvm_newarray` TYPE=(boolean|char|float|double|byte|short|int|long) → `newarray int`
- `jvm_anewarray` TYPE (reference) → `anewarray java/lang/String`
- `jvm_multianewarray` TYPE DIMS → `multianewarray [[I 2`
- `jvm_arraylength`
- `jvm_aload` / `astore` TYPE=(i|l|f|d|a|b|c|s) → `iaload`, `aastore`, `baload`, ...

## Monitor & Exception (colour #FF4D6A)

- `jvm_monitorenter`
- `jvm_monitorexit`
- `jvm_athrow`

## Miscellaneous (colour #8E8E8E)

- `jvm_nop`
- `jvm_ldc`, `jvm_ldc_w`, `jvm_ldc2_w` (the stack category also has a plain `ldc`; the misc versions are for explicit wide-index forms)

## Notes on Duplicates and Evolution

- `ldc` appears in both Stack Operations and Miscellaneous. The Stack version is the one most people reach for first.
- A "Language" category icon and style (`language_category` colour `#6fc3fb`) are prepared in `index.js` (imported `iconLanguage`, `categoryStyles`), but no Language category is present in the current `toolbox.js` export. It is reserved for a future higher-level language-to-bytecode layer.

## Block Implementation Location

All block JSON definitions live in `src/blocks/jvmBlock.js` (created via `Blockly.common.createBlockDefinitionsFromJsonArray` batches, one per category). The generator implementations are in `src/generators/byteCodeGenerator.js` (the `forBlock` map).

See [Code Generation & Embedding](generation.md) for how the generator walks the chain and how you can call it from your own code.