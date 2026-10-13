---
sidebar_position: 10
title: "Interfaces"
description: "Type contracts, interface inheritance, default methods and implements."
---

# Interfaces

Interfaces define type contracts. They are compile-time only and emit **no Lua code**.

## `method` and `function` Members

An interface member is declared with either `method` or `function`, and the keyword decides
how it is called:

- **`method`** declares an instance method. It receives the value it is called on as `self`,
  so it is called with a colon: `value:area()`. Calling it with a dot is rejected with
  `ErrInterfaceMethodNeedsColon`, because `.` would not pass the receiver.
- **`function`** declares a plain function member, called with a dot: `value.describe()`. This
  is how module-shaped declarations are written, such as the stdlib's `declare interface
  StringLib` together with `declare string: StringLib`, whose members are called as
  `string.upper(s)`.

```nebra
interface Shape
    method area(): number
    function describe(): string
end

local function report(shape: Shape): nil
    print(shape:area())
    print(shape.describe())
end
```

In both cases the parameter list is the whole contract: `self` is never written in an
interface signature. A class implements a `method` member with an ordinary instance method.
`method` is a contextual keyword - it only has this meaning at the start of an interface
member and stays usable as a name everywhere else.

## Defining an Interface

```nebra
interface Printable
    method toString(): string
end

interface Drawable
    method draw(): void
    method hide(): void
    visible: boolean
end
```

## Interface Inheritance

Interfaces can extend other interfaces:

```nebra
interface Serializable
    method toJson(): string
end

interface Storable extends Serializable
    method save(): void
    method load(): void
end
```

## Implementing Interfaces

Classes declare which interfaces they implement:

```nebra
class Document implements Printable, Serializable
    content: string

    constructor(content: string)
        self.content = content
    end

    override function toString(): string
        return self.content
    end

    override function toJson(): string
        return '{"content":"' .. self.content .. '"}'
    end
end
```

The compiler checks that all interface methods and fields are present. Missing members produce an error:

```
Error: Class 'Document' does not implement interface member 'save' from 'Storable'
```

## Multiple Interfaces

```nebra
class Widget implements Drawable, Printable, Serializable
    -- must implement all methods from all three interfaces
end
```

## Exported Interfaces

```nebra
export interface Plugin
    method init(): void
    method destroy(): void
    name: string
end
```

## Default Methods

An interface method **with a body** is a *default*: implementing classes inherit it
automatically and only need to override it when they want different behavior. Methods
**without** a body remain abstract and must be implemented.

```nebra
interface Greetable
    name: string
    -- abstract: every implementer must provide this
    method fullName(): string
    -- default: implementers get this for free
    method greet(): string
        return "hi, I'm " .. self:fullName()
    end
end

class Person implements Greetable
    name: string
    constructor(name: string) self.name = name end
    function fullName(): string return self.name end
    -- no greet(): inherits the default
end

class Robot implements Greetable
    name: string
    constructor(name: string) self.name = name end
    function fullName(): string return "unit-" .. self.name end
    override function greet(): string          -- opt out of the default
        return "BEEP " .. self:fullName()
    end
end
```

Inside a default, `self` refers to the implementing instance, so a default may call other
(abstract or default) interface methods and read interface fields. Defaults declared on a
base interface are inherited through `extends`. At compile time the default body is copied
onto each implementing class that does not override it - no runtime library, no shared state.
