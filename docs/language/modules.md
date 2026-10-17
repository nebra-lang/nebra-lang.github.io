---
sidebar_position: 11
title: "Modules (Import / Export)"
description: "import and export, named, default, namespace and side-effect imports."
---

# Modules (Import / Export)

Nebra uses an ES-style module system that compiles to Lua's `require()`.

## Importing

### Named Import

```nebra
import { Vector2, Rect } from "engine/math"

local v = Vector2.new(1, 2)
```

Compiles to:

```lua
local __mod = require("engine/math")
local Vector2 = __mod.Vector2
local Rect = __mod.Rect
```

### Aliased Import

```nebra
import { Vector2 as Vec2, Rect as Box } from "engine/math"
```

### Default Import

```nebra
import Player from "entities/player"
```

### Namespace Import

```nebra
import * as utils from "lib/utils"

utils.debug()
utils.format("hello")
```

### Side-Effect Import

Runs the module without binding any names:

```nebra
import "polyfill"
```

### Type Import

Brings in names that are only used as types. Nothing is loaded at runtime, so two modules may import each other's types, which a regular import rejects as a cycle:

```nebra
-- User.neb
import type { House } from "./House"

export class User
    houses: House[] = {}
end
```

```nebra
-- House.neb
import type { User } from "./User"

export class House
    owner: User? = nil
end
```

Using a type-imported name as a value, as in `new House()`, is an error; import it without `type` there.

## Exporting

### Export Function

```nebra
export function calculate(x: number): number
    return x * 2
end
```

### Export Local Function

```nebra
export local function helper(): string
    return "help"
end
```

### Export Variable

```nebra
export local PI: number = 3.14159
export local mut counter: number = 0
```

### Export Enum

```nebra
export enum LogLevel
    Debug
    Info
    Warn
    Error
end
```

### Export Class

```nebra
export class Connection
    host: string
    port: number

    constructor(host: string, port: number)
        self.host = host
        self.port = port
    end
end
```

### Export Interface

```nebra
export interface Handler
    function handle(data: string): void
end
```

## Module Return

Exports compile to a return table at the end of the file:

```lua
-- Generated Lua
local function calculate(x)
    return x * 2
end

return {
    calculate = calculate
}
```

## Import Path Resolution

The import path is passed to the `import_statement` config option (default: `require(%s)`). Paths use forward slashes and no file extension:

```nebra
import { Foo } from "lib/foo"        -- require("lib/foo")
import Bar from "src/components/bar"  -- require("src/components/bar")
```
