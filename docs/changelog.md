---
sidebar_position: 99
title: "Changelog"
description: "What changed in each Nebra release, newest first."
---

# Changelog

All notable changes to Nebra are listed here, newest first. Entries under **Breaking** can make code that compiled before report errors; they are listed first so you can check them before upgrading.

## Unreleased

### Breaking

- `try`, `catch` and `finally` are keywords and can no longer name a variable or function; they still work as member names, such as `promise:catch(...)`. ([#181](https://github.com/nebra-lang/nebra/issues/181))

### Added

- Completion puts what fits the expected type first: the members of an expected enum as the shorthand, then values and functions of that type, in an annotated local, an argument, an assignment, a `return`, a comparison, a `match` case and a field or element of a table literal. ([#211](https://github.com/nebra-lang/nebra/issues/211))
- `target = "luajit"` is accepted as another name for `"jit"`. ([#209](https://github.com/nebra-lang/nebra/issues/209))
- `try ... catch (err) ... finally ... end`. A `return` inside `try` or `catch` returns from the enclosing function, `try` without `catch` raises the error again after `finally`, and a bare `try ... end` ignores it. ([#181](https://github.com/nebra-lang/nebra/issues/181))
- A member of an enum may be written without the enum where a value of that enum is expected: `local color: Color = Red`, as an argument, in an assignment, a `return`, a comparison, a `match` case or a table field. ([#165](https://github.com/nebra-lang/nebra/issues/165))
- A class may declare several constructors with different parameter types. `new` and `super(...)` pick the one the arguments fit, at compile time where it is known and through `new` at runtime otherwise. ([#182](https://github.com/nebra-lang/nebra/issues/182))

### Fixed

- A half-written `case` of a `match` no longer drops the whole file from the language server, which left it without completion, hover and diagnostics. ([#211](https://github.com/nebra-lang/nebra/issues/211))
- A `nebra.toml` that cannot be read, such as one with an unknown `target`, stops the build instead of building with every setting at its default. ([#209](https://github.com/nebra-lang/nebra/issues/209))

## 0.5.15 (2026-10-27)

### Breaking

- The members of a generic class or interface now have the type arguments of the instantiation they are used through, so a mismatch such as `local name: string = new Box(1).value` is an error where the erased `T` let it pass before. ([#204](https://github.com/nebra-lang/nebra/issues/204))

### Added

- Generic classes and interfaces are fully substituted: each instantiation such as `Box<number>` has its type arguments filled into its fields, methods, getters, operators, base class and interfaces, `new` infers them from the constructor's arguments, and an instantiation fits another of the same class when its type arguments do. ([#204](https://github.com/nebra-lang/nebra/issues/204))
- `is` and `instanceof` against an instantiation, such as `value is Box<number>`, compare the type arguments at runtime. A generic class without type arguments, `value is Box`, matches every instantiation. ([#204](https://github.com/nebra-lang/nebra/issues/204))
- Generated declaration files keep type parameters with their bounds and the type arguments of `extends` and `implements`. ([#203](https://github.com/nebra-lang/nebra/issues/203))
- Literal types such as `"fast" | "slow"`, `1 | 2` and `true`, `keyof T` and indexed access types `T[K]`, so an event map types each callback by the name it is registered under. Interface and struct fields may be named by a string such as `"user:login"`. ([#206](https://github.com/nebra-lang/nebra/issues/206))
- A function literal without parameter annotations takes its parameter types from the function type it is passed or assigned to. ([#207](https://github.com/nebra-lang/nebra/issues/207))

### Fixed

- A generic method implementing a generic interface method is compared with it key by key, a union fits a union holding each of its members, and errors in code an annotation generates point at the annotation. ([#208](https://github.com/nebra-lang/nebra/issues/208))
- `nebra test` no longer hangs when a test directory contains a link back to the project, such as a fixture's `nebra_modules/` linking to the package it tests. The CLI walks directories like the language server does, without following directory links or entering `nebra_modules/`, and test discovery skips nested projects that have their own `nebra.toml`. ([#205](https://github.com/nebra-lang/nebra/issues/205))

## 0.5.14 (2026-10-27)

### Breaking

- A generic class or interface used without its type arguments, such as `Box` instead of `Box<number>`, is an error, in annotations as well as in `extends` and `implements`. ([#190](https://github.com/nebra-lang/nebra/issues/190))
- Inside the body of a generic function, class, method or interface default, a type parameter only accepts a value of itself and fits only what its bounds fit, so `local value: T = 0` or `a + b` on unbounded `T` values are errors. Call sites still infer type parameters as before. ([#177](https://github.com/nebra-lang/nebra/issues/177))
- Member names Nebra reserves are errors: the metamethods `operator` declares (`__add`, `__eq`, `__len`, ...), `__index`, `__newindex`, `__name`, `__interfaces`, `__metatable` and names starting with `__nebra`, `__get_`, `__set_`, `__op_` or `__ext_`. `__tostring`, `__close`, `__gc`, `__pairs` and `__call` stay allowed. ([#198](https://github.com/nebra-lang/nebra/issues/198), [#187](https://github.com/nebra-lang/nebra/issues/187))
- A class written in Nebra, its constructor and its methods cannot be replaced from outside (`A = ...`, `function A.new() end`). ([#201](https://github.com/nebra-lang/nebra/issues/201))
- A second constructor in a class is an error instead of silently replacing the first. ([#185](https://github.com/nebra-lang/nebra/issues/185))
- `super()` outside a derived class's constructor is an error instead of producing invalid Lua. ([#194](https://github.com/nebra-lang/nebra/issues/194))
- Two unrelated interface defaults for the same method must be resolved by overriding it in the class. ([#188](https://github.com/nebra-lang/nebra/issues/188))
- Operators, constructors and getters/setters cannot declare type parameters, and operators other than `()` cannot declare several return values. ([#199](https://github.com/nebra-lang/nebra/issues/199), [#200](https://github.com/nebra-lang/nebra/issues/200))
- A getter takes no parameters, a setter exactly one, and a property with only a setter cannot be read. ([#177](https://github.com/nebra-lang/nebra/issues/177))

### Added

- Code that runs while a file loads and reaches a class, function or `extend` block defined further down the same file is an error, including `class B extends A` above `A`, which left `B` without `A`'s methods. ([#154](https://github.com/nebra-lang/nebra/issues/154))
- Under `strict_nil`, an instance field whose type does not allow nil must be initialized by a default value or on every path of the constructor. ([#186](https://github.com/nebra-lang/nebra/issues/186))
- `sealed class` prevents a class from being extended. ([#180](https://github.com/nebra-lang/nebra/issues/180))
- `local constructor` limits creating instances to the class itself, for factories and singletons. ([#173](https://github.com/nebra-lang/nebra/issues/173))
- A warning for an extension that is never used because the extended type has a member of the same name. ([#189](https://github.com/nebra-lang/nebra/issues/189))

### Changed

- Expressions made only of literals are computed while compiling and written as their result, with Lua's integer and float semantics. ([#19](https://github.com/nebra-lang/nebra/issues/19))
- A derived class whose constructor only creates the base instance compiles to a single `return setmetatable(Base.new(...), Class)`, and `...` is forwarded only when the inherited constructor takes parameters. ([#195](https://github.com/nebra-lang/nebra/issues/195))
- A subclass copies only the metamethods its base classes actually declare; the copying loop is left out when there are none. ([#192](https://github.com/nebra-lang/nebra/issues/192))

### Fixed

- A subclass keeps `__tostring`, `__call`, `__close` and `__pairs` declared as methods on its base class; before, only operator metamethods were carried over. ([#192](https://github.com/nebra-lang/nebra/issues/192))
- The named varargs example in the documentation uses the named parameter. ([#169](https://github.com/nebra-lang/nebra/issues/169))
- Warnings of a successful build are printed. ([#202](https://github.com/nebra-lang/nebra/issues/202))
- A constructor ending in varargs accepts any number of further arguments. ([#197](https://github.com/nebra-lang/nebra/issues/197))
- Loop variables take the types the iterator returns, so `for _, v in next, values` no longer types `v` as possibly nil. ([#171](https://github.com/nebra-lang/nebra/issues/171))
- A table keyed by numbers, such as `{ [1] = n }`, fits an array type. ([#179](https://github.com/nebra-lang/nebra/issues/179))

## 0.5.13 (2026-10-25)

### Fixed

- Packages from a repository subdirectory are installed without the subdirectory's prefix and under a store path of their own, so their `nebra.toml` and dependencies are read again. ([#164](https://github.com/nebra-lang/nebra/issues/164))

## 0.5.12 (2026-10-25)

### Fixed

- A class operator picks the overload whose parameter takes the right operand, and reports operands no overload takes. ([#162](https://github.com/nebra-lang/nebra/issues/162))
- A class operator taking a string wins over string concatenation, in type checking and in the generated Lua. ([#163](https://github.com/nebra-lang/nebra/issues/163))

## 0.5.11 (2026-10-25)

### Added

- Index, index assignment and call operators (`operator []`, `operator []=`, `operator ()`) for classes and extensions. ([#158](https://github.com/nebra-lang/nebra/issues/158))

### Fixed

- An optional function type no longer shares its key with a function returning an optional value, which crashed the compiler. ([#160](https://github.com/nebra-lang/nebra/issues/160))
- Table literals fit struct types and interfaces by their fields; fields typed `T?` may be left out. ([#161](https://github.com/nebra-lang/nebra/issues/161))
- A subclass with getters but no constructor of its own keeps its getters and inherited methods. ([#159](https://github.com/nebra-lang/nebra/issues/159))

## 0.5.10 (2026-10-20)

### Breaking

- `..` between two numbers is an inclusive range instead of a concatenation; `1.` is no longer a number literal. ([#153](https://github.com/nebra-lang/nebra/issues/153))
- Extensions are local to their module; `export extend` shares them with importing modules. ([#149](https://github.com/nebra-lang/nebra/issues/149))

### Added

- Getters, operators and `export extend` for type extensions, and member access on number literals such as `1.hour`. ([#149](https://github.com/nebra-lang/nebra/issues/149))

### Fixed

- Overloads of the same arity are told apart by their parameter types, the known overload is called directly, and duplicate signatures are reported. ([#150](https://github.com/nebra-lang/nebra/issues/150), [#151](https://github.com/nebra-lang/nebra/issues/151), [#152](https://github.com/nebra-lang/nebra/issues/152))

## 0.5.9 (2026-10-17)

### Added

- `super:method()` calls the overridden method of the base class. ([#148](https://github.com/nebra-lang/nebra/issues/148))
- Type arguments of generic calls are inferred, and bounded type parameters get the members of their bounds. ([#145](https://github.com/nebra-lang/nebra/issues/145))
- `nebra publish`, `unpublish`, `scope` and `login` for the package registry. ([#135](https://github.com/nebra-lang/nebra/issues/135), [#136](https://github.com/nebra-lang/nebra/issues/136), [#137](https://github.com/nebra-lang/nebra/issues/137))
- `import_path = "root"` writes imports of own modules as their path from the output root. ([#133](https://github.com/nebra-lang/nebra/issues/133))
- Packages load through a runtime global, and `export { ... } from` re-exports names. ([#127](https://github.com/nebra-lang/nebra/issues/127))
- Type imports may form cycles, top-level classes and local functions are hoisted, and loops iterate values through their `iterator()` method. ([#124](https://github.com/nebra-lang/nebra/issues/124), [#125](https://github.com/nebra-lang/nebra/issues/125), [#126](https://github.com/nebra-lang/nebra/issues/126), [#130](https://github.com/nebra-lang/nebra/issues/130))
- `nebra create` from local setup scripts and template directories.

### Fixed

- `defer` works like Go: arguments are evaluated when reached, each loop pass defers its own call and deferred calls run on errors. ([#134](https://github.com/nebra-lang/nebra/issues/134))
- Local dependencies skip links, nested projects and build output when hashed and loaded. ([#146](https://github.com/nebra-lang/nebra/issues/146), [#147](https://github.com/nebra-lang/nebra/issues/147))
- Further fixes for map types, method scopes, deep annotation targets, keyword method names, syntax errors in imported modules, `while` narrowing, empty table arguments, `from` and `operator` as contextual keywords, untyped locals, class annotations and async errors. ([#122](https://github.com/nebra-lang/nebra/issues/122)–[#144](https://github.com/nebra-lang/nebra/issues/144))
