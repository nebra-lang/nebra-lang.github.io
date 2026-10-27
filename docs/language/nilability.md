---
sidebar_position: 14
title: "Nilability & Optionals"
description: "Strict-nil mode, nil coalescing, the non-nil assertion, optional chaining and flow narrowing."
---

# Nilability & Optionals

Nebra tracks nullability in its type system and provides operators for safe nil handling.

## Nullable Types

Append `?` to a type to allow nil:

```nebra
local name: string? = nil        -- OK
local count: number = nil         -- ERROR: number is not nullable
```

## Strict Nil Mode

With `strict_nil = true` in config, the compiler requires explicit handling of nullable values before using them as non-nil:

```nebra
local name: string? = getName()
print(name)           -- WARNING: name might be nil
if name ~= nil then
    print(name)       -- OK: narrowed to string
end
```

Strict nil mode also requires every instance field whose type does not allow nil to be
initialized: by a default value, or by an assignment on every path of the constructor.
Otherwise the field would start as nil:

```nebra
class Account
    id: number                  -- error: never initialized
    owner: string               -- fine: assigned in the constructor
    note: string?               -- fine: may be nil
    balance: number = 0         -- fine: has a default

    constructor(owner: string)
        self.owner = owner
    end
end
```

An assignment in a method the constructor calls does not count; give such a field a default or
make it nullable.

## Non-Nil Assertion (`!`)

Assert that a value is not nil. The type checker treats the result as non-nullable:

```nebra
local name: string? = getName()
local safe: string = name!        -- assert non-nil
print(name!)                      -- use inline
```

This is a compile-time assertion only. No runtime code is emitted. If the value is nil at runtime, Lua will produce its normal nil errors.

## Optional Chaining (`?.`)

Access fields on a potentially nil object. Returns nil if the object is nil:

```nebra
local city = user?.address?.city
```

Compiles to:

```lua
local city
do
    local __t = user
    if __t ~= nil then
        __t = __t.address
        if __t ~= nil then
            city = __t.city
        end
    end
end
```

## Optional Call (`?()`)

Call a function only if it's not nil:

```nebra
local result = callback?()
local value = obj?.method?()
```

## Nil Coalescing (`??`)

Provide a default value when an expression is nil:

```nebra
local name = user?.name ?? "Anonymous"
local port = config.port ?? 8080
local list = getData() ?? {}
```

Right-associative: `a ?? b ?? c` means `a ?? (b ?? c)`.

## Nil Narrowing in Conditionals

The type checker narrows types based on nil checks:

```nebra
local x: string? = getInput()

if x ~= nil then
    -- x is narrowed to `string` here
    print(x .. "!")
end

if x == nil then
    -- x is `nil` here
    return
end
-- x is `string` after the nil guard
```

A guard branch counts as taken care of when it exits by any means - `return`, `break`, or a call to
a function that returns [`never`](./types.md#the-never-type):

```nebra
local y: string? = getInput()

if y == nil then
    error("no input")
end
print(#y)               -- y is `string` here

local z: string = getInput() or error("no input")
```

The narrowing lasts until the variable is assigned again, which restores its declared type:

```nebra
if y == nil then return end
-- y: string
y = nil                 -- allowed: the assignment ends the narrowing
```

## Configuration

```toml
[rules]
strict_nil = false    # true: require explicit nil handling
```
