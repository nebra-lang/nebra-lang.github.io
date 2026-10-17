---
sidebar_position: 2
title: "Annotations"
description: "Compile-time IR rewriting with builtin annotations and your own annotation files."
---

# Annotations

Annotations are compile-time metadata that can **rewrite the IR**. They are Nebra's answer to decorators (TypeScript / Python), attributes (C#), and macros (Rust): a way to attach behavior to declarations without changing the call sites.

```lua
@log("starting work")
function work()
    -- ...
end
```

When the compiler meets `@log`, it looks up the annotation file named `log`, hands its `apply` function the declaration as a Lua table, and runs it in a sandboxed Lua interpreter. Whatever `apply` returns replaces the declaration before the rest of the compiler sees it.

---

## Writing an annotation

An annotation is a `.neb` file of its own. The file name is the annotation's name, so `annotations/log.neb` defines `@log`. It exports two things: an `annotation` table describing where it may be used and which arguments it takes, and an `apply` function.

```lua
-- annotations/log.neb
export local annotation = {
    targets = { "Function", "ClassMethod" },
    params = {
        message = { type = "string", required = false }
    }
}

export function apply(target, args)
    local label = args.message or "called"
    table.insert(target.body, 1, ir.exprStmt(ir.call("print", {
        ir.stringLiteral("[log] " .. label)
    })))
    return target
end
```

