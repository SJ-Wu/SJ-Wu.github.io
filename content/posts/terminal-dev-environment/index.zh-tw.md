---
title: "為 AI 協作打造的終端機開發環境：NeoVim + Yazi + tmux + LazyGit × Claude Code"
date: 2026-05-01
draft: false
summary: "Claude Code 是終端原生的 AI 協作工具。用 tmux + NeoVim + Yazi + LazyGit 把它包進一個全鍵盤、全終端的工作流，讓人專注在『審查與把關 AI 的產出』，並記錄設定時踩到的坑。"
tags: ["Claude Code", "AI", "NeoVim", "tmux", "終端機"]
---

[Claude Code](../ai-collaboration-guide/) 這類 AI 協作工具是**終端原生**的——它就跑在終端機裡。
所以一個順手的終端環境，等於直接放大了 AI 協作的效率：AI 負責大量產出，人則在同一個
全鍵盤環境裡快速**審查、微調、把關**。這篇整理我目前用 **tmux + NeoVim + Yazi + LazyGit**
搭配 Claude Code 的配置與工作流，以及設定時踩到的坑。

> 環境版本：NeoVim `v0.11.6`、Yazi `26.5.6`、tmux `3.6`、終端機模擬器 Ghostty、主要語言 Python。
> （個人化的自訂鍵位細節此處略過，只談架構、設定與工作流。）

## 1. 為什麼這套組合特別適合搭配 Claude Code

把這四套工具當成 Claude Code 的「外圍」，各自補上 AI 協作中人最需要的環節：

| 工具 | 在 AI 協作流程中的角色 |
|---|---|
| **tmux** | 一個窗格跑 **Claude Code**、另一個窗格開編輯器或跑測試，並排對照 AI 的輸出與實際程式碼；session 可長駐，隨時回到同一個工作現場。 |
| **NeoVim** | AI 改完的檔案在這裡**審查、微調、跑測試**（`neotest`）；LSP 立即標出型別／lint 問題，快速驗證 AI 的改動是否站得住。 |
| **LazyGit** | **把關 AI 產出的關鍵一站**：用 TUI 逐個 hunk 檢視 diff、選擇性 stage、寫 commit——AI 改一批，人在這裡一個一個確認再進版控。 |
| **Yazi** | 在專案與檔案樹間快速導航，找到要交給 Claude Code 處理的路徑。 |

