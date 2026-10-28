---
sidebar_position: 1
title: "Configuration (`nebra.toml`)"
description: "Every nebra.toml key and section: metadata, dependencies, codegen, rules, stdlib, scripts, sides and reflection."
---

# Configuration (`nebra.toml`)

Every Nebra project lives next to a `nebra.toml` file. It declares metadata
(name, version, target Lua dialect), tells the compiler where source lives,
selects language rules, lists dependencies, and tunes the package manager and
test runner. The file is plain TOML and is loaded by the CLI on every
command that needs project context (`build`, `run`, `test`, `install`,
`docs`, `compile`).

The CLI walks the current directory and its ancestors looking for `nebra.toml`
- you can run a Nebra command from anywhere inside the project tree. A
minimal config:

```toml
name = "my-project"
version = "0.1.0"
target = "5.4"
```

Everything else is optional. Defaults below.

## Top-level keys

### `name` (string, optional)

Project name. Used as the package name when this project is consumed as a
dependency (see [Package Manager](../toolchain/package-manager.md)) and appears in
generated documentation.

### `version` (string, optional)

Project version. Free-form - the package manager treats values as
opaque tags unless they look like semver, in which case `^1.2` /`~1.2.3`
range matchers work.

### `target` (string, default `"5.4"`)

Lua dialect the compiler emits. Recognised values: `"5.1"`, `"5.2"`,
`"5.3"`, `"5.4"`, `"jit"` (or `"luajit"`). Any other value is an error that stops the build, as is
anything else in `nebra.toml` that cannot be read. The target gates which built-in operators and
library calls the codegen pass is allowed to use (e.g. integer division
`//` requires 5.3+).

### `entry` (string, optional)

Entry-point Nebra file for `nebra run` / `nebra compile`. Defaults to
`<source>/index.neb` when omitted.

### `output` (string, default `"out"`)

Directory the transpiled Lua files are written to (relative to the project
root). Wiped on every `nebra build` unless `--no-clean` is passed.

### `source` (string, default `"src"`)

Root directory the compiler scans for `.neb` sources. Files are imported
relative to this directory.

### `extends` (list of string, optional)

Other `nebra.toml` files (relative paths) whose settings are loaded as a
baseline before this file's keys override them. Useful for sharing a base
config across multiple projects in a monorepo:

```toml
extends = ["../shared/nebra.base.toml"]
```

`extends` chains transitively; a recursion limit of 10 levels guards
against cycles. Lists merge by appending, dictionaries by `TryAdd` (so the
extending file always wins on key conflict for scalar / section values, but
list keys accumulate).

### `preset` (string, optional)

Loads a named rule preset on top of `extends`. Built-in presets:

- `"strict"` - enables every code-quality rule: `rules.strict_nil`,
  `rules.immutable_default`, and `rules.exhaustive_match = "explicit"`.
  Also disables `rules.allow_any`. Equivalent to opting into all
  type-safety nets at once.
- `"relaxed"` - the default. Plain Lua plus types - no
  rule-level enforcement.

Settings from the preset can be overridden by anything in your own file
(or in `extends`).

### `minify` (bool, default `false`)

Strip whitespace and comments from the generated Lua. Mainly useful when
shipping to environments that load source at runtime.

### `generate_docs` (bool, default `false`)

When `true`, `nebra build` also runs the same emission step as `nebra docs`,
producing a doc site next to the generated Lua. See
[Doc Comments](../toolchain/doc-comments.md) and [CLI Reference](../toolchain/cli.md).

### `generate_declarations` (bool, default `true`)

When `true`, `nebra build` also emits `.d.neb` declaration files alongside
each compiled module so downstream consumers can type-check against this
project without re-reading the source.

### `globals` (list of string, default `[]`)

Extra directories or single files that are scanned for `.d.neb` files at
build time. The found declarations are loaded into the global type
universe - useful when you want to type external Lua APIs that ship
with the project (game-engine globals, framework macros, etc.).

```toml
globals = ["types", "vendor/lua-engine.d.neb"]
```

Directories are scanned recursively. Files must already end in `.d.neb`.