- `targets` lists the kinds of declaration the annotation may be written on (see [Targets](#targets)). A single kind can also be given as `target = "Function"`.
- `params` declares the arguments by name. Each one has a `type` (`"string"`, `"number"`, `"boolean"` or `"any"`), may be `required`, and may have a `default`. Only constant literals are accepted as arguments.
- `apply(target, args)` receives the declaration and the validated arguments.

Annotation files are compiled with a 1-based index, whatever `index_base` the project uses, and imports are emitted as plain `require` calls.

---

## Registering annotation files

Tell the compiler where your annotations live:

```toml
# nebra.toml
annotations = [
    "annotations",          # directory
    "lib/special.neb",      # a single file
]
```

In a directory, every file that exports an `annotation` table or an `apply` function is an annotation. Any other `.neb` file next to them is a helper module the annotations can import.

Installed dependencies contribute their annotations automatically.

---

## Helper modules

An annotation file can import other `.neb` files, so annotations can share code:

```lua
-- annotations/util/naming.neb
export function toSnakeCase(name: string): string
    return string.lower((string.gsub(name, "(%l)(%u)", "%1_%2")))
end
```

```lua
-- annotations/Table.neb
import { toSnakeCase } from "./util/naming"

export local annotation = { targets = { "Class" } }

export function apply(target, args)
    -- ...
end
```

The imported modules are compiled together with the annotation and bundled into its sandbox. They run in the same annotation environment, so they can use `ir` as well.

---

## Arguments

Positional and named arguments both work:

```lua
@log("work")              -- positional, bound in the order of `params`
@log(message = "work")    -- named
@log                       -- no arguments
```

`args` is a table keyed by parameter name, with defaults filled in, so the handler reads `args.message` however the annotation was written.

---

## Targets

| Target             | Applies to                          |
|--------------------|-------------------------------------|
| `Function`         | Global functions                    |
| `LocalFunction`    | `local function` declarations       |
| `Variable`         | `local` declarations                |
| `Parameter`        | Function parameters                 |
| `Enum`             | `enum` declarations                 |
| `EnumMember`       | Members of an enum                  |
| `Class`            | `class` declarations                |
| `ClassMethod`      | Methods of a class                  |
| `ClassField`       | Fields of a class                   |
| `ClassConstructor` | The constructor of a class          |
| `ClassAccessor`    | Getters and setters of a class      |
| `Interface`        | `interface` declarations            |
| `InterfaceMethod`  | Methods of an interface             |
| `InterfaceField`   | Fields of an interface              |
| `Extend`           | `extend` blocks                     |
| `ExtensionMethod`  | Methods of an `extend` block        |

Using an annotation on a kind it does not list is an error.

---

## What `apply` receives

The target is the declaration's IR as a Lua table. `__kind` names the node type (such as `"FunctionDecl"` or `"ClassDecl"`), and every property of the node is a camelCased field: a class has `fields`, `methods` and `annotations`, a field has `name`, `typeAnnotation` and `defaultValue`, a name is `{ name = "...", span = ... }`.

Annotations run before names and types are resolved. A type written by name, such as `Json<any>` or `DateTime`, therefore arrives with `kind = "Named"`: whether it is a class, an interface or an enum is not known yet.

A declaration's own annotations run before those of its members. A class annotation sees its fields and methods as they are written, including their annotations:

```lua
export function apply(target, args)
    for _, field in ipairs(target.fields) do
        for _, marker in ipairs(field.annotations or {}) do
            if marker.name.name == "Column" then
                -- ...
            end
        end
    end
    return target
end
```

The member annotations then run on whatever the class annotation returned.

---

## What `apply` returns

- The target itself, changed or not: it replaces the declaration.
- A list of statements: they replace the declaration, so an annotation can add code before or after it. When further annotations follow on the same declaration, the list has to contain a declaration of the same kind for them to run on.

---

## The `ir` helpers

The global `ir` builds new nodes without bookkeeping node ids:

| Helper | Builds |
|---|---|
| `ir.stringLiteral(value)`, `ir.numberLiteral(raw)`, `ir.boolLiteral(value)`, `ir.nilLiteral()` | Literals |
| `ir.nameExpr(name)` | A reference to a name |
| `ir.call(callee, args)` | A call; `callee` is a node or a name |
| `ir.methodCall(object, method, args)` | `object:method(args)` |
| `ir.dotAccess(object, field)` | `object.field` |
| `ir.newExpr(className, args)` | `new ClassName(args)` |
| `ir.table(fields)` | A table constructor; each entry is a bare node (positional), `{ key = "name", value = node }` or `{ key = node, value = node }` |
| `ir.localDecl(name, value)` | `local name = value` |
| `ir.exprStmt(expr)`, `ir.returnStmt(values)` | Statements |
| `ir.param(name, opts)`, `ir.functionDef(params, body, returnStmt)` | Parameters and anonymous functions |
| `ir.varargExpr()`, `ir.varargMethodCallClosure(object, method)` | `...` and a closure forwarding it |
| `ir.annotation(name, args)` | An annotation, such as a marker for another annotation or for reflection |

---

## The sandbox

An annotation runs with standard Lua (`string`, `table`, `math`, `pcall`, ...), the `ir` helpers, and `require` for the helper modules it imports. It has no `io`, `os`, `package`, `load`, `dofile` or `debug`: annotations cannot read your filesystem, fetch URLs, or escape the compile.

---

## Compiler builtins

A small set of annotations is recognised directly by the compiler. They don't go through annotation files, can't be redefined, and don't need an `annotations` entry in `nebra.toml`. They change compile-time behaviour rather than IR.

- `@side(client | server | shared)` - multiplayer-sandbox execution-side scoping for declarations. See [Sides](./sides.md) for the full mental model.
- `@reflectable` - records a declaration for [reflection](./reflection.md) under `mode = "annotated"`.
- `@overrideCtor("template")` - overrides how `new ClassName(args)` lowers to Lua. Applied to a `class` (or `declare class`) decl. The template is a format string with two placeholders:
    - `$class` - the resolved class identifier
    - `$args` - the comma-separated rendered arguments

  Example: nanos-world exposes every class as a global call (`Player(args)`, not `Player.new(args)`). The `nanos-world-types` package emits `@overrideCtor("$class($args)")` on every class so:

  ```lua
  -- source
  local p = new Player()
  -- generated Lua
  local p = Player()
  ```

  When the template has no `$args` placeholder, the compiler appends a default `(args)` call so the arguments are never silently dropped.

---

## When NOT to use an annotation

Annotations are powerful and run at compile time, which makes them tempting. Don't reach for them when:

- A plain function would do (e.g. `validate(myValue)` instead of `@validate myValue`).
- The behavior depends on runtime state - annotations only see compile-time IR.
- You need it just to suppress a warning - use the relevant `[rules]` knob instead.

Annotations are best for: registering symbols (routes, event handlers, entities), generating boilerplate, and runtime type guards.