核心理念：**讓 AI 與人待在同一個全鍵盤、全終端的環境**，不用在 GUI 與終端之間來回切換，
審查與迭代的循環越短，AI 協作的效益越高。NeoVim 是編輯中樞，tmux 在最外層提供窗格管理，
並負責 Ghostty ↔ TUI 之間的終端能力協商（這也是踩坑最多的地方，見 [§6](#6-踩到的坑與解法)）。

## 2. NeoVim

設定採 **Lua-based**，以 `lazy.nvim` 作為套件管理器。目錄結構：

```
~/.config/nvim/
├── init.lua                  # 進入點：bootstrap lazy.nvim + 載入設定
├── lazy-lock.json            # 套件版本鎖定
└── lua/
    ├── config/
    │   ├── options.lua        # vim.opt.* 基本設定
    │   └── keymaps.lua        # 鍵位對應（個人化，略）
    └── plugins/               # 每個檔案一組外掛 spec
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

### 2.1 進入點 `init.lua`

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

重點：
- 首次啟動會自動 `git clone` lazy.nvim（bootstrap），不需手動安裝。
- `require("lazy").setup("plugins", ...)` 會自動載入 `lua/plugins/` 底下所有檔案。
- `change_detection.notify = false`：關閉「設定檔被外部修改」的提示——這點搭配 Claude Code 特別有感，
  因為 AI 會頻繁改檔，否則通知會一直跳。

### 2.2 基本選項 `config/options.lua`

```lua
local opt = vim.opt

opt.number = true            -- 顯示行號
opt.relativenumber = true    -- 相對行號（搭配 j/k 跳行）
opt.expandtab = true         -- Tab 轉空白
opt.shiftwidth = 4           -- 縮排寬度 4
opt.tabstop = 4              -- Tab 顯示寬度 4
opt.smartindent = true       -- 智慧縮排
opt.wrap = false             -- 不自動換行
opt.cursorline = true        -- 高亮當前行
opt.scrolloff = 8            -- 游標上下保留 8 行
opt.signcolumn = "yes"       -- 固定顯示 sign 欄（避免診斷圖示跳動）
opt.termguicolors = true     -- 24-bit 真彩色
opt.updatetime = 250         -- 更新延遲（影響 diagnostic / CursorHold）
opt.clipboard = "unnamedplus" -- 與系統剪貼簿共用
```

### 2.3 外掛總覽

| 檔案 | 外掛 | 用途 |
|---|---|---|
| `colorscheme.lua` | `rebelot/kanagawa.nvim` | 配色（`dragon` 主題） |
| `lsp.lua` | `mason` + `mason-lspconfig` + `nvim-lspconfig` + `conform.nvim` | LSP / 格式化 |
| `completion.lua` | `nvim-cmp` + `LuaSnip` | 自動補全 |
| `telescope.lua` | `telescope.nvim` | 模糊搜尋 |
| `treesitter.lua` | （停用，見下方） | 語法解析 |
| `motion.lua` | `hop.nvim` + `nvim-surround` + `substitute.nvim` | 快速跳轉 / 環繞 / 交換 |
| `autopairs.lua` | `nvim-autopairs` | 自動補括號 |
| `testing.lua` | `neotest` + `neotest-python` | 測試執行 |
| `lazygit.lua` | `lazygit.nvim` | Git TUI 整合 |
| `yazi.lua` | `yazi.nvim` | 檔案總管整合 |

幾個值得一提、且與 AI 協作直接相關的設定取向：

- **LSP（`lsp.lua`）**：用 `mason` 自動安裝 `pyright`（型別 / 補全）與 `ruff`（lint / format）。
  採 NeoVim 0.11 新 API：`vim.lsp.config()` + `vim.lsp.enable()`，不再用 `lspconfig.setup()`
  （避免 deprecated 警告，見 [§6](#6-踩到的坑與解法)）。`conform.nvim` 讓 Python 在**存檔時自動以
  `ruff_format` 格式化**——AI 產出的程式碼存檔即統一風格。LSP 的即時診斷，也是審查 AI 改動最快的一道關。
- **測試（`testing.lua`）**：`neotest` + `neotest-python`（runner = `pytest`）——AI 改完馬上在編輯器內
  跑測試驗證，是 AI 協作裡最重要的回饋迴路。
- **補全（`completion.lua`）**：`nvim-cmp`，來源優先序 `nvim_lsp` → `luasnip` →（次要）`buffer` / `path`。
- **搜尋（`telescope.lua`）**：find_files、live_grep、document_symbols、buffer 切換——快速定位要交給 AI 的位置。
- **移動（`motion.lua`）**：`hop.nvim`（兩字元跳轉）、`nvim-surround`（環繞符號）、`substitute.nvim`（交換）。
- **Treesitter（`treesitter.lua`）**：NeoVim 0.11 已內建常用 parser，直接停用 `nvim-treesitter`
  外掛、改用內建 `vim.treesitter`：

```lua
{ "nvim-treesitter/nvim-treesitter", enabled = false },
```

## 3. Yazi（檔案總管）

設定檔 `~/.config/yazi/yazi.toml`，目前極簡：

```toml
[manager]
show_hidden = false   # 預設不顯示隱藏檔（可在 yazi 內按 . 暫時切換）
```

使用方式：終端機直接 `yazi` 獨立執行；或透過 `yazi.nvim` 外掛從 NeoVim 內呼叫
（`open_for_directories = true` 讓開啟目錄時也用 Yazi）。在 AI 協作時，常用它快速瀏覽專案結構、
確認 AI 新增 / 異動了哪些檔案。

> Yazi 在 tmux 內執行時，終端能力協商需要額外設定，否則會出現亂碼鍵入問題，見 [§6](#6-踩到的坑與解法)。

## 4. tmux

tmux 是讓「Claude Code 與編輯器並存」的關鍵：一個窗格跑 Claude Code、另一個窗格編輯與跑測試，
session 長駐不中斷。設定檔 `~/.tmux.conf`：

```bash
# 啟用滑鼠支援（可用滑鼠點擊切換 Pane 與調整大小）
set -g mouse on
# 視窗索引從 1 開始（方便對應鍵盤左手位置）
set -g base-index 1
# Vi 風格快捷鍵
setw -g mode-keys vi
# 分區窗格索引也從 1 開始
setw -g pane-base-index 1
# 自動重新編號視窗（刪除視窗後號碼自動遞補）
set -g renumber-windows on

# 允許 DCS/CSI passthrough，避免 yazi 等 TUI 的終端能力探測序列洩漏成鍵盤輸入
set -g allow-passthrough on

# 告訴 tmux Ghostty 支援的能力，讓 yazi 不需自行 probe
set -g default-terminal "tmux-256color"
set -as terminal-features ",xterm-ghostty:RGB:hyperlinks:usstyle"
set -as terminal-features ",*:RGB"

# 新視窗 / split 繼承當前 pane 的工作目錄
bind c new-window -c "#{pane_current_path}"
bind '"' split-window -v -c "#{pane_current_path}"
bind % split-window -h -c "#{pane_current_path}"
```

重點整理：

| 設定 | 作用 |
|---|---|
| `mouse on` | 滑鼠切換窗格 / 調整大小 |
| `base-index 1` / `pane-base-index 1` | 視窗與窗格皆從 1 起算 |
| `renumber-windows on` | 關閉視窗後自動遞補編號 |
| `allow-passthrough on` | **關鍵**：讓 TUI 的終端探測序列正確 passthrough |
| `default-terminal "tmux-256color"` | 正確的 terminfo |
| `terminal-features` | 宣告 RGB / hyperlinks / 底線樣式能力 |
| `bind c / " / %` 加 `-c "#{pane_current_path}"` | 新窗格繼承當前目錄 |

## 5. LazyGit

LazyGit 是**審查 AI 產出的把關站**：AI 改一批，人在 LazyGit 用 TUI 逐個 hunk 檢視 diff、
選擇性 stage、再寫 commit，確保進版控的每一段都看過。

- 設定檔 `~/.config/lazygit/config.yml` **目前為空**（沿用 LazyGit 預設）。
- 使用方式：終端機直接 `lazygit` 獨立執行；或透過 `lazygit.nvim` 外掛從 NeoVim 內呼叫。

> 若日後要自訂（例如 pager、commit 範本、custom commands），在此 yml 補上即可。

## 6. 安裝 / 還原步驟

在新機器上重建環境的大致順序：

1. 安裝四套工具（NeoVim ≥ 0.11、Yazi、tmux、LazyGit），以及 Claude Code。
2. 還原設定檔：
   - `~/.config/nvim/`
   - `~/.config/yazi/yazi.toml`
   - `~/.tmux.conf`
   - `~/.config/lazygit/config.yml`
3. 啟動 NeoVim，lazy.nvim 會自動 bootstrap 並安裝所有外掛；`mason` 會自動安裝 `pyright`、`ruff`。
4. tmux 重新載入設定：`tmux source-file ~/.tmux.conf`（或重開 session）。
5. 用 `lazy-lock.json` 鎖定的 commit 還原套件版本：NeoVim 內執行 `:Lazy restore`。

健康檢查：

```bash
nvim --headless -c "checkhealth" -c "qa"
```

## 7. 踩到的坑與解法

### 坑 1：Yazi 在 tmux 內出現亂碼 / 鍵盤被插入怪字元

**現象**：在 tmux 內開 Yazi（或其他 TUI），畫面出現奇怪字元，或鍵盤輸入被插入無意義序列。

**原因**：Yazi 啟動時會送出終端能力「探測序列」（DCS / CSI），tmux 預設不會把這些序列正確傳遞給外層終端（Ghostty），導致回應序列被當成鍵盤輸入。

**解法**（已寫進 `.tmux.conf`）：
```bash
set -g allow-passthrough on
set -g default-terminal "tmux-256color"
set -as terminal-features ",xterm-ghostty:RGB:hyperlinks:usstyle"
set -as terminal-features ",*:RGB"
```
- `allow-passthrough on`：允許 DCS/CSI passthrough。
- 直接用 `terminal-features` 宣告 Ghostty 支援的能力，讓 Yazi **不需自行 probe**，從源頭避免探測序列外洩。

### 坑 2：NeoVim 0.11 的 LSP deprecated 警告

**現象**：用傳統 `require("lspconfig").pyright.setup{}` 會在 0.11 跳出 deprecated 警告。

**原因**：NeoVim 0.11 推出內建 LSP 設定 API，舊的 `lspconfig.setup()` 流程被標記為過時。

**解法**（已寫進 `lsp.lua`）：改用新 API——
```lua
vim.lsp.config("pyright", { capabilities = ..., on_attach = ... })
vim.lsp.config("ruff",    { capabilities = ..., on_attach = ... })
vim.lsp.enable({ "pyright", "ruff" })
```
`nvim-lspconfig` 仍保留，但**只拿它的 server 預設值**，不呼叫它的 `setup()`。

### 坑 3：nvim-treesitter 與 0.11 內建 parser 衝突 / 多餘

**現象**：裝了 `nvim-treesitter` 反而與內建 parser 重複，徒增維護成本。

**原因**：NeoVim 0.11 已內建 python、markdown、lua 等常用語言的 treesitter parser。

**解法**：直接停用 `nvim-treesitter`（`enabled = false`），改用內建 `vim.treesitter`。

### 坑 4：貼上時內容被刪除動作覆蓋

**現象**：剛複製（yank）的內容，做了刪除動作後再貼上就不見了。

**原因**：NeoVim 的 unnamed register 會同時被刪除（`d` / `x`）覆蓋。

**解法**：改用 yank 專用暫存器 `"0`（只記錄 yank 內容、不受刪除影響），把貼上動作綁到從 `"0` 取值，貼上才穩定。

### 坑 5：clipboard 與系統剪貼簿不同步

**現象**：NeoVim 內複製的內容無法貼到其他應用程式（例如想把程式碼貼給 AI 對話）。

**解法**（已寫進 `options.lua`）：
```lua
opt.clipboard = "unnamedplus"
```
> 注意：Linux 上仍需安裝剪貼簿提供者（如 `xclip` / `xsel` / `wl-clipboard`），否則 `unnamedplus` 無作用。

### 坑 6：signcolumn 抖動

**現象**：LSP 診斷圖示出現 / 消失時，文字會左右跳動。

**解法**（已寫進 `options.lua`）：固定顯示 sign 欄——
```lua
opt.signcolumn = "yes"
```

## 附錄：套件版本鎖定

NeoVim 外掛版本由 `~/.config/nvim/lazy-lock.json` 鎖定（git commit 級別）。
換機或還原時用 `:Lazy restore` 可重現完全相同的套件版本。主要外掛：

`lazy.nvim`、`kanagawa.nvim`、`mason.nvim` / `mason-lspconfig.nvim` / `nvim-lspconfig`、
`nvim-cmp`（+ `cmp-nvim-lsp` / `cmp-buffer` / `cmp-path` / `LuaSnip` / `cmp_luasnip`）、
`conform.nvim`、`telescope.nvim`（+ `plenary.nvim`）、`hop.nvim`、`nvim-surround`、
`substitute.nvim`、`nvim-autopairs`、`neotest`（+ `nvim-nio` / `neotest-python`）、
`lazygit.nvim`、`yazi.nvim`。
