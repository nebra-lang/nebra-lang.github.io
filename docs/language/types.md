---
sidebar_position: 1
title: "Type System"
description: "Primitives, nullable and union types, arrays, maps, structs, tuples, generics, never, and the is/as operators."
---

# Type System

Nebra adds an optional, gradual type system to Lua. All type annotations are compile-time only and stripped from the generated Lua output.

## Primitive Types

| Type      | Description            |
|-----------|------------------------|
| `number`  | Any number (int/float) |
| `string`  | Text                   |
| `boolean` | `true` or `false`      |
| `nil`     | The nil value          |
| `any`     | Opt out of type checks |
| `void`    | No return value        |
| `function`| Any function           |
| `thread`  | A coroutine            |
| `userdata`| Opaque userdata        |
| `never`   | The call never returns |

```nebra
local count: number = 42
local name: string = "Nebra"
local active: boolean = true
```

## Nullable Types

Append `?` to make a type nullable (equivalent to `T | nil`):

```nebra
local name: string? = nil
local age: number? = 25
```

## Union Types

Use `|` to allow multiple types:

```nebra
local id: string | number = "abc"
local result: boolean | nil = true
```

## Array Types

Append `[]` for arrays:

```nebra
local nums: number[] = {1, 2, 3}
local matrix: number[][] = {{1, 2}, {3, 4}}
local names: string?[] = {"a", nil, "c"}  -- array of nullable strings
local data: number[]? = nil               -- nullable array
```

## Map Types

```nebra
local config: { [string]: number } = { timeout = 30, retries = 3 }
local lookup: { [number]: string } = { [1] = "one", [2] = "two" }
```

## Struct Types

Named fields with types:

```nebra
local point: { x: number, y: number } = { x = 10, y = 20 }
local user: { name: string, age: number, active: boolean }
```

### Meta Fields

```nebra
local mt: { meta __index: (any) -> any, value: number }
```

## Function Types

```nebra
local callback: (number, string) -> boolean
local handler: (string) -> void
local factory: () -> (number, string)       -- multi-return
```

## Tuple Types

For multi-return values:

```nebra
local result: (number, string) = getResult()
function multi(): (string, number, boolean)
    return "ok", 42, true
end
```

## Variadic Return Types

Use `...T` to return an arbitrary number of values of type `T` - the counterpart of a variadic
parameter. It is valid as a function return type or as the trailing element of a tuple return:

```nebra
function forward(arr: number[]): ...number
    return table.unpack(arr)          -- any number of numbers
end

function tagged(): (string, ...number)
    return "point", 10, 20, 30        -- a string followed by zero-or-more numbers
end

local a, b, c = forward({1, 2, 3})    -- each binds to a number
local tag, x, y = tagged()            -- tag: string, x/y: number
```

A `...T` return may yield zero values, so - like `void` - it is exempt from the "must return a
value" check. When captured by a single variable it collapses to `T`.

## Generics

Functions, methods, classes and interfaces can take type parameters:

```nebra
local function first<T>(items: T[]): T?
    return items[0]
end

local function map<T, U>(items: T[], f: (T) -> U): U[]
    local out: U[] = {}
    for i, item in ipairs(items) do
        out[i] = f(item)
    end
    return out
end
```

A call fills in the type parameters from its arguments, through lists, maps, tuples, `?` and the signature of a callback, so its result has the specialized type:

```nebra
local name = first({ "a", "b" })              -- string?
local lengths = map({ "a", "bb" }, function(s: string): number
    return #s
end)                                            -- number[]
```

When a type parameter is told by several arguments, it takes the wider of their types, or their union.

A bound limits what a type parameter takes and gives it the members of the bound: `extends` names a class, `implements` one or more interfaces.

```nebra
local function describe<T implements Named>(value: T): string
    return value:label()                        -- a method of Named
end

describe(new Tag())                             -- Tag implements Named
describe(5)                                     -- error: 5 does not
```

A generic class is written `Box<T>` and used as `Box<number>`. Its type arguments are checked where they are written but erased afterwards: a `Box<number>` is a `Box` to the checker, so what its methods return is not specialized the way a generic function's result is.

```nebra
class Box<T>
    value: T

    constructor(value: T)
        self.value = value
    end
end

local box: Box<number> = new Box(1)
```

## The `never` Type

`never` is the bottom type: the type of an expression that produces no value because control flow
does not come back. A function declared to return `never` always raises, exits the process, or
loops forever. The two standard library functions that behave this way are declared that way:

```nebra
declare function error(message: any, level: number?): never
-- and, inside the `os` library
function exit(code: any?, close: boolean?): never
```

You can declare your own:

```nebra
function panic(message: string): never
    error("panic: " .. message)
end
```

The compiler enforces the promise. A `never` function may not return a value and may not be able to
reach its own end:

```nebra
function bad(): never
    print("still here")     -- error: a function returning 'never' must not complete normally
end
```

### Unreachable code

Anything after a call that returns `never` can never run, so the compiler rejects it:

```nebra
error("stop")
print("unreachable")        -- error: code is unreachable
```

### Narrowing

Because a `never` branch cannot produce a value, it drops out of the type of the surrounding
expression. In `a or b`, reaching the result means `a` was truthy, so its `nil` is stripped:

```nebra
local ffi = require("ffi") or error("not luajit")
-- `require` succeeded - a failure would have gone to `error`, which never returns

local function name(id: number): string?
    -- ...
end

local n = name(1) or panic("no name")   -- n: string, not string?
```

The same holds for a guard clause whose body diverges - see
[Nilability & Optionals](./nilability.md#nil-narrowing-in-conditionals):

```nebra
local input: string? = read()
if input == nil then
    panic("no input")
end
print(#input)               -- input: string from here on
```

`never` also disappears from unions (`string | never` is `string`) and is assignable to every type,
while nothing is assignable to it.

## Type Check Expression (`is`)

Check a value's type at runtime:

```nebra
if x is number then
    print(x + 1)
end

if value is string then
    print(#value)
end
```

Compiles to `type(x) == "number"`.

## Type Predicates (custom guards)

A function whose return is written `param is Type` is a **type predicate**: it returns a
`boolean` at runtime, and where the call appears as an `if` condition the compiler narrows the
argument bound to `param`. This is the way to recover a precise type from `any` or untyped Lua
data (JSON, external APIs) - something `is`/`instanceof` can't do for arbitrary shapes.

```nebra
function isString(value: any): value is string
    return type(value) == "string"
end

function process(input: string | number)
    if isString(input) then
        print(#input)          -- input narrowed to string here
    else
        print(input + 1)        -- input narrowed to number here
    end
end
```

The predicate is erased at compile time - the guard is just a boolean-returning function. Like
`as`, it is a trust-me assertion: the compiler believes the predicate, so a wrong check narrows
to the wrong type. The named parameter must exist, and the guard must return on all paths.

## Type Cast Expression (`as`)

Assert a type at compile-time (no runtime check):

```nebra
local x: any = 42
local n: number = x as number
```

## Grouping

Use parentheses to group complex types:

```nebra
local arr: (string | number)[] = {"hello", 42}
```
