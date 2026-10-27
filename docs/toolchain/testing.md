---
sidebar_position: 3
title: "Testing"
description: "Writing tests with the built-in nebra:test framework and running them with nebra test."
---

# Testing

Nebra ships with a built-in test framework as the module `nebra:test`, plus the `nebra test` CLI command for discovery, execution, and reporting. No external testing library required.

```bash
nebra test                  # run every discovered test
nebra test "math"           # only tests whose full name contains "math"
nebra test --quiet           # suppress per-test output
```

---

## Writing tests

A test file is any `.neb` source that calls `test(...)` (and optionally `describe(...)`) at the top level. The framework is imported from the magic module `"nebra:test"`:

```lua
-- tests/math_test.neb
import { describe, test, expect } from "nebra:test"

describe("Math operations", function(): nil
    test("addition", function(): nil
        expect(2 + 2):toBe(4)
    end)

    test("near floats", function(): nil
        expect(0.1 + 0.2):toBeNear(0.3, 1e-9)
    end)

    test("string contains", function(): nil
        expect("hello world"):toContain("world")
    end)
end)

test("standalone test", function(): nil
    expect({1, 2, 3}):toHaveLength(3)
end)
```

The `nebra:test` module is resolved by the runtime - the consuming code doesn't need any `.lua` files on disk for it. At runtime, `nebra test` injects the implementation into `package.preload["nebra:test"]` before each test file is loaded.

---

## API

### Test registration

```lua
test(name: string, fn: () -> nil): nil          -- alias: it(...)
describe(name: string, fn: () -> nil): nil      -- group tests with a prefix
beforeEach(fn: () -> nil): nil                  -- run before each test in scope
afterEach(fn: () -> nil): nil                   -- run after each test in scope
skip(name: string, fn: () -> nil): nil          -- marker - fn is not invoked
```

Tests register **and run eagerly** at module load time - there is no "collect first, run later" phase. This means errors at module level surface immediately, and a `describe` group runs its tests in declaration order.

### Assertion API (`expect`)

```lua
expect(value):
    toBe(other)                       -- ==
    toEqual(other)                    -- deep equality (recurses into tables)
    toBeNear(other, eps?)             -- numeric tolerance (default eps = 1e-9)
    toBeTruthy()
    toBeFalsy()
    toBeNil()
    toBeDefined()                     -- value ~= nil
    toContain(needle)                 -- string substring or table member
    toMatch(pattern)                  -- Lua string pattern
    toHaveLength(n)                   -- #value == n
    toThrow(message?)                 -- value must be a callable that raises;
                                       -- if message given, the error string
                                       -- must contain it

    -- Negated forms:
    toNotBe(other)
    toNotEqual(other)
    toNotContain(needle)
```

A failed assertion raises an error that the runner catches per-test; subsequent assertions in the same test do not run.

---

## Discovery rules

`nebra test` discovers test files using two complementary mechanisms (both configurable in `nebra.toml`):

```toml
[test]
dirs = ["tests", "test"]       # walked recursively
patterns = ["_test.neb", ".test.neb"]
quiet = false
```

- Any `.neb` file inside one of `[test].dirs` is a test file.
- Any `.neb` file (anywhere under `[source]`) whose filename ends with one of `[test].patterns` is a test file.
- A directory inside `[test].dirs` that has its own `nebra.toml` is a separate project, such as a fixture, and its files are not test files. Directory links, hidden directories and `nebra_modules/` are not entered.

```
src/
├-- main.neb                ← regular source
├-- math.neb                ← regular source
└-- math_test.neb           ← test (suffix match)
tests/
├-- integration_test.neb    ← test (in tests/ dir)
└-- api/v2_test.neb         ← test (recursive in tests/)
```

`.d.neb` files are always excluded.

---

## Filtering

The positional argument to `nebra test` filters tests by their **full name** (substring match):

```bash
nebra test "Math > addition"      # exact dot-separated path
nebra test "Math"                 # any test inside a Math describe block
nebra test addition               # any test whose name contains "addition"
```

`describe` groups contribute their name as a prefix; nested groups join with ` > `. So `describe("Math operations", function() test("addition", ...) end)` produces the full name `Math operations > addition`.

The filter applies after discovery - tests that don't match are silently skipped (not counted as failures or skips). Test output and summary reflect only matched tests.

---

## Output

Default output mirrors the structure of the source:

```
-- /tmp/nebra-test-abc123/math_test.lua --
Math operations
  ✓ addition
  ✓ near floats
  ✓ string contains
✓ standalone test

-----------------------------------------
✓ all 4 test(s) passed
```

Failure output includes the assertion message and a stack trace:

```
Math operations
  ✓ addition
  ✗ subtraction
      expected 2 to be 3
      stack traceback:
          [C]: in function 'error'
          [string "nebra:test"]:142: in method 'toBe'
          .../math_test.lua:9: in function <.../math_test.lua:7>

-----------------------------------------
✗ 1 failed, 3 passed (4 total)

  • Math operations > subtraction
    /tmp/nebra-test-abc123/math_test.lua
```

`--quiet` removes the per-test ticks; the summary at the bottom is always shown.

---

## Exit codes

| Code | Meaning                                |
|------|----------------------------------------|
| 0    | All discovered tests passed            |
| 1    | One or more failures, or compile error |

So `nebra test` is CI-friendly without further configuration.

---

## Hooks (`beforeEach` / `afterEach`)

Hooks defined inside a `describe` block apply only to tests in that block (and its nested blocks). Hooks defined at the file top level apply to every test in the file.

```lua
describe("Database tests", function(): nil
    local conn: any

    beforeEach(function(): nil
        conn = connect()
    end)

    afterEach(function(): nil
        conn:close()
    end)

    test("inserts a row", function(): nil
        expect(conn:insert("foo")):toBeTruthy()
    end)
end)
```

A failure in a hook is reported as the test's failure, not a separate event.

---

## Integration with the package manager

Test-only dependencies belong in `[dev-dependencies]`:

```toml
[dev-dependencies]
test-helpers = "github:DasDarki/nebra-test-helpers@v0.3.0"
```

`nebra install --no-dev` (used in production CI before a binary build) skips them. `nebra install` (default) and `nebra test` (which respects `--no-dev`'s absence) include them.

---

## Comparison with the legacy adapter

The repository's [`test/`](https://github.com/nebra-lang/nebra/tree/master/test) project (164 tests covering the language itself) uses a thin adapter (`test/src/_runner.neb`) that re-exports `section / expect / expectEq / ...` on top of `nebra:test`. This was kept for source compatibility while the legacy `run()`-per-file structure was migrated. New code should use `nebra:test` directly.
