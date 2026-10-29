---
sidebar_position: 15
title: "Declaration Files (.d.neb)"
description: "Writing .d.neb files to type existing Lua code and third-party libraries."
---

# Declaration Files (.d.neb)

Declaration files provide type information for existing Lua code without any runtime output. They use the `.d.neb` extension.

## Declaring Functions

```nebra
declare function print(msg: string): void
declare function type(v: any): string
declare function tostring(v: any): string
declare function tonumber(s: string): number?
```

## Declaring Variables

```nebra
declare pi: number
declare version: string
declare config: { [string]: any }
```

## Declaring Modules

Group declarations for a Lua module:

```nebra
declare module "math"
    function abs(x: number): number
    function floor(x: number): number
    function ceil(x: number): number
    function sqrt(x: number): number
    function sin(x: number): number
    function cos(x: number): number
    pi: number
    huge: number
    maxinteger: number
end
```

## Declaring Enums

```nebra
declare enum ErrorCode
    Success: number
    NotFound: number
    Timeout: number
end
```

Declared enums have typed members but no values (type-only).

## Declaring Classes

```nebra
declare class Vector2
    x: number
    y: number

    constructor(x: number, y: number)
    function magnitude(): number
    function normalize(): Vector2
    static function zero(): Vector2
    static function one(): Vector2
end

declare abstract class Shape
    abstract function area(): number
    protected function validate(): boolean
end
```

Declared classes support all modifiers: `abstract`, `static`, `protected`, `override`.

## Declaring Interfaces

```nebra
declare interface Iterator
    function next(): any?
    function reset(): void
end
```

### Adding to a Declared Interface

A `declare interface` whose name is already declared as an interface adds its members to it, so a declaration file can extend what another one declares, such as the standard library's `MathLib` behind `math`:

```nebra
declare interface MathLib
    function Clamp(value: number, min: number, max: number): number
end

local clamped = math.Clamp(5, 0, 1)
```

Interfaces written in source code are declared once; a second one of the same name is an error.

## Declaring Built-in Types

`declare type` declares a type the compiler builds itself. It gives the type a place to be found, completed, hovered and documented, while what it stands for is worked out by the compiler. The standard library declares the [utility types](types.md#utility-types) this way:

```nebra
--- Every field of the struct, class or interface `T`, each made optional.
declare type Partial<T>
```

Only the compiler's own types can be declared like this. A declaration of the same name in your code, such as `interface Pick`, takes the place of the built-in one.

## Module Declarations with Classes

```nebra
declare module "game/entities"
    class Entity
        id: number
        name: string
        constructor(name: string)
        function update(dt: number): void
    end

    interface Renderable
        function render(): void
    end

    enum EntityType
        Player: number
        Enemy: number
        NPC: number
    end
end
```

## Auto-Generation

When `generate_declarations = true` in config, Nebra automatically generates `.d.neb` files from exported symbols:

```toml
generate_declarations = true
```

For a file with:

```nebra
export class Player
    name: string
    constructor(name: string)
    function attack(): void
end

export function createPlayer(name: string): Player
    return new Player(name)
end
```

Nebra generates:

```nebra
declare module "path/to/file"
    class Player
        name: string
        constructor(name: string)
        function attack(): void
    end
    declare function createPlayer(name: string): Player
end
```

## Loading Global Declarations

Point to declaration files in config:

```toml
globals = ["lib/std.d.neb", "lib/"]
```

- Individual `.d.neb` files are loaded directly
- Directories are scanned recursively for `.d.neb` files
- All declared symbols become available globally across the project
