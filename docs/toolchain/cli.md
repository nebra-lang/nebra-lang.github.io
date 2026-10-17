---
sidebar_position: 1
title: "CLI Reference"
description: "Every nebra command and flag: build, watch, run, init, create, install, docs, test, compile, repl and lps."
---

# CLI Reference

The `nebra` binary is the only tool you need: compiler, runner, REPL, test runner, package manager, language server, docs generator, and standalone-binary bundler - all behind subcommands.

```
nebra <command> [args]
```

Run `nebra help` for a one-screen summary. The sections below are the complete reference.

---

## `nebra init`

Scaffolds a new Nebra project in the current directory. Creates:

- `nebra.toml` with sensible defaults (`target = "5.4"`, `source = "src"`, `output = "out"`)
- `src/` and `out/` directories
- `.gitignore` with `out/` and `nebra_modules/` pre-listed

Refuses to overwrite an existing `nebra.toml`.

```bash
mkdir my-app && cd my-app
nebra init
```

---

## `nebra create <spec> [dir]`

Scaffolds a project from a template. The spec can be:

- A git URL: `https://github.com/owner/template.git`, `git@github.com:owner/template.git`
- A GitHub shortcut: `gh:owner/template`, `github:owner/template@v1`
- A direct URL to a `setup.neb` file that handles prompting and file generation
- A local `setup.neb` or `setup.lua`, such as one you are writing, run as a downloaded one would be
- A local template directory, copied or set up through its own `setup.neb` / `setup.lua`

| Flag           | Description                                              |
|----------------|----------------------------------------------------------|
| `--skip-setup` | Clone the template but don't run its `setup.neb`         |
| `--offline`    | Use only the cached registry; don't fetch from network    |
| `--no-cache`   | Bypass the local git cache for this run                  |

```bash
nebra create gh:DasDarki/nebra-app-template my-game
nebra create https://example.com/setup.neb
nebra create ../my-templates/setup.lua my-game
```

---

## `nebra build [files...]`

Compiles the project (no args) or specific `.neb` files (with args).

**Project mode** - reads `nebra.toml`, scans `<source>/**/*.neb`, writes `<output>/**/*.lua`. The configured `entry`, `target`, code section, mangle section, and rules section all apply.

**File mode** - compile the given files plus an optional `nebra.toml` (auto-detected). Output lands in the project's `<output>` dir (or `out/` if no config). Useful for one-off compilation.

```bash
nebra build                          # project mode
nebra build src/foo.neb src/bar.neb  # file mode
nebra build nebra.toml src/main.neb    # explicit config
```

Pre/post-build scripts from `[scripts]` run before/after compilation.

---

## `nebra watch`

Watches `<source>/` recursively and recompiles the whole project whenever a `*.neb` file
is created, changed, deleted, or renamed. File-system events are debounced so a burst of
saves triggers a single rebuild. Compile errors are printed and the watcher keeps running;
press `Ctrl+C` (or send `SIGTERM`) to stop.

Unlike `nebra build`, the watch loop does **not** run `[scripts]` pre/post-build hooks - it
only recompiles, so side-effecting hooks don't fire on every save. Each rebuild reloads
`nebra.toml`, so changes to the output path or compiler options are picked up live.

```bash
nebra watch                  # rebuild on every change
nebra watch --debounce 150   # shorten the debounce window (ms, default 300)
```

---

## `nebra run [files...] [-- args...]`

Compiles **and** executes via the embedded Lua 5.4 interpreter (KeraLua). State on the target machine is irrelevant - the binary brings its own Lua.

Same file vs project mode as `nebra build`. Everything after `--` is passed to the running Lua script as the global `arg` table.

```bash
nebra run                       # project mode, entry from nebra.toml
nebra run -- --verbose foo bar  # forward args to script

nebra run src/script.neb        # one-off file
```

The `package.path` is augmented with the compiled output dir and `nebra_modules/`, so cross-module requires Just Work. Exit code mirrors the Lua chunk's success.

---

## `nebra test [filter]`

Discovers and runs unit / integration tests.

**Discovery**: a file is a test iff
- It lives under any directory listed in `[test].dirs` (default: `tests/`, `test/`), recursively; or
- Its filename ends with one of `[test].patterns` (default: `_test.neb`, `.test.neb`).

The optional `[filter]` argument matches against the full test name (substring; case-sensitive). Only matching tests run; the others are skipped silently.

| Flag             | Description                                  |
|------------------|----------------------------------------------|
| `--quiet`, `-q`  | Suppress per-test tick output; show summary only |

```bash
nebra test                  # run every discovered test
nebra test "match"          # only tests whose name contains "match"
nebra test --quiet
```

Exit code is 0 on all-pass, 1 on any failure or compile error. See [Testing](./testing.md) for the assertion API and `describe`/`test` semantics.

---

## `nebra repl`

Starts an interactive Nebra session.

```text
nebra> 2 + 2
4
nebra> function greet(n: string): string
...>     return "hi " .. n
...> end
nebra> greet("world")
hi world
```

