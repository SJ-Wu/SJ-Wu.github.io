---
title: "把 GitHub Pages 站台搬到自訂網域（Cloudflare Registrar）"
date: 2026-06-07T02:00:00+08:00
draft: false
summary: "把剛註冊好的 apex 網域接到 GitHub Pages 站台的完整流程：repo 要改什麼、Cloudflare 的 DNS 紀錄怎麼設、哪一個 proxy 預設值會讓 HTTPS 永遠發不出來，以及如何驗證切換完成 —— 最後加上網域驗證，防止別人搶綁你的網域。"
tags: ["GitHub Pages", "Cloudflare", "DNS", "Hugo"]
---

GitHub Pages 站台一開始的網址是 `<user>.github.io`。接上自己的網域其實不難，重點是順序對；
但有幾個步驟有坑 —— 尤其註冊商是 Cloudflare 時，有一個預設值會默默讓 HTTPS 永遠發不出來。
這篇把整個流程記下來，並用 `example.com` 當範例，你直接換成自己的網域即可。

> 延伸閱讀：[用 AI 一天打造這個網站](../building-this-site-with-ai/)。

## 先決定：apex 還是 www

你得先選哪一種形式是 **canonical（主網址）**—— 也就是網址列最後會停在哪一個。

- **apex**（`example.com`）—— 最短最乾淨。代價在 DNS：apex 不能用 `CNAME`，所以要用 `A`
  指向 GitHub 的四個 IP（IPv6 再加 `AAAA`）。
- **www**（`www.example.com`）—— DNS 只要一筆乾淨的 `CNAME`，但網址比較長。

常見做法是以 apex 為主、讓 `www` 轉址過去。兩邊都設好之後，GitHub 會自動處理這個轉址，
所以兩個網址都能用，而網址列顯示的是裸 apex。

## 步驟一 —— repo 改動

有兩樣東西放在 repo 裡，跟 DNS 無關。

**一個 `CNAME` 檔**告訴 GitHub 這個站要回應哪個網域。用 Hugo 的話，把它放在 `static/`，
它會原封不動被複製進建置輸出：

```
static/CNAME      →  example.com
```

檔案裡只放 **canonical** 主機 —— apex 本身，不是 `www`。

**`baseURL`。** 更新它，讓產生的絕對連結、sitemap、RSS 都用新主機：

```toml
# config/_default/hugo.toml
baseURL = "https://example.com/"
```

如果你在 CI 建置，檢查一下 workflow 有沒有覆寫 `baseURL`。常見的 Pages workflow 會這樣：

```yaml
run: hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
```

這個 `base_url` 輸出會在你於 Pages 設好自訂網域**之後**自動變成你的網域，所以 workflow
不用改 —— 但 `hugo.toml` 裡的值對本機建置仍有意義，還是一併更新。

把這些 push 上去，部署就會把 `CNAME` 發佈進線上輸出。

## 步驟二 —— Cloudflare DNS

用 Cloudflare 註冊的網域本來就在用 Cloudflare DNS，所以這步只在 **DNS → Records** 分頁。
以 apex 為主的設定：

| Type  | Name  | Value                  |
|-------|-------|------------------------|
| A     | `@`   | `185.199.108.153`      |
| A     | `@`   | `185.199.109.153`      |
| A     | `@`   | `185.199.110.153`      |
| A     | `@`   | `185.199.111.153`      |
| AAAA  | `@`   | `2606:50c0:8000::153`  |
| AAAA  | `@`   | `2606:50c0:8001::153`  |
| AAAA  | `@`   | `2606:50c0:8002::153`  |
| AAAA  | `@`   | `2606:50c0:8003::153`  |
| CNAME | `www` | `<user>.github.io`     |

（這些是 GitHub 公布的 Pages IP —— 建議對照 GitHub 官方文件確認，以防未來變動。）

### 坑：把橘色雲朵關掉

這就是會咬人的那一個。Cloudflare 預設會 proxy 紀錄 —— 也就是**橘色雲朵**。開著 proxy 時，
GitHub 沒辦法完成它簽發 Let's Encrypt 憑證所需的 HTTP 驗證，HTTPS 就一直發不出來；而如果
你硬讓 Cloudflare 自己的 TLS 擋在前面，又會變成 redirect loop。**把每一筆都設成
「DNS only」—— 灰色雲朵。** 點雲朵圖示直到變灰。

至少保持灰雲，直到 GitHub 憑證簽好、「Enforce HTTPS」可以打勾為止。如果你之後*真的*想用
Cloudflare 的 proxy/CDN，可以再打開 —— 但要把 Cloudflare 的 **SSL/TLS 模式設成「Full」**
（絕對不要用「Flexible」，會 loop）。最簡單、最不出事的做法，就是讓它維持灰雲。

## 步驟三 —— GitHub Pages + HTTPS

在站台 repo：**Settings → Pages**。

1. **Custom domain** → 填 `example.com` → **Save**。GitHub 會跑 DNS check；等你的紀錄
   生效後，會出現綠勾。
2. 等憑證。這要幾分鐘到數十分鐘 —— 憑證好之前，**Enforce HTTPS** 是反灰不能點的。
3. 勾 **Enforce HTTPS**。之後每個 `http://` 請求都會 301 轉到 `https://`。

小撇步：先把 `CNAME`（步驟一）push 上去，再去填自訂網域，這樣 DNS check 跑的時候，網域
已經在部署輸出裡了。

## 步驟四 —— 驗證切換

先看 DNS：

```bash
dig example.com +short          # 應回 GitHub 的四個 IP
dig www.example.com +short      # 應回 <user>.github.io. 後接 IP
```

接著每個入口都應該導到 canonical 的 HTTPS 網址：

```bash
curl -sI https://example.com        | head -1   # HTTP/2 200
curl -sI https://www.example.com    | head -1   # 301 -> https://example.com/
curl -sI http://example.com         | head -1   # 301 -> https（Enforce HTTPS）
curl -sI https://<user>.github.io   | head -1   # 301 -> https://example.com/
```

四個入口 —— apex、www、純 http、舊的 `github.io` —— 全部匯流到 canonical 的 HTTPS 網址。
切換就完成了。

## 步驟五 —— 網域驗證（防搶綁）

還有一個比較少人注意、但值得關掉的風險。如果你哪天把網域從這個 repo 移除、卻沒移除 DNS
紀錄，別人就可能在*他自己的* Pages 站台宣告你的網域。GitHub 的**網域驗證**可以擋掉這件事：
驗證過的網域只能被你的帳號使用。

它在你的**帳號**設定裡，不在 repo：**Settings → Pages → 「Add a domain」**。GitHub 會給你
一筆像這樣的 `TXT` 紀錄：

```
_github-pages-challenge-<user>.example.com   TXT   <token>
```

把那筆 `TXT` 加進 Cloudflare（灰雲 —— TXT 本來也不會被 proxy），再按 **Verify**。用這個
確認它已生效：

```bash
dig _github-pages-challenge-<user>.example.com TXT +short
```

驗證通過後，這個網域就鎖定在你的帳號上了。那筆 `TXT` 紀錄可以留著。

## 一口氣講完順序

repo `CNAME` + `baseURL` → push → Cloudflare `A`/`AAAA`/`www` 紀錄設**灰雲** →
Pages 自訂網域 → 等憑證 → Enforce HTTPS → 用 `dig`/`curl` 驗證 → 加上網域驗證的 `TXT`。
最可能浪費你一個下午的，就是那個橘色雲朵。把它轉灰。
