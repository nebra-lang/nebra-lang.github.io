---
sidebar_position: 3
title: "Functions"
description: "Named, local and anonymous functions, default parameters, varargs, overloads and multi-return."
---

# Functions

## Named Functions

```nebra
function add(a: number, b: number): number
    return a + b
end
```

## Local Functions

```nebra
local function multiply(x: number, y: number): number
    return x * y
end
```

## Definition Order

The top-level functions, classes and `extend` blocks of a file are declared before its first
statement, so a function body can use one written further down. Each of them only exists once its
definition has run, though, since a file runs from top to bottom. Code that runs right away and
reaches a definition further down is therefore an error:

```nebra
local function makeShape(): Shape
    return new Shape()          -- fine: runs only when called
end

local early = makeShape()       -- error: calls makeShape before class Shape has run
print(1.hour)                   -- error: the extend block below has not run yet

class Circle extends Shape      -- error: Shape is defined further down
end

class Shape
end

extend number
    get hour(): number
        return self * 3600
    end
end

local late = makeShape()        -- fine
```

Function bodies, method bodies and closures run later and may use anything in the file; only
code that runs while the file loads, including static field defaults and the class a class
extends, has to come after what it uses.

## Method Syntax

```nebra
function obj:method(x: number): string
    return tostring(x)
end
```

## Anonymous Functions

```nebra
local square = function(x: number): number
    return x * x
end

table.sort(items, function(a, b) return a < b end)
```

Inside a table constructor a function field may carry its name directly - see
[Named Functions in tables](./tables.md#named-functions):

```nebra
local ops = { function add(a: number, b: number): number return a + b end }
```

## Default Parameters

```nebra
function greet(name: string, greeting: string = "Hello"): string
    return greeting .. ", " .. name
end

greet("Alice")              -- "Hello, Alice"
greet("Bob", "Hi")          -- "Hi, Bob"
```

Default parameters compile to `if param == nil then param = default end` checks at the start of the function body.

## Variadic Functions

### Standard Varargs

```nebra
function printf(fmt: string, ...)
    print(string.format(fmt, ...))
end
```

### Named Varargs

Give a name and type to the variadic parameter. Inside the function, the name holds the
arguments as an array (`number[]` here):

```nebra
function sum(...values: number): number
    local mut total = 0
    for _, v in ipairs(values) do
        total = total + v
    end
    return total
end

sum(1, 2, 3)   -- 6
```

`...` stays available as well, for passing the arguments on unchanged, as in `print(...)`.

## Overloading

Several functions may share a name as long as their parameters differ, in number or in type:

```nebra
function format(x: number): string
    return tostring(x)
end

function format(x: number, decimals: number): string
    return string.format("%." .. decimals .. "f", x)
end

function format(x: string): string
    return "\"" .. x .. "\""
end

function format(...: any): string
    return "(" .. select("#", ...) .. " values)"
end

format(3.14)        -- "3.14"
format(3.14, 1)     -- "3.1"
format("hi")        -- "\"hi\""
format(true, nil)   -- "(2 values)"
```

The compiler picks the overload whose parameters fit the arguments best. When it can tell which
one a call means, the call goes straight to that overload: each one is compiled to its own Lua
function, such as `format__number_number`. The plain name `format` is a dispatcher that picks the
overload at runtime from the number and the types of its arguments. It handles calls with
arguments typed `any`, functions passed around as values, and calls from other modules:

- Primitives are told apart by `type()`.
- Classes are told apart by their class, so an overload taking `Square` is tried before one
  taking the `Shape` it extends.
- Interfaces are recognised by the interfaces a class implements.
- A varargs overload is tried last and catches whatever the others do not.

Like in Java or C#, the compiler chooses by the type a value has where it is called. A `Square`
held in a variable typed `Shape` goes to the `Shape` overload. Only an argument typed `any` is
matched by its runtime value.

Two overloads with the same parameter types are an error, even if their return types differ,
since no call could choose between them. Class methods overload the same way, including methods
a subclass adds to or overrides from its base class.

## Multi-Return

```nebra
function divide(a: number, b: number): (number, number)
    return math.floor(a / b), a % b
end

local quotient, remainder = divide(10, 3)
```

## Return Statement

```nebra
function getValue(): number
    return 42
end

-- No return value
function doWork(): void
    print("working")
end

-- Never returns: always raises, exits, or loops forever
function panic(message: string): never
    error("panic: " .. message)
end
```

A `never` function may not return a value, and the compiler rejects it if any path can reach its
end. Code after a call to one is reported as unreachable. See
[The `never` Type](./types.md#the-never-type).
