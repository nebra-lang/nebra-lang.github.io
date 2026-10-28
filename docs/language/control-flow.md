---
sidebar_position: 4
title: "Control Flow"
description: "if, while, repeat, numeric and generic for, do blocks, break N, continue, goto and labels."
---

# Control Flow

## If / Elseif / Else

```nebra
if x > 0 then
    print("positive")
elseif x < 0 then
    print("negative")
else
    print("zero")
end
```

## While Loop

```nebra
while condition do
    -- body
end
```

## Repeat-Until Loop

```nebra
repeat
    -- body
until condition
```

## Numeric For

```nebra
for i = 1, 10 do
    print(i)
end

for i = 0, 100, 5 do  -- with step
    print(i)
end
```

### Ranges

`a..b` between two numbers is the range from `a` up to and including `b`. A `for ... in` over
a range compiles to the numeric `for` above, so it costs nothing:

```nebra
for i in 1..10 do
    print(i)
end
```

Used anywhere else, a range is a `number[]` holding each of its numbers:

```nebra
local steps = 1..5          -- { 1, 2, 3, 4, 5 }
```

`..` only builds a range when both operands are typed `number`. As soon as one of them is a
string, it concatenates as usual, so `1 .. "x"` is `"1x"` and `1 .. 2 .. "x"` is `"12x"`.

## Generic For

```nebra
for k, v in pairs(tbl) do
    print(k, v)
end

for i, v in ipairs(arr) do
    print(i, v)
end
```

The compiler performs iteration type checking: when iterating over typed collections, loop variables are inferred. Enums are also iterable.

A value whose class has an `iterator()` method is iterated through it: `for i, item in collection do` runs over what `collection:iterator()` returns. When `iterator()` declares the function it returns, the loop variables get that function's result types:

```nebra
class Collection<T>
    items: T[] = {}

    function iterator(): () -> (number?, T)
        local index = 0
        return function(): (number?, T)
            index = index + 1
            if index > #self.items then
                return nil, nil!
            end
            return index, self.items[index]
        end
    end
end

for i, item in collection do  -- item is a T
    print(i, item)
end
```

When the value's type is not known, such as `any`, the loop asks at runtime: a table with an `iterator` method is iterated through it, anything else, such as an iterator function, is used as it is. Calls such as `pairs(t)` are always used as they are.

## Do Block

Creates a new scope:

```nebra
do
    local temp = compute()
    -- temp is only visible here
end
```

## Break

```nebra
while true do
    if done then break end
end
```

## Defer

`defer` schedules a call for when the enclosing function exits, however it exits: at its end, through any `return`, or because it raises an error. It works like Go's `defer`:

```nebra
local function save(path: string): nil
    local file = open(path)
    defer file:close()          -- runs when save exits, even if write raises

    file:write(serialize())
end
```

- The function and its arguments are evaluated where the `defer` stands; the call runs later. `defer print(x)` prints `x` as it was then.
- Several defers run in reverse order, the last one first.
- A `defer` in a loop schedules one call per pass.
- A function's return values are computed first, then its defers run.
- When the function raises an error, its defers run and the error is raised again unchanged.

A block defers several statements. It is a closure, so it sees its variables as they are when it runs:

```nebra
local function measure(): nil
    local started = os.clock()
    defer do
        print("took", os.clock() - started)
    end
    work()
end
```

`defer` can be used in functions and methods, not at the top level of a module or in a constructor. On the Lua 5.1 target it cannot be used in an `async` function.

## Try, Catch & Finally

`try` runs a block and catches the error it raises. `catch (err)` runs when it does, with the
value the error was raised with, and `finally` runs afterwards either way:

```nebra
local function load(path: string): string
    try
        return readFile(path)
    catch (err)
        print("could not read " .. path .. ": " .. tostring(err))
        return ""
    finally
        closeHandles()
    end
end
```

- The name in `catch (err)` is optional; `catch` alone ignores the error value.
- Without `catch`, an error is raised again once `finally` has run.
- Without both `catch` and `finally`, `try ... end` simply ignores an error.
- An error raised inside `catch` is raised again after `finally`.

A `return` in the `try` or `catch` block returns from the enclosing function, after `finally` has
run. A function that returns from both the `try` and the `catch` block needs no return after them.

Each block compiles to a function run under `pcall`, so a few things cannot cross its edge: `break`,
`continue` and `goto` cannot leave a block, a loop inside it may use them freely, a `finally` block
cannot `return`, and `defer` cannot be used inside the blocks. `await` works inside `try` on Lua 5.2
and newer and on LuaJIT; Lua 5.1 cannot yield across `pcall`, so there it is an error.

## Labels & Goto

```nebra
::retry::
local ok = tryOperation()
if not ok then
    goto retry
end
```

Available on Lua 5.2+ and LuaJIT targets.
