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

## What the language server gives you

The server runs the same compiler pipeline the CLI does, so its answers match what `nebra build`
would say.

| Feature | Notes |
|---------|-------|
| Diagnostics | Errors and warnings as you type, with the same codes and help text the CLI prints |
| Hover | Inferred type of any expression, resolved signatures, and your `---` doc comments |
| Completion | Scope-aware symbols, class and interface members, and primitive type names inside annotations. Where a value of a known type is expected (an annotated local, an argument, an assignment, a `return`, `==` or a `match` case), what fits comes first: the members of an expected enum, written as the [shorthand](../language/enums.md#member-shorthand), then the values and functions of that type |
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
