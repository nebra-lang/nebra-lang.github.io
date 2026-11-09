---
sidebar_position: 10.5
title: "Contracts"
description: "Strict interfaces that reject additional fields and check rules such as ranges, lengths and patterns."
---

# Contracts

A contract is an interface for plain tables that accepts exactly what it describes. A table with a field the contract does not have is a type error, and fields can carry rules that say which values are valid.

```nebra
contract Player
    @length(1, 16)
    @pattern("^%a+$")
    name: string

    @range(0, 150)
    @integer
    age: number

    nick: string?
end

local a: Player = { name = "Ann", age = 30 }              -- ok
local b: Player = { name = "Ann", age = "old" }           -- error: wrong type
local c: Player = { name = "Ann", age = 30, extra = 1 }   -- error: additional field
local d: Player = { name = "Ann", age = 200 }             -- error: must be between 0 and 150
```

Additional fields are rejected through a variable as well. Inline types such as `{ age: number }` and interfaces accept them, so existing code keeps working; a contract is what you write when you want the strict check.

## Rules

| Rule | Field type | Meaning |
|---|---|---|
| `@range(min, max)` | number | The value lies between `min` and `max`, both included. `@range(min = 0)` and `@range(max = 10)` set one side. |
| `@integer` | number | The value is a whole number. |
| `@length(min, max)` | string, list | The length lies between `min` and `max`. `@length(8)` is exactly 8; `@length(min = 1)` and `@length(max = 32)` set one side. |
| `@nonEmpty` | string, list | The value has content. |
| `@pattern("...")` | string | The value matches a [Lua pattern](https://www.lua.org/manual/5.4/manual.html#6.4.1). |

A field may carry several rules, and all of them have to hold. On an optional field they apply when it has a value. The length of a string counts bytes, like `#` in Lua.

Only these rules can be written on the fields of a contract, not your own annotations.

## When rules are checked

**At compile time**, for values written as literals in a table given to a contract: as the value of a variable, an argument, a return value, or a nested table.

A value the compiler cannot know is accepted:

```nebra
local a: Player = { name = "Ann", age = readAge() } -- no error, whatever readAge returns
```

**At runtime**, only when you ask with `is` or a type pattern in `match`. This is the check for data from outside, such as the network or a database. It tests the fields, their types, additional fields and every rule, and looks into nested contracts:

```nebra
local received: any = decode(message)

if received is Player then
    print(received.name)
end

match received
    case player: Player then greet(player)
    case _ then reject()
end
```

Everywhere else a contract costs nothing at runtime. The elements of a list are not tested at runtime, only that the field is a table.

## Extending

A contract can extend contracts and interfaces. A table for it has the fields of all of them:

```nebra
contract Named
    name: string
end

contract Employee extends Named
    salary: number
end

local e: Employee = { name = "Ann", salary = 1 }   -- ok
local n: Named = { name = "Ann", salary = 1 }      -- error: additional field
```

## Differences from interfaces

| | Interface | Contract |
|---|---|---|
| Additional fields | allowed | type error |
| Rules on fields | no | yes |
| Implemented by a class | yes | no, a class is more than its fields |
| `is` at runtime | the interfaces a class implements | fields, types and rules |

`contract` is only a keyword at the start of a declaration, so names such as `local contract = 1` keep working. It can be exported with `export contract` and described in declaration files with `declare contract`.
