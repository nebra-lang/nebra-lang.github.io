---
sidebar_position: 3
title: "Editor Setup"
description: "Install the VS Code extension, or wire the built-in Nebra language server into any LSP-capable editor."
---

# Editor Setup

The Nebra language server is not a separate download. It lives inside the same `nebra` binary you
already installed and starts with `nebra lps`, speaking LSP over stdio.

## VS Code

Install the **Nebra** extension from the Visual Studio Marketplace:

<a
  className="button button--primary button--lg"
  href="https://marketplace.visualstudio.com/items?itemName=DasDarki.nebra">
  Open in Marketplace
</a>

<br />
<br />

Or from the command line:

```bash
code --install-extension DasDarki.nebra
```

The extension activates on any `.neb` or `.d.neb` file, finds `nebra` on your `PATH` and launches
`nebra lps` for you. If the binary is somewhere unusual, point at it explicitly:

```json title=".vscode/settings.json"
{
  "nebra.serverPath": "/opt/nebra/bin/nebra"
}
```

### Extension settings

| Setting | Default | What it does |
|---------|---------|--------------|
| `nebra.serverPath` | `""` | Path to the `nebra` executable. Empty means "find `nebra` on `PATH`". |
| `nebra.trace.server` | `"off"` | LSP traffic logging. Set to `"messages"` or `"verbose"` when reporting a bug. |

### Commands

| Command | What it does |
|---------|--------------|
| `Nebra: Compile this file` | Compiles the active file through the language server, without leaving the editor. |

Server logs appear in the **Nebra Language Server** output channel, which is the first place to look
if something behaves oddly.

## Zed

The [Nebra extension for Zed](https://github.com/nebra-lang/zed-nebra) brings a tree-sitter grammar
for highlighting, brackets, indentation, the outline and text objects, and starts the language server
for `.neb` and `.d.neb` files.

Until it is listed under `zed: extensions`, install it from a checkout:

```bash
git clone https://github.com/nebra-lang/zed-nebra
```

Then run `zed: install dev extension` from the command palette and pick the `zed-nebra` directory.
Zed builds the extension itself, which needs Rust installed through `rustup`.

The extension starts `nebra lps` with the binary set in `lsp.nebra.binary.path`, otherwise `nebra`
on your `PATH`, otherwise the latest release downloaded for your platform. The server settings go
under `lsp.nebra.settings`, with the same names the VS Code extension uses:

```json title="settings.json"
{
  "lsp": {
    "nebra": {
      "binary": {
        "path": "/opt/nebra/bin/nebra"
      },
      "settings": {
        "inlayHints": {
          "types": true,
          "parameterNames": true
        },
        "sideMarkers": "hover",
        "diagnostics": {
          "delay": 120
        }
      }
    }
  },
  "inlay_hints": {
    "enabled": true
  }
}
```

| Setting | Default | What it does |
|---------|---------|--------------|
| `inlayHints.types` | `true` | Show the inferred type of locals declared without an annotation. |
| `inlayHints.parameterNames` | `true` | Show parameter names before the arguments of a call. |
| `sideMarkers` | `"hover"` | `"off"` leaves the side code runs on out of the hover. |
| `diagnostics.delay` | `120` | Milliseconds the server waits after typing stops before it updates diagnostics. |

Zed shows inlay hints only with `inlay_hints.enabled` set.

Tests written with `nebra:test` get run buttons in the gutter next to `test(...)`, `it(...)` and
`describe(...)`, which run that test or suite with `nebra test <file> <name>`. `task: spawn` offers
`nebra build`, `run`, `test`, `watch`, `install`, `compile` and `docs`, run in the root of the
worktree.

Unlike in VS Code, the side of a function is shown only in its hover, without the colored markers in
the editor, and there is no command to compile a single file. Use the `nebra build` task instead.
The server logs are under `dev: open language server logs`.

## What the language server gives you

The server runs the same compiler pipeline the CLI does, so its answers match what `nebra build`
would say.

| Feature | Notes |
|---------|-------|
| Diagnostics | Errors and warnings as you type, with the same codes and help text the CLI prints |
| Hover | Inferred type of any expression, resolved signatures, and your `---` doc comments |
| Completion | Scope-aware symbols, class and interface members, and primitive type names inside annotations. Where a value of a known type is expected (an annotated local, an argument, an assignment, a `return`, `==`, a `match` case, or a field or element of a table literal written for a known type), what fits comes first: the members of an expected enum, written as the [shorthand](../language/enums.md#member-shorthand), then the values and functions of that type |
| Signature help | Parameter hints while you type a call, including overloads |
| Go to definition | Across files and across packages, including into `.d.neb` declarations |
| Find references | Whole-workspace symbol usage |
| Rename | Symbol-aware rename across the workspace |
| Document symbols | Outline view and breadcrumb navigation |
| Semantic tokens | Type-aware highlighting that plain regex grammars cannot do |
| Code actions | Quick fixes, such as stubbing out unimplemented interface members |

## Other editors

Any LSP client works. Point it at `nebra lps` over stdio and register the `.neb` and `.d.neb`
extensions.

### Neovim

```lua title="init.lua"
vim.filetype.add({ extension = { neb = "nebra" } })

vim.lsp.config.nebra = {
  cmd = { "nebra", "lps" },
  filetypes = { "nebra" },
  root_markers = { "nebra.toml", ".git" },
}

vim.lsp.enable("nebra")
```

### Helix

```toml title="languages.toml"
[[language]]
name = "nebra"
scope = "source.nebra"
file-types = ["neb"]
roots = ["nebra.toml"]
language-servers = ["nebra-lps"]

[language-server.nebra-lps]
command = "nebra"
args = ["lps"]
```

### Sublime Text (LSP package)

```json title="LSP.sublime-settings"
{
  "clients": {
    "nebra": {
      "enabled": true,
      "command": ["nebra", "lps"],
      "selector": "source.nebra"
    }
  }
}
```

## Troubleshooting

**Nothing happens when I open a `.neb` file.** Check that `nebra version` works in the same shell your
editor inherits. GUI editors launched from a desktop environment sometimes do not see a `PATH` that
was set in your shell profile. Setting `nebra.serverPath` to an absolute path sidesteps this.

**Types resolve inside a file but not across files.** The server resolves imports relative to
`nebra.toml`, so open the project root as your workspace folder rather than a single file or a
subdirectory.

**Diagnostics look stale.** The server reanalyses on change. If it gets stuck, reload the window and
report it with `nebra.trace.server` set to `"verbose"`.