### `annotations` (list of string, default `[]`)

Directories or single `.neb` files that contain user-defined annotation
plugins. See [Annotations](./annotations.md) for the plugin format.

### `types_only` (bool, default `false`)

Marks the project as a *types-only* package - it ships only
`.d.neb` declarations and has no runnable source. `nebra build`, `compile`,
`run` and `test` become graceful no-ops; `nebra docs` still works. Use this
for type-shim packages (the moral equivalent of TypeScript's `@types/*`).

### `runtime_global` (string, optional)

The global this package is reachable as at runtime, for runtimes where one package cannot load the files of another, such as nanos world. The package's root module, `init.neb` in its source directory, sets the global from its exports, and a project that depends on the package reads it instead of loading files:

```nebra
import { Model, Collection } from "nanos-orm"   -- local _mod0 = NanosORM or error(...)
```

Values can only be imported from the package itself, since the global holds its root module; `import type` still reaches any module of the package. See [Packages loaded through a global](../language/modules.md#packages-loaded-through-a-global).

### `runtime_export` (string, default `"_G[%name] = %value"`)

How the root module sets `runtime_global`. `%name` stands for the quoted global name and `%value` for the exports. nanos world packages use `"Package.Export(%name, %value)"`.

### `assets` (table, optional)

Files copied verbatim into the output directory. Keys are paths relative
to the project root; values are paths relative to the output root.

```toml
[assets]
"assets/icon.png" = "icon.png"
"vendor/runtime.lua" = "runtime/runtime.lua"
```

## Dependencies

Three independent dictionaries, all keyed by package name. Values can be a
plain specifier string or an inline table with the same fields used by
`nebra add`.

```toml
[dependencies]
nanos-world-types = "github:nebra-lang/nanos-world-types@v1.2"
inspect = "github:kikito/inspect.lua"

[dev_dependencies]
luaunit = { git = "https://github.com/bluebird75/luaunit", tag = "v3.4" }

[peer_dependencies]
some-engine-runtime = "^1.0"
```

| Section | Installed when... |
|---|---|
| `[dependencies]` | always (production) |
| `[dev_dependencies]` | running locally / in CI (filtered out of published packages) |
| `[peer_dependencies]` | expected to be supplied by the consumer; checked but not auto-installed |

See [Package Manager](../toolchain/package-manager.md) for the full specifier
grammar.

## `[code]` - codegen tweaks

```toml
[code]
index_base = 0                 # default 0
concat_operator = "+"          # default "+"
string_interpolation = true    # default true
alt_boolean_operators = true   # default true
semicolons = "optional"        # default "optional"
import_statement = "require(%s)"  # default "require(%s)"
import_extension = ""          # default ""
import_path = "relative"       # default "relative"
strip_unused = true            # default true
```

- **`index_base`** - what `array[0]` translates to. `0` rewrites
  every index expression to `array[expr + 1]` at codegen time so Nebra looks
  zero-indexed while the emitted Lua stays valid. Set to `1` if you want
  Nebra to inherit Lua's quirky one-based indexing.
- **`concat_operator`** - symbol that becomes the string-concat
  operator in addition to `..`. The default `"+"` lets you write
  `"hello " + name` like JavaScript. The token must still be a valid
  binary operator in the Nebra grammar.
- **`string_interpolation`** - enables `` `hello {name}` ``
  template-literal syntax. Lowers to `"hello " .. tostring(name)`.
- **`alt_boolean_operators`** - allow `&&`, `||`, `!`, `!=` in
  addition to `and`, `or`, `not`, `~=`. Disable when you want to enforce
  Lua-only syntax for style consistency. The Lua-style forms are
  always accepted regardless of this setting.
- **`semicolons`** - `"optional"` (default), `"required"`, or
  `"forbidden"`. Controls whether the generated Lua allows / requires / rejects
  trailing `;` on statements.
- **`import_statement`** - pattern emitted for every Nebra `import`.
  `%s` is replaced with the **quoted** module path string. Override to
  integrate with a custom module loader (e.g. `"__require(%s)"` or
  `'Package.Require(%s)'`).
- **`import_extension`** - suffix appended to every import path
  before it lands in the `%s` substitution. Nebra *always* strips a
  trailing `.neb` from the source path first - that extension is never
  meaningful in the lowered Lua. Set this to `".lua"` for runtimes whose
  loader requires it (e.g. nanos-world: `Package.Require("Foo.lua")`).
  Default empty.
- **`import_path`** - how imports of the project's own modules are
  written. `"relative"` keeps the path as the source writes it, such as
  `../types/Calendar`. `"root"` writes the module's normalized path from
  the output root, such as `Server/types/Calendar`, so every importer names
  a module the same way. Use it for loaders that cache modules under the
  path they are given without resolving it, as nanos world does; there a
  relative path loads one module once per spelling. Modules of packages and
  declarations keep the path as written.
- **`strip_unused`** - remove unreferenced locals / functions /
  imports from the generated Lua. Off for friendlier debug output;
  default on for smaller bundles.

## `[mangle]` - symbol name mangling

```toml
[mangle]
enabled = false               # default false
mangle_locals = true          # default true (only applied when enabled)
mangle_params = true          # default true (only applied when enabled)
mangle_top_level = false      # default false
keep_function_names = true    # default true
```

Mangling rewrites symbol names to short tokens (`a`, `b`, `c`, ...) to
shrink the output. Top-level names are kept by default because they form
the public surface of the bundle. Disable `keep_function_names` only if
you've verified that nothing outside the bundle calls into named
functions by string lookup.

## `[rules]` - type-safety knobs

```toml
[rules]
allow_any = true              # default true
strict_nil = false            # default false
immutable_default = false     # default false
deep_freeze = false           # default false
exhaustive_match = "none"     # default "none"
```

- **`allow_any`** - if `false`, `any` is rejected as a type
  annotation and inferred-`any` locals become errors. Forces the
  codebase to spell out every type. Most projects keep this `true` and
  use type narrowing where it matters.
- **`strict_nil`** - assigning `nil` to a non-`?` variable is an
  error. See [Nilability](../language/nilability.md).
- **`immutable_default`** - `local x = ...` is immutable by
  default; you must spell `local mut x = ...` to allow reassignment.
- **`deep_freeze`** - companion to `immutable_default`. Also
  freezes data *referenced* by immutable locals via a `__newindex`
  metatable trap. Adds runtime overhead on table writes - opt in
  only when you genuinely need it.
- **`exhaustive_match`** - how strictly the compiler checks
  `if`/`elseif`/`match` chains over unions and enums:
    - `"none"` - no check (default)
    - `"relaxed"` - an `else` branch counts as a catch-all
    - `"explicit"` - every variant must have its own branch;
      `else` is allowed but doesn't satisfy the check

## `[stdlib]` - embedded Lua declarations

```toml
[stdlib]
enabled = true                          # default true
disabled = ["string.dump", "io"]        # default []
```

The Nebra compiler ships with `.d.neb` declarations for the Lua standard
library so you get types on `print`, `string.format`, `math.pi`, etc.
out of the box.

- **`enabled = false`** - suppress the *whole* embedded stdlib.
  Use this when targeting a sandboxed runtime that doesn't expose any
  standard Lua globals. You'll need to provide your own globals via
  `globals = [...]`.
- **`disabled`** - surgically remove individual entries. Names
  accept two forms:
    - bare name (`"math"`, `"print"`) - removes the whole package
      or top-level binding
    - dotted name (`"string.dump"`, `"math.pi"`) - removes a
      single member of a package

Useful when your target sandbox forbids specific Lua features (e.g.
`string.dump`, `io.popen`, `os.execute`).

## `[scripts]` - lifecycle hooks

```toml
[scripts]
pre_build = ["echo building...", "node scripts/gen-types.js"]
post_build = []
pre_install = []
post_install = []
```

Lists of shell commands run at specific points:

- **`pre_build`** / **`post_build`** - run before / after every
  `nebra build`.
- **`pre_install`** / **`post_install`** - run by `nebra install` for
  *this* package when consumed as a dependency. **Off by default;**
  opt-in via `nebra install --allow-scripts` to protect against malicious
  packages.

Commands run in the project root with the project's environment.
Non-zero exit aborts the operation.

## `[install]` - package manager knobs

```toml
[install]
linker = "auto"                # default "auto"
allow_scripts = false          # default false
```

- **`linker`** - how `nebra install` materialises packages from the
  global store into `nebra_modules/`:
    - `"auto"` - symlink on Unix, junction on Windows (default)
    - `"symlink"` - force POSIX symlink
    - `"junction"` - force Windows directory junction
    - `"copy"` - physical copy (slower; useful when the consuming
      tool can't follow links)
- **`allow_scripts`** - project-level opt-in equivalent to the
  CLI flag. When `true`, lifecycle hooks declared in dependencies'
  `[scripts]` blocks are allowed to run during `nebra install`.

## `[test]` - test runner

```toml
[test]
dirs = ["tests", "test"]               # default ["tests", "test"]
patterns = ["_test.neb", ".test.neb"]  # default
quiet = false                           # default false
```

- **`dirs`** - directories scanned recursively for test files.
  Test files anywhere inside these directories are picked up regardless
  of their filename suffix. The configured `source` directory is *always*
  scanned in addition to this list.
- **`patterns`** - filename suffixes that mark a file as a test
  when it lives outside the test directories. Matching is
  case-insensitive.
- **`quiet`** - suppress the per-test tick line, leaving only the
  final summary on stdout. Default is verbose; switch on in CI to keep
  log noise down.

See [Testing](../toolchain/testing.md) for how the runner discovers and executes
tests.

## `[sides]` - client / server / shared scoping

```toml
[sides]
"src/Client/**" = ["client", "shared"]
"src/Server/**" = ["server", "shared"]
"src/Shared/**" = ["shared"]
```

Glob-to-side-list mapping that enforces multiplayer-sandbox execution-side
scoping for declarations annotated with `@side(...)`. Keys are glob
patterns (`*`, `**`, `?`) relative to the project root; values list the
side bits files matching that glob are allowed to reach.

Files outside any glob inherit `Side.All` (= reachable from anywhere) so
the feature is opt-in per project - just adding `[sides]` doesn't
silently restrict anything until you start emitting `@side` annotations.

See [Sides](./sides.md) for the full mental model and annotation syntax.

## `[reflection]` - runtime type metadata

```toml
[reflection]
mode = "all"        # "all" (default) | "annotated" | "none"
```

Controls how much runtime reflection metadata the compiler emits into the output
(the global `_G.__nebra_reflect` registry read by the `reflect` library).

| `mode` | Emits |
| --- | --- |
| `all` | Descriptors for every class, interface, enum, top-level function and variable. **Default.** |
| `annotated` | Only declarations marked `@reflectable`. |
| `none` | Nothing - strips the registry and reader library entirely. |

`all` is the default; `annotated` and `none` are opt-in size/overhead
optimizations. See [Reflection](./reflection.md) for the full API.

## Full example

```toml
name = "my-game"
version = "0.3.1"
target = "5.4"
entry = "src/init.neb"
output = "build"
source = "src"
preset = "relaxed"
globals = ["types"]
annotations = ["annotations"]

[dependencies]
inspect = "github:kikito/inspect.lua@v3"
nanos-world-types = "github:nebra-lang/nanos-world-types@v1"

[dev_dependencies]
luaunit = "github:bluebird75/luaunit@v3.4"

[code]
index_base = 0
string_interpolation = true
strip_unused = true

[rules]
strict_nil = true
immutable_default = false
exhaustive_match = "relaxed"

[stdlib]
disabled = ["string.dump", "io.popen", "os.execute"]

[scripts]
pre_build = ["node scripts/sync-engine-types.js"]

[test]
dirs = ["tests"]
quiet = false

[sides]
"src/Client/**" = ["client", "shared"]
"src/Server/**" = ["server", "shared"]
"src/Shared/**" = ["shared"]

[assets]
"assets/icon.png" = "icon.png"
```
