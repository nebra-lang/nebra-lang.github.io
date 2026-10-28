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

Struct types are checked by their shape. A table fits a struct type when it holds every field the
type declares with a fitting value. A field typed `T?` may be left out, and fields beyond the declared
ones are allowed. That makes typed option tables work:

```nebra
function spawn(options: { x: number, z: number? }): nil
    print(options.x, options.z ?? 0)
end

spawn({ x = 1, z = 2 })
spawn({ x = 1 })              -- z is optional
spawn({ z = 2 })              -- error: x is missing
```

A table literal also fits an [interface](./interfaces.md#tables-as-interfaces) in the same way.

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

A function literal whose parameters have no annotation takes their types from the function type it is written for: a call argument, an annotated local, a field, an assignment target or a field of a table literal. The same goes for a function declared on a field, such as `function love.update(dt)`. In a generic call the type parameters are filled in from the other arguments first:

```nebra
local function on(name: string, callback: (number) -> nil): nil
    callback(1)
end

on("tick", function(delta)          -- delta: number
    print(delta + 1)
end)

local labels = map({ 1, 2 }, function(item)   -- map from Generics below, item: number
    return tostring(item)
end)                                           -- string[]
```

A parameter with an annotation keeps it, and a function literal without an expected type takes anything, as before.

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

## Literal Types

A string, number or boolean literal is a type that holds just that value, usually in a union:

```nebra
local mode: "fast" | "slow" = "fast"
local level: 1 | 2 | 3 = 2
local flag: true = true

mode = "medium"                     -- error: '"medium"' is not '"fast" | "slow"'

local function speedOf(mode: "fast" | "slow"): number
    return mode == "fast" and 2 or 1
end
```

A literal type is used like its base type, so `mode .. "!"`, `#mode` and `level + 1` work, and it fits wherever its base type does. A plain `string` does not fit a literal type, since it may hold any text. A literal keeps its literal type only where one is expected; `local name = "fast"` is still a `string`.

## `keyof` and Indexed Access Types

`keyof T` is the union of the names of the fields of a struct, class or interface, each as a string literal type. `T[K]` is the type of the field `K` of `T`, or the union of several fields for a union of names. Together with a type parameter they type an event map, where each event name has its own callback:

```nebra
interface AppEvents
    "user:login": (LoginEvent) -> nil
    "user:logout": () -> nil
    tick: (number) -> nil
end

declare function on<K extends keyof AppEvents>(name: K, callback: AppEvents[K]): nil

on("user:login", function(event)    -- event: LoginEvent
    print(event.userId)
end)
on("tick", function(delta)          -- delta: number
end)
on("user:logn", function() end)     -- error: expected '"user:login" | "user:logout" | "tick"'
```

The call fills in `K` with the literal type of the name it is given, so `AppEvents[K]` is the type of that one field. A field whose name is no identifier is written as a string, such as `"user:login"`, and read with an index: `events["user:login"]`. A table literal written for such a type uses string keys for those fields:

```nebra
local events: AppEvents = {
    ["user:login"] = function(event) print(event.userId) end,
    ["user:logout"] = function() end,
    tick = function(delta) end,
}
```

`keyof` of a type without fields, such as `keyof number`, and an indexed access naming a field the type does not have are errors. Mapped, template literal and conditional types are not supported.

## Utility Types

Four built-in types make a struct type from the fields of a struct, class or interface, so a second type does not have to be declared by hand:

| Type | Fields |
|---|---|
| `Partial<T>` | every field of `T`, made optional |
| `Required<T>` | every field of `T`, without `nil` |
| `Pick<T, K>` | the fields of `T` that `K` names |
| `Omit<T, K>` | every field of `T` except those `K` names |

`K` is a string literal type or a union of them, such as `"name" | "email"` or `keyof Timestamps`. Fields include inherited ones; methods, getters and operators are not part of the result.

```nebra
class User
    name: string = ""
    email: string = ""
    age: number = 0
end

local function update(user: User, changes: Partial<User>): nil
    user.name = changes.name ?? user.name
    user.age = changes.age ?? user.age
end

update(user, { age = 31 })              -- the other fields may be left out
update(user, {})
update(user, { age = "31" })            -- error: age is a number

local function contact(person: Pick<User, "name" | "email">): string
    return person.name .. " <" .. person.email .. ">"
end
```

The result is a struct type, checked by its shape like any other, so a table literal written for it takes its fields and callbacks. Used with a type parameter, such as `Partial<T>`, it stands for the fields of the type the call fills in; inside the body a type parameter bounded by a class stands for the fields of that class. A declaration of the same name, such as your own `class Partial<T>`, takes precedence. A utility type of a type without fields, and a `Pick` or `Omit` naming a field the type does not have, are errors.

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

Inside the body of a generic function, class, method or interface default, a type parameter stands for one type the caller chooses. Only a value of that type parameter fits it, and it fits only what its bounds fit:

```nebra
local function wrap<T>(value: T): T
    local copy: T = value       -- fine
    local wrong: T = 5          -- error: 5 is not a T
    return value
end

local function add<T>(a: T, b: T): T
    print(a + b)                -- error: T is not known to be a number
    return a
end
```

At a call site the type parameters stay open and are inferred from the arguments, so `wrap(5)` and `wrap("x")` both work, also as a recursive call inside `wrap` itself.

A generic class is written `Box<T>` and used as `Box<number>`. Naming it without type arguments, as `Box`, is an error in annotations as well as in `extends` and `implements`. Each instantiation is its own type whose members have the type arguments filled in, so what the methods of a `Box<number>` return is specialized just like a generic function's result. `new` infers the type arguments from the constructor's arguments:

```nebra
class Box<T>
    value: T

    constructor(value: T)
        self.value = value
    end

    function map<U>(f: (T) -> U): Box<U>
        return new Box(f(self.value))
    end
end

local box = new Box(1)                          -- Box<number>
local n: number = box.value
local label = box:map(function(x: number): string
    return tostring(x)
end)                                            -- Box<string>
local wrong: string = box.value                 -- error: number is not a string
```

Type arguments also pass through `extends` and `implements`, and through interfaces:

```nebra
interface Source<T>
    method next(): T?
end

class Stack<T> implements Source<T>
    -- ...
end

class Numbers extends Stack<number>
end

local source: Source<number> = new Numbers({ 1, 2 })    -- fine
local strings: Source<string> = new Numbers({ 1, 2 })   -- error
```

An instantiation fits another one of the same class when each of its type arguments fits the other's, so a `Box<number>` fits a `Box<number?>`, and a `Numbers` fits a `Stack<number>` and a `Source<number>`.

Generated declaration files keep type parameters, their bounds and the type arguments of `extends` and `implements`, so a package's generic classes and functions are just as specialized for the code that uses it.

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

A check against a generic class without type arguments, `value is Box`, matches every instantiation. A check against an instantiation, `value is Box<number>` or `value instanceof Box<number>`, also compares the type arguments. Lua values do not carry them, so when a build contains such a check, `new` records the type arguments of each generic object it creates in a weak side table, and the check reads them from there. Objects created by code that does not record them (such as plain Lua) only have their class checked. Type arguments that are type parameters, as in `value is Box<T>`, are not known at runtime, so such a check only tests the class and warns about it.

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
local onDone: (() -> number)? = nil     -- an optional function
local next: () -> number?               -- a function returning an optional number
```
