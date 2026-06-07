---
title: "A Terminal Dev Environment Built for AI Collaboration: NeoVim + Yazi + tmux + LazyGit × Claude Code"
date: 2026-05-01
draft: false
summary: "Claude Code is a terminal-native AI coding tool. Wrapping it in tmux + NeoVim + Yazi + LazyGit gives an all-keyboard, all-terminal workflow where the human focuses on reviewing and gatekeeping what the AI produces — plus the gotchas I hit setting it up."
tags: ["Claude Code", "AI", "NeoVim", "tmux", "Terminal"]
---

AI coding tools like [Claude Code](../ai-collaboration-guide/) are
**terminal-native** — they run right inside the terminal. So a comfortable
terminal environment directly amplifies AI collaboration: the AI produces a lot,
and the human **reviews, tweaks, and gatekeeps** in the same all-keyboard
environment. This post covers how I set up **tmux + NeoVim + Yazi + LazyGit**
around Claude Code, and the gotchas I hit along the way.

> Versions: NeoVim `v0.11.6`, Yazi `26.5.6`, tmux `3.6`, terminal emulator
> Ghostty, primary language Python. (Personal custom keymaps are omitted — this is
> about the architecture, config, and workflow.)

## 1. Why this combination pairs well with Claude Code

Think of these four tools as the "periphery" around Claude Code, each covering the
part of AI collaboration the human most needs:

| Tool | Role in the AI-collaboration workflow |
|---|---|
| **tmux** | Run **Claude Code** in one pane and an editor or test runner in another, side by side with the actual code; the session stays alive so you can return to the same workspace anytime. |
| **NeoVim** | **Review, tweak, and test** the files the AI changed (`neotest`); LSP flags type/lint issues instantly, so you can quickly verify whether the AI's change holds up. |
| **LazyGit** | **The key gatekeeping stop:** review the diff hunk by hunk in a TUI, stage selectively, and write the commit — the AI changes a batch, the human confirms each piece before it enters version control. |
| **Yazi** | Navigate the project and file tree quickly to find the path to hand to Claude Code. |

