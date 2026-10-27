---
sidebar_position: 9
title: "Classes & OOP"
description: "Constructors, inheritance, abstract, override, protected, static, getters and setters, operator overloading."
---

# Classes & OOP

Nebra provides TypeScript-like classes that compile to idiomatic Lua metatable patterns.

## Basic Class

```nebra
class Animal
    name: string
    age: number = 0

    constructor(name: string, age: number)
        self.name = name
        self.age = age
    end

    function speak(): string
        return self.name .. " makes a sound"
    end

    static function create(name: string): Animal
        return new Animal(name, 0)
    end
end
```

### Instantiation

```nebra
local a = new Animal("Rex", 5)
a:speak()                         -- instance method (colon syntax)
Animal.create("Buddy")            -- static method (dot syntax)
```

`new Animal(args)` compiles to `Animal.new(args)`.

> **Always use `new` for construction.** Nebra requires `new ClassName(args)` to
> create an instance - even when the underlying runtime expects a
> different call shape (e.g. `ClassName(args)` without a method qualifier).
> The [`@overrideCtor`](../advanced/annotations.md#compiler-builtins) builtin rewrites
> the **codegen** to match the runtime's convention, but the **source** must
> still write `new`. A bare `ClassName(args)` is parsed as a function call
> and rejected by the type checker (`expected 'function', but got 'class<…>'`).
> This makes the constructor call site visually distinct from every other
> function call and keeps the parser unambiguous.

### Lua Output

```lua
local Animal = {}
Animal.__index = Animal

function Animal.new(name, age)
    local self = setmetatable({}, Animal)
    self.name = name
    self.age = age
    return self
end

function Animal:speak()
    return self.name .. " makes a sound"
end

function Animal.create(name)
    return Animal.new(name, 0)
end
```

## Fields

```nebra
class Config
    -- Instance field (on every instance)
    timeout: number = 30

    -- Static field (on the class table itself)
    static version: string = "1.0"

    -- Private field (on every instance, reachable only inside this class)
    local internalCounter: number = 0

    -- Protected field (on the class table, accessible in subclasses)
    protected debugMode: boolean = false
end
```

## Methods

```nebra
class Service
    -- Instance method (uses colon syntax)
    function process(data: string): string
        return data
    end

    -- Static method (uses dot syntax)
    static function initialize(): void
        print("init")
    end

    -- Async method
    async function fetchData(url: string): string
        return await httpGet(url)
    end

    -- Private method (reachable only inside this class)
    local function helper(): void
        print("internal")
    end
end
```

> **Instance methods carry an implicit `self`.** Nebra models every non-static
> class method as if its first parameter were `self: ThisClass`, even though
> you don't write it. That mirrors Lua's runtime: `obj:method(args)` desugars
> to `obj.method(obj, args)`, and Nebra's codegen emits exactly that. Two
> practical consequences:
>
> - Instance methods must be called with **`obj:method(args)`** (colon).
>   Writing `obj.method(args)` is rejected with `ErrInstanceMethodNeedsColon`
>   because `.` would leave `self` as `nil` at runtime.
> - Static methods are stored separately and dispatched via **`Class.method(args)`**
>   (dot). Calling them with a colon is a type error.
>
> Interface methods are different - see the alert in the [Interfaces](./interfaces.md)
> doc.

## Inheritance

```nebra
class Dog extends Animal
    breed: string

    constructor(name: string, age: number, breed: string)
        super(name, age)         -- must call super in derived constructor
        self.breed = breed
    end

    override function speak(): string
        return "Woof!"
    end
end
```

### Lua Output (Inheritance)

```lua
local Dog = setmetatable({}, { __index = Animal })
Dog.__index = Dog

function Dog.new(name, age, breed)
    local self = Animal.new(name, age)
    setmetatable(self, Dog)
    self.breed = breed
    return self
end

function Dog:speak()
    return "Woof!"
end
```

### Super Calls

`super(args)` can only be used inside a derived class constructor. It must be the first statement;
anywhere else, including methods and interface defaults, it is a compile error. To call the parent's
version of a method, use `super:method(args)` (see [Override](#override)).

A class has exactly one constructor. Declaring a second one is an error; give parameters default
values or make them optional (`name: string?`) to allow several ways of constructing it. A
constructor may end in varargs (`constructor(name: string, ...)`), which take any number of
further arguments.

```nebra
class Cat extends Animal
    constructor(name: string)
        super(name, 0)
    end
end
```

## Override

Mark methods that override a parent method with `override`:

```nebra
class Circle extends Shape
    override function area(): number
        return 3.14 * self.radius ^ 2
    end
end
```

If you define a method that exists in a parent without `override`, the compiler emits a **warning** about shadowing. If you use `override` but no matching parent method exists, it's an **error**.

An override reaches the method it overrides with `super:method(args)`, which calls the base class's version on the same instance:

```nebra
class User extends Model
    override function onPrePersist(): nil
        super:onPrePersist()      -- the base class sets its timestamps
        self.name = string.lower(self.name)
    end
end
```

It compiles to `Model.onPrePersist(self)`, works with `await` and is only available in a method of a class that extends another.

## Abstract Classes

Abstract classes cannot be instantiated directly. They may contain abstract methods (signature only, no body):

```nebra
abstract class Shape
    abstract function area(): number
    abstract function perimeter(): number

    function describe(): string
        return `Area: {self:area()}, Perimeter: {self:perimeter()}`
    end
end

-- ERROR: Cannot instantiate abstract class
-- local s = new Shape()

class Rectangle extends Shape
    width: number
    height: number

    constructor(w: number, h: number)
        self.width = w
        self.height = h
    end

    override function area(): number
        return self.width * self.height
    end

    override function perimeter(): number
        return 2 * (self.width + self.height)
    end
end

local r = new Rectangle(10, 5)  -- OK
```

Abstract methods compile to error stubs:

```lua
function Shape:area()
    error("Abstract method 'area' must be implemented")
end
```

Non-abstract subclasses **must** implement all abstract methods, or the compiler reports an error.

## Private Members

Fields and methods declared `local` are private: they live on the instance like any other
member and are reached through `self`, but only the methods of the class declaring them may
use them - not subclasses, and not code outside the class.

```nebra
class Counter
    local count: number = 0

    local function bump(by: number): number
        self.count = self.count + by
        return self.count
    end

    function add(by: number): number
        return self:bump(by)          -- OK: inside Counter
    end
end

local counter = new Counter()
counter:add(2)                        -- OK
counter:bump(2)                       -- error: 'bump' is private to Counter
```

Like protected members, private members are compiled as regular members; the restriction is
checked at compile time only.

## Protected Members

Protected fields and methods are accessible within the class and its subclasses:

```nebra
class Base
    protected secret: string = "hidden"

    protected function validate(): boolean
        return true
    end
end

class Child extends Base
    function check(): boolean
        return self:validate()     -- OK: accessing protected from subclass
    end
end
```

Protected members are compiled as regular members on the class table (no runtime enforcement, compile-time checks only).

## Getters & Setters

```nebra
class Person
    _age: number = 0

    get age(): number
        return self._age
    end

    set age(value: number)
        if value >= 0 then
            self._age = value
        end
    end
end

local p = new Person()
p.age = 25           -- calls setter
print(p.age)         -- calls getter
```

A getter takes no parameters and a setter exactly one, the assigned value. A property with only a
setter can be assigned but not read; reading it is an error. Getters and setters cannot declare
type parameters.

### Lua Output (Accessors)

When a class has getters/setters, Nebra uses a proxy metatable:

```lua
local function __nebra_class_proxy(cls, parent)
    return {
        __index = function(t, k)
            local g = cls["__get_" .. k]
            if g then return g(t) end
            local v = cls[k]
            if v ~= nil then return v end
            if parent then
                g = parent["__get_" .. k]
                if g then return g(t) end
                return parent[k]
            end
        end,
        __newindex = function(t, k, v)
            local s = cls["__set_" .. k]
            if s then s(t, v) return end
            if cls["__get_" .. k] then
                error("Cannot set readonly property '" .. k .. "'")
            end
            rawset(t, k, v)
        end
    }
end
```

Read-only properties (getter without setter) throw an error on write.

## Implementing Interfaces

```nebra
class Sprite implements Drawable, Serializable
    override function draw(): void
        print("drawing")
    end

    override function serialize(): string
        return "{}"
    end
end
```

See [Interfaces](./interfaces.md).

## Operator Overloading

Classes can define how built-in operators behave on their instances using the
`operator` keyword. Each overload compiles to a Lua metamethod on the class
table, so operator dispatch happens through Lua's normal metatable mechanism.

```nebra
class Vector2
    x: number
    y: number

    constructor(x: number, y: number)
        self.x = x
        self.y = y
    end

    operator +(other: Vector2): Vector2
        return new Vector2(self.x + other.x, self.y + other.y)
    end

    operator -(other: Vector2): Vector2
        return new Vector2(self.x - other.x, self.y - other.y)
    end

    operator -(): Vector2
        return new Vector2(-self.x, -self.y)
    end

    operator ==(other: Vector2): boolean
        return self.x == other.x and self.y == other.y
    end
end

local a = new Vector2(1, 2)
local b = new Vector2(3, 4)
local sum = a + b      -- calls Vector2:__add
local neg = -a         -- calls Vector2:__unm
local eq  = a == b     -- calls Vector2:__eq
```

### Supported Operators

| Operator | Arity  | Metamethod  |
|----------|--------|-------------|
| `+`      | binary | `__add`     |
| `-`      | binary | `__sub`     |
| `-`      | unary  | `__unm`     |
| `*`      | binary | `__mul`     |
| `/`      | binary | `__div`     |
| `//`     | binary | `__idiv`    |
| `%`      | binary | `__mod`     |
| `^`      | binary | `__pow`     |
| `..`     | binary | `__concat`  |
| `==`     | binary | `__eq`      |
| `<`      | binary | `__lt`      |
| `<=`     | binary | `__le`      |
| `#`      | unary  | `__len`     |
| `&`      | binary | `__band`    |
| `\|`     | binary | `__bor`     |
| `~`      | binary | `__bxor`    |
| `~`      | unary  | `__bnot`    |
| `<<`     | binary | `__shl`     |
| `>>`     | binary | `__shr`     |
| `[]`     | key    | `__index`   |
| `[]=`    | key, value | `__newindex` |
| `()`     | any    | `__call`    |

Unary vs. binary `-` and `~` are disambiguated by parameter count: zero parameters means
the unary operator (`__unm`, `__bnot`); one parameter means the binary one (`__sub`, `__bxor`).
The bitwise operators need a target with bitwise operators, Lua 5.3 or later.

Inside an operator body, `self` refers to the left-hand operand (or the sole
operand for unary overloads). Binary overloads receive the right-hand operand
as the single parameter.

A binary operator can be declared several times for different right-hand operand
types, like any overloaded method. The compiler picks the one whose parameter takes
the right operand and types the result from it, so with `operator *(other: Quaternion): Quaternion`
and `operator *(other: Vector): Vector`, `rotation * rotation` is a `Quaternion` and
`rotation * direction` a `Vector`. A right operand no overload takes is an error. A class
operator also wins over string concatenation: although `+` concatenates strings by default
(`concat_operator`), a class whose `operator +` accepts a `string` handles `price + "0.03"` itself.

### Index and Call Operators

`operator []` and `operator []=` let an instance be indexed like a table, and
`operator ()` lets it be called like a function:

```nebra
class Store
    data: { [string]: number } = {}

    operator [](key: string): number
        return self.data[key] ?? 0
    end

    operator []=(key: string, value: number)
        self.data[key] = value
    end
end

class Greeter
    prefix: string = "hi"

    operator ()(who: string): string
        return self.prefix .. " " .. who
    end
end

local store = new Store()
store["hp"] = 100        -- calls operator []=
print(store["hp"])       -- calls operator [], 100

local greet = new Greeter()
print(greet("bob"))      -- calls operator (), "hi bob"
```

- `store[key]` is typed by the operator's return type, and `store[key] = value`
  checks the key and the value against its parameters. A class with `operator []`
  but no `operator []=` cannot be assigned to by index.
- The operators only handle keys that are not members of the class. Methods,
  getters and static members are found as usual, and a declared field such as
  `store.data` is read and written on the instance, so constructors and field
  defaults are never routed to the operators.
- The key reaches the operator as written. `index_base` only shifts the indexes
  of arrays.
- `operator ()` takes any parameters and can be overloaded like a method.
  Calling the class itself, as in `Greeter()`, stays an error: instances are
  created with `new`.

### Inheritance

Operator overloads are inherited: a subclass automatically picks up its base
class's operators unless it defines its own. Re-declaring an operator in a
subclass overrides the inherited one.

### Rules for Operators

- An operator returns a single value, since Lua passes on only the first value a
  metamethod returns. `operator ()` is the exception: a call hands on everything.
- Operators, constructors and getters/setters cannot declare type parameters;
  they are never called with type arguments.
- Operators are declared with `operator`, not as methods named after their
  metamethod. Member names Nebra reserves are an error: the metamethods
  `operator` declares (`__add`, `__eq`, `__len`, ...), `__index` and `__newindex`
  (use `operator []` and `operator []=`), `__name`, `__interfaces`, `__metatable`,
  and names starting with `__nebra`, `__get_`, `__set_`, `__op_` or `__ext_`.
- Metamethods without a syntax of their own stay available as methods, since the
  class table is its instances' metatable: `function __tostring(): string`,
  `__close`, `__gc`, `__pairs`, and `__call` as long as the class does not also
  declare `operator ()`.

## Replacing Classes

A class written in Nebra, its constructor and its methods cannot be replaced from
outside: `A = ...`, `A.new = ...`, `function A.new() end` and `function A:method() end`
for an existing method are errors. Static fields stay assignable, and declared classes,
which describe tables the runtime owns, may still be patched.

## Exported Classes

```nebra
export class User
    name: string
    email: string

    constructor(name: string, email: string)
        self.name = name
        self.email = email
    end
end
```

## Extensions

`extend Type ... end` adds methods, getters and operators to an existing type - a class, an
interface, or a built-in like `string`, `number`, `function` or `thread` - without touching its
original definition. Inside an extension `self` is the receiver.

```nebra
extend number
    function double(): number
        return self * 2
    end

    get hour(): number
        return self * 3600
    end

    get min(): number
        return self * 60
    end
end

extend string
    function shout(): string
        return string.upper(self) .. "!"
    end

    operator <<(n: number): string
        return string.sub(self, n + 1) .. string.sub(self, 1, n)
    end
end

print((21):double())        -- 42
print(1.hour + 2.min)       -- 3720
print(("hi"):shout())       -- HI!
print("hello" << 3)         -- lohel
```

A number literal can be read from directly: `1.hour` and `1.5.min` read the getters of `1` and
`1.5`. A decimal number always needs digits after its dot, so `1.` is not a number.

Every use lowers at compile time to a plain call of a local function, such as
`__ext_PrimitiveNumber_get_hour(1)`. Extensions therefore work on **every** type, including
`number` and `boolean`, with no runtime overhead and no metatable tricks. An extension on a base class
or an interface is visible on all its subclasses / implementors. A real member always wins over an
extension of the same name, and the compiler warns about an extension that is shadowed that way. Built-in methods are not reachable through `:` on a primitive, so an
extension on `string` calls `string.upper(self)` rather than `self:upper()`.

### Getters

`get name(): T` adds a read-only property. An extension cannot declare a setter.

### Operators

An extension may declare the same operators a class can, listed under
[Supported Operators](#supported-operators). Overloads of one operator differ by the type of their operand. The extension is used when the
left operand has the extended type and the right one fits its parameter.

An extension cannot replace what Lua already does: arithmetic and bitwise operators between numbers,
`..` between strings and numbers, comparing two numbers or two strings, `#` of strings and tables,
indexing strings and tables, calling functions, `==` on anything, and any operator a class declares
itself. These are compile errors. An extension's `[]`, `[]=` and `()` lower to calls like its other
operators: `v[0]` on an extended `Vec` calls the extension's `operator []` with `v` and `0`.

### Sharing extensions between modules

An `extend` block is local to its module. Write `export extend` to let other modules use it: a
module that imports the declaring one, with any form of `import`, may use its exported extensions.

```nebra
-- units.neb
export extend number
    get kb(): number
        return self * 1024
    end
end

-- main.neb
import "./units"

print(5.kb)                 -- 5120
```

Using an extension of a module you do not import, or one it does not export, is a compile error.
Exported extensions are also part of a package's generated declarations, so they reach other
packages the same way.
