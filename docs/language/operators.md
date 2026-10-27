---
sidebar_position: 5
title: "Operators"
description: "Arithmetic, comparison, logical, bitwise, increment and decrement, nil coalescing and optional chaining."
---

# Operators

## Standard Lua Operators

### Arithmetic

| Operator | Description      |
|----------|------------------|
| `+`      | Addition         |
| `-`      | Subtraction      |
| `*`      | Multiplication   |
| `/`      | Division         |
| `//`     | Floor division   |
| `%`      | Modulo           |
| `^`      | Exponentiation   |
| `-x`     | Unary negation   |

### Comparison

| Operator | Description      |
|----------|------------------|
| `==`     | Equal            |
| `~=`     | Not equal        |
| `<`      | Less than        |
| `>`      | Greater than     |
| `<=`     | Less or equal    |
| `>=`     | Greater or equal |

### Logical

| Operator | Description |
|----------|-------------|
| `and`    | Logical AND |
| `or`     | Logical OR  |
| `not`    | Logical NOT |

### String & Other

| Operator | Description       |
|----------|-------------------|
| `..`     | Concatenation, or a [range](./control-flow.md#ranges) between two numbers |
| `#`      | Length            |

### Bitwise (Lua 5.3+)

| Operator | Description  |
|----------|-------------|
| `&`      | Bitwise AND |
| `\|`     | Bitwise OR  |
| `~`      | Bitwise XOR |
| `~x`     | Bitwise NOT |
| `<<`     | Left shift  |
| `>>`     | Right shift |

## Alternative Boolean Operators

When `alt_boolean_operators = true` in config:

```nebra
a && b        -- same as: a and b
a || b        -- same as: a or b
!a            -- same as: not a
a != b        -- same as: a ~= b
```

These compile directly to their Lua equivalents.

## Increment / Decrement

Nebra adds C-style increment and decrement operators. Because `--` is a Lua comment, decrement uses `~~` instead:

```nebra
x++           -- post-increment (returns old value)
++x           -- pre-increment (returns new value)
x~~           -- post-decrement (returns old value)
~~x           -- pre-decrement (returns new value)
```

Works on any l-value:

```nebra
arr[i]++
obj.count~~
++matrix[x][y]
```

Compiles to runtime helper functions:

```lua
-- Post-increment example
local __old = x; x = __old + 1  -- returns __old
-- Pre-increment example
x = x + 1                        -- returns x
```

## Nil Coalescing (`??`)

Returns the left operand if it is not nil, otherwise the right operand:

```nebra
local name = user?.name ?? "Anonymous"
local port = config.port ?? 8080
```

Compiles to: `(function() local __v = left; if __v ~= nil then return __v end return right end)()`

## Non-Nil Assertion (`!`)

Asserts that a value is not nil. The type checker treats the result as non-nullable:

```nebra
local name: string? = getName()
print(name!)                   -- type: string (not string?)
```

The `!` postfix operator is a compile-time assertion. No runtime code is emitted.

## Optional Chaining (`?.`)

Safely access fields on a potentially nil value:

```nebra
local value = obj?.field       -- nil if obj is nil
local nested = a?.b?.c         -- chains safely
```

## Optional Call (`?()`)

Safely call a potentially nil function:

```nebra
local result = callback?()     -- nil if callback is nil
```

## Type Check (`is`)

Runtime type check:

```nebra
if x is number then
    print(x + 1)
end
```

See [Types](./types.md) for details.

## Type Cast (`as`)

Compile-time type assertion (no runtime effect):

```nebra
local n = value as number
```

## Operator Precedence (low to high)

1. `or`, `||`
2. `??`
3. `and`, `&&`
4. Comparison: `<`, `>`, `<=`, `>=`, `~=`, `==`, `!=`
5. `is`, `as`
6. `|` (bitwise OR)
7. `~` (bitwise XOR)
8. `&` (bitwise AND)
9. `<<`, `>>` (shifts)
10. `..` (concatenation, right-associative)
11. `+`, `-`
12. `*`, `/`, `//`, `%`
13. Postfix: `!`, `++`, `~~`
14. Unary: `not`, `!`, `#`, `-`, `~`, `await`, `++`, `~~`
15. `^` (exponentiation, right-associative)

## Configurable Concat Operator

With `concat_operator = "+"` in config, the `+` operator can be used for string concatenation:

```nebra
local msg = "Hello" + " " + "World"  -- compiles to .. in Lua
```

## Runtime Type Introspection

### `typeof`

Returns a string describing the runtime type of a value. For classes and enums the returned string is the declared name; for primitives it matches Lua's `type()` result.

```nebra
typeof 42            -- "number"
typeof "hi"          -- "string"
typeof myDog         -- "Dog"
typeof Colors.Red    -- "Colors" (when static type is known as enum)
```

When the static type is known at compile time (class or enum), the name is inlined as a string literal. Otherwise a runtime helper walks the metatable chain for a `__name` marker and falls back to `type()`.

### `instanceof`

Runtime check whether a value is an instance of the given class (including subclasses).

```nebra
if pet instanceof Dog then
    pet:bark()
end
```

Walks the metatable chain so both direct and inherited class matches return true. Returns `false` for non-table values or unknown classes.

## Constant Folding

An expression made only of literals is computed while compiling and written as its result, the way
Lua would compute it at runtime:

```nebra
print((1 + 2.0) * -0x3)    -- compiles to print((-9.0))
print(7 // 2, "a" .. 1)    -- compiles to print(3, "a1")
```

On targets with an integer subtype (Lua 5.3 and later), integers stay integers and floats stay
floats. Expressions whose result depends on the runtime are left as written: a division by zero,
results that are infinite or not a number, floats turned into text, and operators supplied by an
extension or a class.