The core idea: **keep the AI and the human in the same all-keyboard, all-terminal
environment**, with no switching between a GUI and the terminal. The shorter the
review-and-iterate loop, the more AI collaboration pays off. NeoVim is the editing
hub; tmux sits on the outside for pane management and handles the terminal
capability negotiation between Ghostty and the TUIs (which is where most of the
gotchas live — see [§7](#7-gotchas-and-fixes)).

## 2. NeoVim

A **Lua-based** config with `lazy.nvim` as the plugin manager. Directory layout:

```
~/.config/nvim/
├── init.lua                  # entry: bootstrap lazy.nvim + load config
├── lazy-lock.json            # plugin version lock
└── lua/
    ├── config/
    │   ├── options.lua        # vim.opt.* basic options
    │   └── keymaps.lua        # keymaps (personal, omitted)
    └── plugins/               # one plugin spec per file
        ├── lsp.lua
        ├── completion.lua
        ├── telescope.lua
        ├── treesitter.lua
        ├── colorscheme.lua
        ├── motion.lua
        ├── autopairs.lua
        ├── testing.lua
        ├── lazygit.lua
        └── yazi.lua
```

### 2.1 Entry point `init.lua`

```lua
-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git", "clone", "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable", lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

require("config.options")
require("config.keymaps")

require("lazy").setup("plugins", {
  change_detection = { notify = false },
})
```

Notes:
- On first launch it auto `git clone`s lazy.nvim (bootstrap) — no manual install.
- `require("lazy").setup("plugins", ...)` auto-loads every file under
  `lua/plugins/`.
- `change_detection.notify = false`: silence the "config changed on disk"
  notification — especially nice with Claude Code, since the AI edits files
  frequently and the notification would otherwise keep popping up.

### 2.2 Basic options `config/options.lua`

```lua
local opt = vim.opt

opt.number = true            -- line numbers
opt.relativenumber = true    -- relative numbers (pairs with j/k)
opt.expandtab = true         -- tabs to spaces
opt.shiftwidth = 4           -- indent width 4
opt.tabstop = 4              -- tab display width 4
opt.smartindent = true       -- smart indent
opt.wrap = false             -- no line wrap
opt.cursorline = true        -- highlight current line
opt.scrolloff = 8            -- keep 8 lines above/below the cursor
opt.signcolumn = "yes"       -- always show the sign column (no icon jitter)
opt.termguicolors = true     -- 24-bit true color
opt.updatetime = 250         -- update delay (affects diagnostics / CursorHold)
opt.clipboard = "unnamedplus" -- share the system clipboard
```

### 2.3 Plugin overview

| File | Plugin | Purpose |
|---|---|---|
| `colorscheme.lua` | `rebelot/kanagawa.nvim` | color scheme (`dragon`) |
| `lsp.lua` | `mason` + `mason-lspconfig` + `nvim-lspconfig` + `conform.nvim` | LSP / formatting |
| `completion.lua` | `nvim-cmp` + `LuaSnip` | completion |
| `telescope.lua` | `telescope.nvim` | fuzzy finding |
| `treesitter.lua` | (disabled, see below) | parsing |
| `motion.lua` | `hop.nvim` + `nvim-surround` + `substitute.nvim` | jump / surround / swap |
| `autopairs.lua` | `nvim-autopairs` | auto-close pairs |
| `testing.lua` | `neotest` + `neotest-python` | running tests |
| `lazygit.lua` | `lazygit.nvim` | Git TUI integration |
| `yazi.lua` | `yazi.nvim` | file-manager integration |

A few choices worth noting that bear directly on AI collaboration:

- **LSP (`lsp.lua`):** `mason` auto-installs `pyright` (types/completion) and
  `ruff` (lint/format). Uses NeoVim 0.11's new API — `vim.lsp.config()` +
  `vim.lsp.enable()` — instead of `lspconfig.setup()` (avoids the deprecation
  warning, see [§7](#7-gotchas-and-fixes)). `conform.nvim` **formats Python on
  save with `ruff_format`**, so AI-produced code is normalized the moment it's
  saved. The live LSP diagnostics are also the fastest gate for reviewing an AI
  change.
- **Testing (`testing.lua`):** `neotest` + `neotest-python` (runner = `pytest`) —
  run tests right in the editor after the AI edits, the most important feedback
  loop in AI collaboration.
- **Completion (`completion.lua`):** `nvim-cmp`, source priority `nvim_lsp` →
  `luasnip` → (secondary) `buffer` / `path`.
- **Search (`telescope.lua`):** find_files, live_grep, document_symbols, buffer
  switching — quickly locate the spot to hand to the AI.
- **Motion (`motion.lua`):** `hop.nvim` (two-char jump), `nvim-surround`,
  `substitute.nvim` (swap).
- **Treesitter (`treesitter.lua`):** NeoVim 0.11 ships parsers for common
  languages, so the `nvim-treesitter` plugin is disabled in favor of the built-in
  `vim.treesitter`:

```lua
{ "nvim-treesitter/nvim-treesitter", enabled = false },
```

## 3. Yazi (file manager)

Config at `~/.config/yazi/yazi.toml`, currently minimal:

```toml
[manager]
show_hidden = false   # hide dotfiles by default (toggle with . inside yazi)
```

Usage: run `yazi` standalone in the terminal, or call it from NeoVim via the
`yazi.nvim` plugin (`open_for_directories = true` so opening a directory uses Yazi
too). During AI collaboration I use it to quickly browse the project structure and
see which files the AI added or changed.

> Running Yazi inside tmux needs extra terminal-capability setup, or you get
> garbled keystrokes — see [§7](#7-gotchas-and-fixes).

## 4. tmux

tmux is what lets "Claude Code and the editor coexist": one pane runs Claude Code,
another edits and runs tests, and the session persists. Config at `~/.tmux.conf`:

```bash
# Mouse support (click to switch panes and resize)
set -g mouse on
# Window index starts at 1
set -g base-index 1
# Vi-style keys
setw -g mode-keys vi
# Pane index also starts at 1
setw -g pane-base-index 1
# Auto-renumber windows after one is closed
set -g renumber-windows on

# Allow DCS/CSI passthrough so TUIs' capability probes don't leak as keystrokes
set -g allow-passthrough on

# Tell tmux Ghostty's capabilities so yazi doesn't have to probe
set -g default-terminal "tmux-256color"
set -as terminal-features ",xterm-ghostty:RGB:hyperlinks:usstyle"
set -as terminal-features ",*:RGB"

# New window / split inherits the current pane's working directory
bind c new-window -c "#{pane_current_path}"
bind '"' split-window -v -c "#{pane_current_path}"
bind % split-window -h -c "#{pane_current_path}"
```

Highlights:

| Setting | Effect |
|---|---|
| `mouse on` | switch/resize panes with the mouse |
| `base-index 1` / `pane-base-index 1` | windows and panes both start at 1 |
| `renumber-windows on` | renumber after closing a window |
| `allow-passthrough on` | **key:** lets TUI capability probes pass through correctly |
| `default-terminal "tmux-256color"` | correct terminfo |
| `terminal-features` | declare RGB / hyperlinks / underline-style support |
| `bind c / " / %` with `-c "#{pane_current_path}"` | new pane inherits the cwd |

## 5. LazyGit

LazyGit is the **gatekeeping stop for AI output**: the AI changes a batch, and the
human reviews the diff hunk by hunk in the TUI, stages selectively, and writes the
commit — so every piece that enters version control has been looked at.

- Config `~/.config/lazygit/config.yml` is **currently empty** (LazyGit defaults).
- Usage: run `lazygit` standalone, or call it from NeoVim via the `lazygit.nvim`
  plugin.

> If you later want to customize (pager, commit templates, custom commands), add
> them to that yml.

## 6. Install / restore steps

Rough order to rebuild the environment on a new machine:

1. Install the four tools (NeoVim ≥ 0.11, Yazi, tmux, LazyGit) — and Claude Code.
2. Restore the config files:
   - `~/.config/nvim/`
   - `~/.config/yazi/yazi.toml`
   - `~/.tmux.conf`
   - `~/.config/lazygit/config.yml`
3. Launch NeoVim; lazy.nvim bootstraps and installs every plugin; `mason`
   auto-installs `pyright` and `ruff`.
4. Reload tmux: `tmux source-file ~/.tmux.conf` (or restart the session).
5. Restore exact plugin versions from `lazy-lock.json`: run `:Lazy restore` in
   NeoVim.

Health check:

```bash
nvim --headless -c "checkhealth" -c "qa"
```

## 7. Gotchas and fixes

### Gotcha 1: Yazi shows garbage / injects junk keystrokes inside tmux

**Symptom:** opening Yazi (or another TUI) inside tmux shows odd characters, or
keystrokes get junk sequences injected.

**Cause:** on startup Yazi emits terminal-capability "probe sequences" (DCS / CSI);
by default tmux doesn't pass these through to the outer terminal (Ghostty), so the
responses get treated as keyboard input.

**Fix** (in `.tmux.conf`):
```bash
set -g allow-passthrough on
set -g default-terminal "tmux-256color"
set -as terminal-features ",xterm-ghostty:RGB:hyperlinks:usstyle"
set -as terminal-features ",*:RGB"
```
- `allow-passthrough on`: allow DCS/CSI passthrough.
- Declaring Ghostty's capabilities via `terminal-features` means Yazi **doesn't
  need to probe**, avoiding the leak at the source.

### Gotcha 2: NeoVim 0.11 LSP deprecation warning

**Symptom:** the classic `require("lspconfig").pyright.setup{}` throws a
deprecation warning on 0.11.

**Cause:** NeoVim 0.11 introduced a built-in LSP config API; the old
`lspconfig.setup()` flow is now deprecated.

**Fix** (in `lsp.lua`): use the new API —
```lua
vim.lsp.config("pyright", { capabilities = ..., on_attach = ... })
vim.lsp.config("ruff",    { capabilities = ..., on_attach = ... })
vim.lsp.enable({ "pyright", "ruff" })
```
`nvim-lspconfig` stays, but **only for its server defaults** — its `setup()` is no
longer called.

### Gotcha 3: nvim-treesitter conflicts with / duplicates the 0.11 built-in parsers

**Symptom:** installing `nvim-treesitter` duplicates the built-in parsers and adds
maintenance overhead.

**Cause:** NeoVim 0.11 already ships treesitter parsers for python, markdown, lua,
and other common languages.

**Fix:** disable `nvim-treesitter` (`enabled = false`) and use the built-in
`vim.treesitter`.

### Gotcha 4: paste gets clobbered by a delete

**Symptom:** something you just yanked disappears after a delete, then a paste.

**Cause:** NeoVim's unnamed register is also overwritten by deletes (`d` / `x`).

**Fix:** use the dedicated yank register `"0` (it only records yanks and isn't
affected by deletes) and bind paste to read from `"0` — then paste is stable.

### Gotcha 5: clipboard out of sync with the system clipboard

**Symptom:** what you copy in NeoVim can't be pasted into other apps (e.g. to paste
code into an AI chat).

**Fix** (in `options.lua`):
```lua
opt.clipboard = "unnamedplus"
```
> Note: on Linux you still need a clipboard provider (`xclip` / `xsel` /
> `wl-clipboard`), or `unnamedplus` does nothing.

### Gotcha 6: signcolumn jitter

**Symptom:** text shifts left/right as LSP diagnostic icons appear/disappear.

**Fix** (in `options.lua`): always show the sign column —
```lua
opt.signcolumn = "yes"
```

## Appendix: plugin version lock

NeoVim plugin versions are pinned in `~/.config/nvim/lazy-lock.json` (git-commit
level). On a new machine, `:Lazy restore` reproduces the exact same versions. Main
plugins:

`lazy.nvim`, `kanagawa.nvim`, `mason.nvim` / `mason-lspconfig.nvim` /
`nvim-lspconfig`, `nvim-cmp` (+ `cmp-nvim-lsp` / `cmp-buffer` / `cmp-path` /
`LuaSnip` / `cmp_luasnip`), `conform.nvim`, `telescope.nvim` (+ `plenary.nvim`),
`hop.nvim`, `nvim-surround`, `substitute.nvim`, `nvim-autopairs`, `neotest`
(+ `nvim-nio` / `neotest-python`), `lazygit.nvim`, `yazi.nvim`.
