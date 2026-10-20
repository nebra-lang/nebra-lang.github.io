---
sidebar_position: 2
title: "Hello, Typed World"
description: "The smallest useful Nebra program: type annotations, doc comments, nil handling and what the compiler strips from the output."
---

# Hello, Typed World

This is the shortest example that shows something Lua cannot do on its own. It is a single file, it
runs with `nebra run`, and every line of it has a reason to exist.

## The program

```nebra title="src/main.neb"
--- Formats a person's display name.
---
--- Falls back to a placeholder when no name was supplied, so the caller never
--- has to deal with nil coming back out.
---@param name the raw name, possibly nil
---@return string a name that is always safe to print
function displayName(name: string?): string
    local raw: string = name ?? ""
    local trimmed: string, replacements: number = string.gsub(raw, "^%s*(.-)%s*$", "%1")
    if #trimmed == 0 then
        return "anonymous"
    end
    return trimmed
end

--- Greets everyone in a list, one line each.
function greetAll(names: string?[]): nil
    for _, entry in ipairs(names) do
        print("Hello, " .. displayName(entry) .. "!")
    end
end

greetAll({ "Ada", "  Grace  ", "", "Alan" })
```

```bash
nebra run
```

```
Hello, Ada!
Hello, Grace!
Hello, anonymous!
Hello, Alan!
```

## What is doing the work

**`name: string?`** says this parameter may be nil. That is not a comment, it is a contract. Calling
`displayName(42)` is a compile error, and so is doing `#name` before you have dealt with the nil.

**`??`** is nil coalescing. `name ?? ""` produces the string when there is one and `""` when there
is not, which is exactly the shape `string.gsub` wants.

**`local trimmed: string, replacements: number = string.gsub(...)`** destructures a multi-return.
`string.gsub` returns the new string *and* a replacement count, and Nebra knows that from its stdlib
declaration. Writing `local trimmed: string = string.gsub(...)` is an error, because the value on
the right is a `(string, number)` tuple and not a `string`. Lua would have silently discarded the
second value. Here you have to acknowledge it.

**`string?[]`** is an array whose elements may each be nil. Note where the `?` sits: it binds to the
element type. `string[]?` would instead mean "an array that may itself be nil". The suffixes read
left to right.

**`: nil`** as a return type means the function returns nothing. You can also write `: void`, an
alias for the same thing that reads better when you mean "no result" rather than "the nil value".

**The `---` comments** are doc comments in LuaCATS style. They show up on hover in the editor and
feed [`nebra docs`](../toolchain/doc-comments.md). Ordinary `--` comments do not.

## What the compiler emits

```lua title="out/main.lua"
function displayName(name)
	local raw = (function() local __v = (name); if __v ~= nil then return __v else return ("") end end)()
	local trimmed, replacements = string.gsub(raw, "^%s*(.-)%s*$", "%1")
	if #trimmed == 0 then
		return "anonymous"
	end
	return trimmed
end
function greetAll(names)
	for _, entry in ipairs(names) do
		print("Hello, " .. displayName(entry) .. "!")
	end
end
greetAll({
	"Ada",
	"  Grace  ",
	"",
	"Alan"
})
```

Every annotation is gone. There is no type table, no runtime check and nothing to require. This is
the property worth internalising early: the type system is a compile-time argument between you and
the compiler, and the runtime never hears about it.

The one construct that expands is `??`, which becomes a small immediately-invoked function. That is
deliberate: it evaluates the left side exactly once, which the usual `a and b or c` idiom cannot
guarantee when the left side is `false`.

:::note
This output was produced with `[reflection] mode = "none"` in `nebra.toml`. Reflection metadata is
emitted by default, which adds a registry block at the top of the file. See
[Reflection](../advanced/reflection.md) for what it is and when you want it.
:::

## A method-call gotcha

Lua lets you write `("  x  "):gsub(...)` because strings carry a metatable. Nebra does not model that
out of the box, so this is an error:

```nebra
local upper: string = string.upper(s)   -- correct
local upper: string = s:upper()         -- error: type 'string' has no method 'upper'
```

Call the library function directly, or add the method yourself with an
[`export extend string`](../language/classes.md#extensions) block if you want the receiver syntax across your codebase.

## Turning the safety up

Add this to `nebra.toml`:

```toml title="nebra.toml"
[rules]
strict_nil = true
```

Now the compiler refuses to let a possibly-nil value be used where a non-nil one is required. Drop
the `?? ""` and you get:

```
error[E2006]: Expression of type 'string | nil' is possibly nil. Use '?.' to access fields safely or check for nil first.
```

Strict-nil mode is off by default so that existing Lua compiles unchanged. Turning it on is the
single highest-value flag in `nebra.toml` for a codebase you intend to maintain.

## Next

- [Classes and interfaces](./classes-and-interfaces.md) puts structure around this.
- [Nilability](../language/nilability.md) covers `?.`, `!`, flow narrowing and the `never` type.
- [Type system](../language/types.md) is the full reference for what you can write after the colon.
