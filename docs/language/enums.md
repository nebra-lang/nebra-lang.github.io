---
sidebar_position: 8
title: "Enums"
description: "Named constants with auto-numbering or explicit string and number values."
---

# Enums

Enums define a set of named constants.

## Basic Enum

Values are auto-numbered starting from 0:

```nebra
enum Color
    Red
    Green
    Blue
end
```

Compiles to:

```lua
local Color = {Red = 0, Green = 1, Blue = 2}
```

## Explicit Values

```nebra
enum HttpStatus
    Ok = 200
    NotFound = 404
    ServerError = 500
end
```

## String Enums

```nebra
enum Direction
    Up = "up"
    Down = "down"
    Left = "left"
    Right = "right"
end
```

String enums compile to a bidirectional lookup table:

```lua
local Direction = {Up = "up", Down = "down", Left = "left", Right = "right"}
Direction["up"] = "Up"
Direction["down"] = "Down"
-- ...
```

## Usage

```nebra
local c = Color.Red
if c == Color.Green then
    print("Green!")
end
```

## Enums in Match

```nebra
match status
case HttpStatus.Ok then
    print("Success")
case HttpStatus.NotFound then
    print("Not found")
case _ then
    print("Other")
end
```

## Member Shorthand

Where a value of an enum type is expected, a member may be written without the enum:

```nebra
local color: Color = Green              -- Color.Green

local function paint(color: Color): nil
end

paint(Blue)                             -- an argument
color = Red                             -- an assignment to a typed target
print(color == Green)                   -- a comparison with an enum value

local function name(status: HttpStatus): string
    return match status
        case Ok then "success"          -- a match on an enum
        case NotFound then "not found"
        case _ then "other"
    end
end
```

The same goes for a `return` from a function returning the enum and a field of a table written for
a struct, class or interface type. The shorthand compiles to the member's value, so the enum does
not need to be imported where it is used; a member of an enum from a declaration file is read from
its table instead. A name that is declared in scope always means that declaration, and a name used
where no enum is expected, such as `local color = Green`, is reported as undeclared.

Completion offers the members first wherever such an enum is expected.

## Iteration

Enums are iterable:

```nebra
for name, value in pairs(Color) do
    print(name, value)
end
```

## Exported Enums

```nebra
export enum Severity
    Low
    Medium
    High
    Critical
end
```
