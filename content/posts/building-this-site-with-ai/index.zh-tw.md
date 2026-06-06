---
title: "用 AI 一天打造這個網站：把履歷與技術筆記自動化部署上線"
date: 2026-06-07T01:00:00+08:00
draft: false
summary: "用 Claude Code 搭配 Hugo + Blowfish + GitHub Actions，一天內把一份 PDF 履歷和散落的技術筆記，變成這個中英雙語、自動部署的個人網站。記錄流程、脫敏紀律與踩到的坑。"
tags: ["AI", "Hugo", "GitHub Actions", "Claude Code"]
---

這個網站本身，就是我今天用 AI（Claude Code）協作一天做出來的。起點只有一份 PDF 履歷、
幾篇散落的技術筆記和演講稿；終點是這個**中英雙語、push 就自動上線**的個人站。這篇把流程、
紀律與踩到的坑記下來，給想做類似事情的人參考。

> 延伸閱讀：[AI 協作開發指南](../ai-collaboration-guide/)。

## 技術組合

- **Claude Code** — AI 結對夥伴，負責大部分機械性的工作、翻譯與一致性檢查
- **Hugo（extended）+ Blowfish 主題** — 靜態網站產生器；Blowfish 以 Hugo Module 安裝
- **GitHub Actions → GitHub Pages** — push 到 `main` 就自動建置、部署

## 流程

### 1. 先定位，再寫 README

第一步不是寫程式，是想清楚**定位**。GitHub profile README 是一張「名片」，所以先決定
受眾（求職／海外）、主軸（後端 + 應用 ML/CV）、深度（極簡名片、細節留給網站）。
這段我用一個「逐層拷問」的流程，把每個決策分支問清楚再動手——避免做完才發現方向錯了。

結果就是一張極簡名片：一句 tagline、分組的技術 badges、三個聯絡連結，其餘全部導流到這個網站。

### 2. 用 Hugo + Blowfish 搭出網站骨架

- 用 **Hugo Modules** 安裝 Blowfish（`hugo mod init` → 匯入主題 → `hugo mod get`），
  之後升級主題只要 `hugo mod get -u`。
- **中英雙語**用 Hugo 內建 i18n：語言定義放在 `languages.toml`，內容用 `.en.md` /
  `.zh-tw.md` 後綴。
- 首頁採 Blowfish 的 **profile 版面**，把履歷的精華（後端、ML/CV、演講、論文）放上去。

### 3. 用 GitHub Actions 自動部署

寫一支 workflow：push 到 `main` → 安裝 Hugo extended → `hugo mod get` → `hugo --gc
--minify` → 透過官方 `upload-pages-artifact` / `deploy-pages` 發佈到 GitHub Pages。
之後寫文章只要 push，1–2 分鐘就上線。

> 唯一的手動步驟：到 repo 的 **Settings → Pages → Source 設成「GitHub Actions」**，
> 第一次設定後就不用再管。

### 4. 把現有文件變成文章——關鍵是「紀律」

這是最需要人來把關、AI 不能自己決定的部分。我把履歷、演講稿、自學筆記、工作筆記
逐一轉成雙語文章，過程中堅持幾條紀律：

- **脫敏（最重要）**：任何來自工作的內容，只描述**領域與技術**，不出現雇主名稱、
  內部服務名、業務細節。例如一份內部的 Feign 開發指南，原本範例是公司的某個敏感業務，
  我把整個範例領域換成中性的「電商訂單」，只保留通用的架構模式。
- **清掉私人連結**：筆記裡指向個人知識庫的引用連結，對讀者是壞連結、還會外洩內部 ID，
  全部移除。
- **清掉追蹤參數**：貼課程／外部連結時，把網址後面一長串 `gclid`、`utm_*` 之類的
  追蹤參數砍掉，只留乾淨網址。
- **雙語**：每篇都出中英兩版。

每次部署後我都跑一個簡單的稽核（例如 `grep -ri` 掃公開產物有沒有殘留敏感字眼），
確認線上乾淨才算完成。

## 踩到的坑

- **主題與 Hugo 版本要對齊**：Blowfish 有測試過的 Hugo 上限版本，用更新的 Hugo 會跳
  相容性警告。把本機與 CI 都鎖在主題支援的版本最省事。
- **未來日期會被略過**：Hugo 預設不建置「未來」的文章。我寫當天日期時，因為機器時區
  （UTC+8）讓「當天 UTC 午夜」其實還在未來，文章就沒被建出來。解法：日期帶上明確時區
  （或用前一天）。
- **設定檔命名**：語言「定義」要放在 `languages.toml`；`languages.en.toml` 這種檔名
  會被 Hugo 把 `.en` 當成語言限定詞而忽略內容。`.<lang>` 後綴只適用於 `menus.<lang>.toml`。

## 心得

AI 把機械性的工作（搭骨架、寫 workflow、翻譯、格式整理、一致性稽核）做得又快又穩，
讓「一天上線」變得可行。但有兩件事始終是人的責任：**定位的判斷**，以及**揭露界線的把關**
——什麼能公開、什麼要脫敏，AI 可以提醒、可以執行，但最後拍板的是你。