The runtime persists across inputs - globals and top-level `function`s survive. Top-level `local`s do not (Lua semantics; use `name = ...` without `local` to get a global).

REPL commands (start with `:`):

| Command          | Effect                                        |
|------------------|-----------------------------------------------|
| `:help`, `:h`    | Show inline help                              |
| `:quit`, `:q`    | Exit (also `Ctrl+D`)                         |
| `:clear`         | Clear the screen                              |
| `:reset`         | Drop all globals (fresh runtime)              |
| `:load <path>`   | Read a `.neb` file and evaluate it in-session |

Multi-line input is detected automatically - the REPL keeps prompting (`...>`) until brackets and blocks balance. See [REPL](./repl.md) for details.

---

## `nebra compile`

Bundles the project into a **single self-contained native executable**. Embeds all compiled Lua, the Lua 5.4 interpreter, and the Nebra stdlib bindings. The result runs on a machine with no Lua and no .NET.

| Flag             | Default                              | Description                                              |
|------------------|--------------------------------------|----------------------------------------------------------|
| `--out <path>`   | `./<name>` (`.exe` on Windows)       | Output binary path                                       |
| `--name <s>`     | `config.Name` or entry basename       | Application name (also the AssemblyName)                 |
| `--target <rid>` | current RID                          | .NET RID, e.g. `linux-arm64`, `win-x64`, `osx-arm64`     |
| `--aot`          | off                                  | Use Native AOT (experimental - smaller binary, faster start; needs clang/lld on Linux, MSVC on Windows) |
| `--keep-build`   | off                                  | Don't delete the temporary launcher project on success    |

Bundle process (5 steps, all logged to stdout):

