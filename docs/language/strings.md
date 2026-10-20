---
sidebar_position: 6
title: "Strings"
description: "String literal forms, escape sequences, interpolation and the configurable concat operator."
---

# Strings

## String Literals

Nebra supports all Lua string forms:

```nebra
local a = "double quoted"
local b = 'single quoted'
local c = [[long bracket string
spans multiple lines]]
local d = [==[nested brackets]==]
```

## String Interpolation

When `string_interpolation = true` in config, backtick strings support embedded expressions:

```nebra
local name = "Alice"
local age = 30
local msg = `Hello, {name}! You are {age} years old.`
```

Compiles to:

```lua
local msg = "Hello, " .. tostring(name) .. "! You are " .. tostring(age) .. " years old."
```

### Expression Interpolation

Any expression can be embedded:

```nebra
local result = `2 + 2 = {2 + 2}`
local info = `Length: {#items}, First: {items[1]}`
```

### Escaping

```nebra
local escaped = `Use \{braces\} literally`
local backtick = `Use \` for backticks`
```

## String Concat Operator

The `..` operator always works for concatenation:

```nebra
local greeting = "Hello" .. " " .. "World"
```

Between two numbers, `..` builds a [range](./control-flow.md#ranges) instead. Convert one side
with `tostring` to join two numbers as text.

With `concat_operator = "+"` in config, `+` also works:

```nebra
local greeting = "Hello" + " " + "World"
```

This compiles to a helper function that handles both string and number addition.
