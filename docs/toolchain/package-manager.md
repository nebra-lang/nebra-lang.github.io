---
sidebar_position: 2
title: "Package Manager"
description: "Dependency specs, the install pipeline, the registry, lockfiles and lifecycle scripts."
---

# Package Manager

Nebra ships with a built-in package manager modeled after npm / cargo, with git as the source of truth. Dependencies are declared in `nebra.toml`, fetched from git, and linked into a local `nebra_modules/` folder. The [package registry](#the-package-registry) maps names such as `@nebra-lang/nanos-world` to git repositories, so packages can be added by name; there is no package server, the registry is a GitHub repository and every version is a git tag.

---

## Declaring dependencies

`nebra.toml` has three dependency tables:

```toml
[dependencies]
# From the registry, by name
nanos-world-types = "@nebra-lang/nanos-world@^1"

# Straight from git
nebra-strings = "github:DasDarki/nebra-strings@v1.2.0"

# Inline table - same fields as a [packages.<name>] block
lua-math = { git = "https://example.com/lua-math.git", tag = "v0.5.0" }

# Local checkout for development
my-utils = "file:../my-utils"

[dev-dependencies]
# Only installed when --no-dev is not set
nebra-bench = "github:DasDarki/nebra-bench@v0.3.0"

[peer-dependencies]
# Required by libraries, not auto-installed
nebra-core = ">=1.0.0"
```

---

## Specifier formats

| Format                                | Resolves to                                          |
|---------------------------------------|------------------------------------------------------|
| `@scope/name`                          | A package of the registry, its default branch        |
| `@scope/name@^1`                       | A package of the registry at a tag or version range  |
| `name[@range]`                         | An unscoped name the registry still knows, such as `nanos-world` |
| `github:owner/repo`                    | `https://github.com/owner/repo.git`                  |
| `github:owner/repo@v1`                 | `https://github.com/owner/repo.git#v1`               |
| `github:owner/repo/sub/dir@v1`         | One package inside a monorepo (sparse-checked subdir)|
| `gh:owner/repo[@ref]`                  | Shorthand for `github:...`                           |
| `https://example.com/foo.git`          | Direct git URL, default branch                       |
| `https://example.com/foo.git#v1.0`     | Direct git URL, specific ref                         |
| `file:../local-path`                   | Symlink / copy of a local directory                  |
| `{ git = "...", tag = "v1" }`          | Inline table form, same fields as the spec strings   |
| `{ git = "...", branch = "main" }`     | Branch instead of tag                                |
| `{ git = "...", rev = "abc1234" }`     | Specific commit SHA                                  |
| `{ git = "...", subdir = "pkgs/x" }`   | Package living in a subdirectory of the repo         |

A ref can be a branch, a tag, or a commit SHA. Tags are preferred for reproducibility.

### Monorepos (multiple packages in one repo)

When a repository hosts more than one Nebra package, name the package's subdirectory so the
resolver knows which one to add. Any path segments after `owner/repo` in a host shortcut
become the in-repo subdirectory, or you can spell it out with the `subdir` key:

```bash
nebra add github:owner/monorepo/packages/math@v1
```

```toml
[dependencies]
math = { git = "https://github.com/owner/monorepo.git", tag = "v1", subdir = "packages/math" }
```

Nebra sparse-checks only that subdirectory into the store and reads **its** `nebra.toml` to
determine the dependency name and its own transitive dependencies. The same repo can be
added multiple times pointing at different subdirectories.

---

## Adding / removing

```bash
nebra add @nebra-lang/nanos-world
nebra add github:DasDarki/nebra-strings@v1.2.0
nebra add github:owner/lua-math@v0.5.0 --dev
nebra remove lua-math
```

`nebra add` mutates `nebra.toml` via a roundtrip TOML parser (Tomlyn `DocumentSyntax`) - **your formatting and comments are preserved**. If the resulting spec is ambiguous with an inline-table form already present, the inline table is upgraded; otherwise a string form is appended.

After mutation, the installer runs - you get the new package in `nebra_modules/` immediately.

---

## The install pipeline

`nebra install` (or `nebra add`'s implicit install) runs five stages:

1. **Resolve** - every spec is normalized into a `(name, gitUrl, ref)` triple. Registry names are looked up in the registry's index (cached for a day at `~/.nebra/cache/registry/index.json`, refresh with `nebra pm refresh-registry`).
2. **Fetch** - each unique `(gitUrl, ref)` is fetched into a bare-clone cache under `~/.nebra/cache/git/`. Subsequent fetches reuse the cache. Use `--no-cache` to force a fresh clone.
3. **Materialize** - each package is checked out into the **store** at `~/.nebra/store/<host>/<owner>/<repo>/<commit>/`, shared by every project.
4. **Link** - the store directory is linked into `<project>/nebra_modules/<name>/`. On Linux/macOS this is a symlink; on Windows it's a junction (or a copy on filesystems that don't support junctions).
5. **Transitive resolve** - the freshly installed package's own `nebra.toml` is read; its `[dependencies]` are processed the same way. Versions are de-duplicated by `(name, resolved-ref)`.

Output:

```
$ nebra install
  resolved nebra-strings -> github:DasDarki/nebra-strings@v1.2.0
  resolved lua-math    -> file:../lua-math@
Installed 2 package(s) into nebra_modules/.
```

---

## The lockfile (`nebra.lock`)

After every successful install, Nebra writes `nebra.lock` next to `nebra.toml`. It pins every transitively-resolved `(name, git, ref, commit, integrity)` tuple. Commit it to source control - `nebra install --frozen` (CI mode) refuses to drift from the lockfile.

```toml
# nebra.lock v1
version = 1

[[package]]
name = "nebra-strings"
git = "https://github.com/DasDarki/nebra-strings.git"
ref = "v1.2.0"
commit = "abcdef1234..."
integrity = "sha256-..."

[[package]]
name = "lua-math"
git = "file:/path/to/lua-math"
```

---

## How dependencies are seen by the compiler

After install, the compiler discovery (Phase 4) walks `nebra_modules/`:

- Every `<pkg>/init.neb` or `<pkg>/init.lua` becomes the entry for `import { x } from "<pkg>"`.
- Every `<pkg>/<sub>.neb` (or `.lua`) becomes addressable as `import { x } from "<pkg>/<sub>"`.
- `.d.neb` files in any installed package are loaded as type-only declarations - their types become available to the type checker, but no Lua code is generated for them.
- Each package's own `[scripts]` annotations are loaded (provided you used `--allow-scripts`).

This means a pure-Lua package can ship without any Nebra source - just a `.lua` entry point and an `.d.neb` next to it (the [`examples/lua-math/`](https://github.com/nebra-lang/nebra/tree/master/examples/lua-math) example shows the pattern).

---

## Lifecycle scripts

A dependency can declare scripts in its own `nebra.toml`:

```toml
# In nebra_modules/foo/nebra.toml
[scripts]
install = ["echo 'building native helpers...'", "make native"]
postinstall = ["./scripts/copy-assets.sh"]
```

These are **disabled by default** - Nebra does not run arbitrary code from dependencies unless you opt in:

```bash
nebra install --allow-scripts            # allow all
nebra install --allow-scripts=foo,bar    # allow only specific packages
```

This mirrors npm's `--ignore-scripts` default-flipped to safe-by-default. Trusted internal mirrors can set `[install].allow_scripts = true` in the project's own `nebra.toml` to suppress the per-invocation flag.

---

## The package registry

The registry is the GitHub repository [`nebra-lang/pm-registry`](https://github.com/nebra-lang/pm-registry). It maps scoped names to public repositories; the versions are those repositories' git tags, so a package is published once and released by tagging.

### Type declarations for games and frameworks

| Package | For | Scaffold a project |
|---|---|---|
| `@nebra-lang/nanos-world` | [nanos world](https://nanos.world) | `nebra create https://raw.githubusercontent.com/nebra-lang/nanos-world-types/main/setup.lua my-package` |
| `@nebra-lang/love2d` | [LÖVE](https://love2d.org) | `nebra create https://raw.githubusercontent.com/nebra-lang/love2d-types/main/setup.lua my-game` |
| `@nebra-lang/gmod` | [Garry's Mod](https://gmod.facepunch.com) | `nebra create https://raw.githubusercontent.com/nebra-lang/gmod-types/main/setup.lua my-addon` |

Each declares the whole API of its target, generated from its official documentation, with typed callbacks and events where the target has them. The setup scripts write a `nebra.toml` with the right target, folder layout and sides.

The CLI caches the index for a day; a name it does not have refreshes it, so a package is found as soon as it is published.

### Publishing your own package

1. Have a `nebra.toml` with a `name` in a public GitHub repository, at its root or in a directory of a monorepo.
2. Either ship `.neb` sources (run through the consumer's compiler) or pre-built `.lua` + `.d.neb` (faster install, no consumer needs to recompile yours).
3. Log in once and publish:

```bash
nebra login                          # GitHub device flow in the browser
nebra publish --description "..."    # as @<your account>/<name>
```

4. Tag releases as `vX.Y.Z`. Users add the package with `nebra add @you/your-lib@^1`.

`nebra publish` sends the request as an issue to the registry, signed by your GitHub login, and prints the answer of the workflow that carries it out. Without a login it opens the prepared issue in your browser. Packages can also be added straight from git without the registry: `nebra add github:you/your-lib@v1.2.0`.

### Scopes

Every package lives in a scope, `@scope/name`, and a scope belongs to GitHub accounts:

- **Your account's scope**, `@<your login>`, is claimed by the first package you publish to it.
- **An organization's scope** is used by its *public* members, or from one of the organization's repositories whose `nebra.toml` names the package with the scope, such as `name = "@my-org/core"`. Only those who can write to the organization's repositories can do that, so it works for private members too.
- Owners can let other accounts in: `nebra scope add-owner my-org alice`.

`nebra unpublish` removes a package again. Older unscoped names such as `nanos-world` keep working and resolve to their scoped package.

---

## Caches & file locations

| Path                                  | Purpose                                              |
|---------------------------------------|------------------------------------------------------|
| `~/.nebra/cache/git/`                 | Bare-clone cache, deduplicated per git URL           |
| `~/.nebra/store/<host>/<owner>/<repo>/<commit>/` | Materialized checkout, shared across projects |
| `~/.nebra/cache/registry/index.json`  | Registry index cache (refreshed by `nebra pm refresh-registry`) |
| `~/.nebra/credentials.json`           | The GitHub login of `nebra login`, readable only by you |
| `<project>/nebra_modules/<name>/`       | Link to the store; what the compiler reads from      |
| `<project>/nebra.lock`                  | Pinned resolution; commit this                       |

`nebra pm prune` resets the caches and the store; `NEBRA_HOME` moves `~/.nebra` elsewhere.