1. Compile the project (same pipeline as `nebra build`).
2. Scan `nebra_modules/` for native libraries (`.so`, `.dylib`, `.dll`) - fail if any are found (current Nebra can't ship them inside the binary).
3. Generate a temporary `Launcher.csproj` that references `Nebra.Runtime.dll` (the embedded Lua wrapper), with every compiled `.lua` plus every pre-built `.lua` from `nebra_modules/` as embedded resources.
4. Run `dotnet publish -c Release -r <rid>` with single-file self-contained settings.
5. Copy the published binary to `--out`, chmod +x on Unix.

```bash
nebra compile                    # → ./my-app (~70 MB)
nebra compile --out dist/server  # custom path
nebra compile --aot              # smaller binary (~15 MB), needs native toolchain
```

See [Standalone Binaries](./native-binaries.md) for tuning and troubleshooting.

---

## `nebra docs [args]`

Generates documentation for the project's exported APIs from inline doc comments.

| Flag             | Default | Description                                          |
|------------------|---------|------------------------------------------------------|
| `--out <dir>`    | `docs`  | Output directory for generated files                 |
| `--no-html`      | off     | Skip the HTML site, keep only Markdown               |
| `--no-md`        | off     | Skip Markdown, keep only the HTML site               |

The generator reads LuaCATS-style triple-dash comments (`--- summary`, `---@param`, `---@return`) on exported declarations and produces one Markdown file per module + an HTML site index. See [Doc Comments](./doc-comments.md) for the comment syntax.

```bash
nebra docs                       # → docs/index.html + per-module .md
nebra docs --out public/api      # custom dir
nebra docs --no-html             # plain Markdown only
```

---

## `nebra install`

Installs declared dependencies from `nebra.toml` into `nebra_modules/`. Reads `nebra.lock` if present, otherwise resolves fresh from git.

| Flag                                        | Description                                                |
|---------------------------------------------|------------------------------------------------------------|
| `--frozen`                                  | Require an up-to-date `nebra.lock`; error on drift           |
| `--offline`                                 | Only use the local git cache; never fetch                  |
| `--no-dev`, `--production`                  | Skip `[dev-dependencies]`                                  |
| `--no-cache`                                | Bypass the registry cache for alias resolution             |
| `--allow-scripts`                           | Allow `[scripts] install` / `postinstall` to run           |
| `--allow-scripts=pkg1,pkg2`                 | Allow scripts only for the named packages                  |

```bash
nebra install                              # install everything
nebra install --frozen --no-dev            # CI mode
nebra install --allow-scripts=lua-protobuf  # allow specific package's scripts
```

---

## `nebra add <spec>`

Adds a dependency to `nebra.toml` (preserving formatting + comments via TOML round-trip) and installs it.

Spec formats:

- `@scope/name[@range]` (package registry): `@nebra-lang/nanos-world@^1`
- `name[@range]`: an unscoped name the registry still knows, such as `nanos-world`
- `github:owner/repo[@ref]`: `github:DasDarki/nebra-strings@v1.2.0`
- `gh:owner/repo[@ref]`: shorthand for the above
- `https://...`: any git URL, optional `#ref` suffix
- `file:../local-path`: link a local checkout

| Flag                                        | Description                                                |
|---------------------------------------------|------------------------------------------------------------|
| `--dev`                                     | Add to `[dev-dependencies]` instead of `[dependencies]`    |
| `--peer`                                    | Add to `[peer-dependencies]`                               |
| `--no-cache`                                | Bypass the registry cache                                  |
| `--allow-scripts[=names]`                   | Allow lifecycle scripts (all, or comma-separated names)    |

```bash
nebra add @nebra-lang/nanos-world
nebra add github:owner/cool-lib@v1
nebra add @someone/test-utils@^1 --dev
nebra add file:../my-utils
```

---

## `nebra remove <name>`

Removes a dependency from `nebra.toml` and deletes it from `nebra_modules/`. Also unlinks transitive deps that no other package references.

```bash
nebra remove cool-lib
```

---

## `nebra pm prune [<spec>]`

Wipe the on-disk package caches so the next `install` / `create` re-fetches from origin. Use when a package was republished under the same ref (force-pushed tag, mutable branch) and the cached snapshot is now stale.

```bash
nebra pm prune                                        # wipe ALL caches
nebra pm prune github:nebra-lang/nanos-world-types        # wipe one repo's bare clone + snapshots
```

`nebra pm prune` (no args) removes `~/.neb/cache/git`, `~/.neb/store` and `~/.neb/tmp`. With a git spec it removes only that repo's bare clone and every commit snapshot of it.

## `nebra pm update [<name>]`

Re-resolves dependencies against origin and rewrites the lockfile. The bare clone is `git fetch`'d on every invocation, so any floating ref (default branch, semver range, mutable tag) picks up new commits. Pinned exact-commit specs end up at the same SHA.

```bash
nebra pm update                          # re-resolve every dep
nebra pm update nanos-world-types        # re-resolve just one
```

`pm update` (no args) wipes the entire lockfile and re-resolves; `pm update <name>` drops only the named entry. Either way the install step that follows re-extracts snapshots and re-links `nebra_modules/`. For *cache-corruption* scenarios (mis-named ref, force-pushed branch overwriting the bare clone), reach for `nebra pm prune` instead.

## `nebra pm refresh-registry`

Re-downloads the alias registry index so name → git-URL lookups (`nebra add cool-lib@v1`) see the latest published packages.

```bash
nebra pm refresh-registry
```

`nebra registry refresh` still works as a deprecated alias and prints a one-line migration notice.

---

## `nebra publish`

Publishes the project in the current directory to the [package registry](https://github.com/nebra-lang/pm-registry), so others can `nebra add` it. Run it from the root of a public GitHub repository; the repository is taken from its `origin` remote and its `nebra.toml` names the package. Versions are the repository's git tags, so publish once and tag releases as usual.

| Flag                   | Description                                                       |
|------------------------|-------------------------------------------------------------------|
| `--scope <scope>`      | Publish as `@<scope>/<name>`; default is your GitHub account      |
| `--description <text>` | A short description shown with the package                        |

```bash
nebra publish                      # @<your account>/<name>
nebra publish --scope my-org       # @my-org/<name>
```

The request is sent as an issue to the registry through your GitHub login (`gh auth login`, `GITHUB_TOKEN` or `GH_TOKEN`), and the command waits for its answer. Without a login it opens the prepared issue in your browser. A scope belongs to the GitHub accounts that own it: the first package published to `@name` claims it for the account or organization of that name, for an organization through a *public* membership.

## `nebra unpublish [<package>]`

Removes a package from the registry, by default the current project's.

```bash
nebra unpublish
nebra unpublish @my-org/old-lib
```

## `nebra scope add-owner|remove-owner <scope> <user>`

Lets another GitHub account publish to one of your scopes, or takes that right away. A scope keeps at least one owner.

```bash
nebra scope add-owner my-org alice
```

---

## `nebra lps`

Starts the Language Server Protocol server over stdio. Editors invoke this transparently; you should not need to call it directly.

```bash
nebra lps          # blocks reading from stdin / writing to stdout
```

The server provides diagnostics, hover, go-to-definition, completion, signature help, rename, find-references, document symbols, semantic tokens, and code actions ("Implement interface", "Auto-import", "Compile current file"). It reuses the compiler's `CheckPipeline` (full pipeline minus mangle/codegen) for fast incremental feedback.

---

## `nebra use [path]` / `nebra unuse`

Forwards every call of `nebra` to another build, such as a local checkout of the compiler, until `nebra unuse`. The path may be a `nebra` executable, a `nebra.dll` or the directory a build put them in. `nebra use` without a path shows which build is in use. The language server your editor starts goes through it as well; `nebra upgrade` and `nebra check` always act on the installed version.

```bash
nebra use ~/src/nebra/compiler/bin/Debug/net10.0
nebra unuse
```

---

## `nebra version`

Prints the Nebra compiler version (semver, three components).

```bash
nebra version       # → nebra 0.X.Y
```

---

## `nebra help`

Prints a one-screen summary of every command + the most common flags. Detailed flags live in this document.

---

## Exit codes

| Code | Meaning                                              |
|------|------------------------------------------------------|
| 0    | Success                                              |
| 1    | Any failure: compile error, test failure, missing file, install failure, etc. |

For `nebra run` and `nebra compile`'d binaries, the exit code propagates from the user's Lua script (`os.exit(N)` works as expected).
