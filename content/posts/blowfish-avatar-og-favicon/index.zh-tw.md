---
title: "Blowfish 的頭像、OG 圖與 favicon:三件事,三個參數"
date: 2026-06-07T03:00:00+08:00
draft: false
summary: "個人頭像、社群分享的 OG 圖、瀏覽器分頁的 favicon,看起來像同一張「網站圖」,但在 Blowfish 裡是三個獨立設定、放在三個不同地方。這篇講清楚哪個參數管什麼、為什麼 homepageImage 三者都不是,以及如何用一行指令生出一組 SJ favicon。"
tags: ["Hugo", "Blowfish", "Open Graph", "Favicon"]
---

幫這個網站「補上一張臉」——個人頭像、社群分享預覽圖、favicon——的時候,我本來以為是
同一個設定。並不是。在 **Blowfish** 主題裡,這是**三件獨立的事**,設定在**三個不同的
地方**;而那個名字聽起來最像「首頁圖片」的參數(`homepageImage`),三者都不是它管的。

這篇就是我當初希望手邊先有的那張地圖。

## 陷阱:腦中一個概念,實際三個設定

| 你看到的 | Blowfish 參數 | 放在哪 | 從哪解析 |
| --- | --- | --- | --- |
| Profile **頭像** | `author.image` | `languages.toml`(每語言各一份) | `assets/`,經 `resources.Get` |
| **OG / 社群** 分享圖 | `defaultSocialImage` | `params.toml` | `assets/`,經 `resources.Get` |
| 瀏覽器 **favicon** | *(沒有參數)*——固定檔名 | `static/` | 原樣複製 |

先講結論:**`homepageImage` 是個假線索。** 它是餵給 `background` / `hero` 首頁版型當
橫幅背景用的。如果你的首頁版型是 `profile`(作品集式的著陸頁),頭像**不是**來自
`homepageImage`,而是來自 author image。我是直接讀主題的 partial
`layouts/partials/home/profile.html` 確認的——它只抓
`.Site.Params.Author.image`,別無其他。

## 1. 頭像 —— `author.image`,每語言各設

Blowfish 的 author 掛在每個語言區塊底下,所以圖片是分語言設定的(兩邊指向同一個檔即可):

```toml
# config/_default/languages.toml
[en.params.author]
  name  = "SJ.Wu"
  image = "img/avatar.jpeg"   # 相對於 assets/
  headline = "Software Engineer"

[zh-tw.params.author]
  name  = "SJ.Wu"
  image = "img/avatar.jpeg"
  headline = "軟體工程師"
```

路徑相對於 **`assets/`**,所以檔案放在 `assets/img/avatar.jpeg`。主題會把它丟進 Hugo
的圖片管線——`Fill` 成 288×288 正方、品質 96——所以你只要準備一張大致正方的圖就好,
它會自動以短邊裁切。

## 2. OG 圖 —— `defaultSocialImage`,不是頭像

社群分享預覽(Facebook / LinkedIn / Slack 從你的連結算出來的那張卡片)是**另一張圖**、
**另一個參數**:

```toml
# config/_default/params.toml
defaultSocialImage = "img/og.png"   # 1.91:1,相對於 assets/
```

關於 Blowfish 怎麼挑 OG 圖(出自 `layouts/partials/head.html`),有兩件事值得知道:

1. **單頁優先。** 如果某個 page bundle 裡有一個名稱含 `*featured*`、`*cover*` 或
   `*thumbnail*` 的資源,那張圖就成為該頁的 `og:image`。`defaultSocialImage` 只是
   **沒有這種資源時的後備**——這正好就是首頁與多數文章想要的行為。
2. **比例是 1.91:1。** 社群標準是 **1200×630**。一開始就照這個比例做,事後再裁只是浪費
   左右兩側。

一個實用的小技巧:要**生成**一張能對齊風格化頭像的 OG 圖,把頭像當參考圖丟進生圖模型,
提示詞要求「**1200×630 橫幅、同樣的扁平插畫風、人物在左、右側留白**」,而且**圖裡不要
燒上任何文字**。生圖模型寫小字幾乎都會糊;真要標題,事後在 Figma/Canva 疊上去,才能保持
銳利。

## 3. favicon —— 它**不會**順便跟著換

這點最容易讓人意外:設定 `author.image` 跟 `defaultSocialImage`,對 favicon **毫無
影響**。Blowfish 是從 **`static/`** 底下一組固定檔名提供 favicon,而你放進去的同名檔會
**蓋過**主題預設:

```
static/favicon.ico          # 多尺寸:48 / 32 / 16
static/favicon-32x32.png
static/favicon-16x16.png
static/apple-touch-icon.png # 180×180
```

我是用 checksum 比對確認站台還停在主題預設上的——build 出來的 `public/favicon.ico` 跟
主題內建檔的 md5 **一模一樣**。在你把自己的檔案丟進 `static/` 之前,出去的都是 Blowfish
的圖示。

### 用一行指令生出一組「SJ」favicon

半身頭像縮到 16×16 根本看不清,所以 favicon 我改用簡單的字母標誌。ImageMagick 從一張母圖
就能產出整組:

```bash
FONT=/usr/share/fonts/truetype/liberation/LiberationSans-Bold.ttf

# 512px 母圖:圓角方形 + 置中的「SJ」
magick -size 512x512 xc:none \
  -fill "#1f2a37" -draw "roundrectangle 0,0 511,511 96,96" \
  -font "$FONT" -fill "#f4f1ea" -gravity center -pointsize 250 \
  -annotate +0-10 "SJ" master.png

# 衍生尺寸
magick master.png -resize 180x180 static/apple-touch-icon.png
magick master.png -resize 32x32   static/favicon-32x32.png
magick master.png -resize 16x16   static/favicon-16x16.png
magick master.png -define icon:auto-resize=48,32,16 static/favicon.ico
```

顏色直接從頭像取樣——深藍灰當底(髮色/輪廓色)、暖白當字(白 T / 背景)——讓 favicon 跟
頭像讀起來是同一套品牌。

## `assets/` vs `static/` —— 為什麼這個區分重要

這正是三個設定各自放在不同地方的安靜理由:

- **`assets/`** 是 Hugo 的**處理**管線。`resources.Get` 從這裡讀,主題可以縮放、裁切、
  加指紋(這就是為什麼 build 出來的頭像網址長得像 `avatar_hu_9d0…jpeg`)。頭像跟 OG 圖
  放這裡,是因為 Blowfish 要**轉換**它們。
- **`static/`** 是**原封不動**複製到站台根目錄,完全不處理。favicon 必須保持精確的檔名
  與位元組內容,所以該放這裡。

把 favicon 放進 `assets/`,主題會找不到;把頭像放進 `static/`,你就失去自動縮放。檔案要
配對到對的資料夾。

## 驗證線上站台

GitHub Actions 部署完之後,我不信 build、改看實際產出——還帶了一個破快取的查詢參數,
確保看到的是**當下**的資源:

```bash
# OG meta 標籤,以及圖片是否真的有提供
curl -s https://sj-wu.com/ | grep -oE '<meta property="og:image"[^>]*>'
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' https://sj-wu.com/img/og.png

# favicon 已不再是主題預設
curl -s "https://sj-wu.com/apple-touch-icon.png?nocache=$RANDOM" -o live.png
```

三者都回 `200`,OG 標籤指向正確網址,下載下來的 favicon 就是新的 SJ 標誌。

## 那些讓它「看起來壞掉」的快取

伺服器端全都正確,卻仍然看起來是舊的——因為你前面卡著兩層快取:

- **社群平台會快取 OG 預覽。** Facebook / LinkedIn 會留著舊卡片,直到你在它們的
  post-inspector / 分享偵錯工具裡重新抓取。
- **瀏覽器對 favicon 的快取特別頑固**——常常連一般重整都不會更新。在下任何「壞掉」的
  結論前,先用無痕視窗,或在網址後加 `?v=2` 強制重抓。

## 重點整理

- 頭像、OG 圖、favicon 是**三個設定、三個地方**——別預期改一個會連動其他。
- 在 `profile` 版型下,`homepageImage` **不是**頭像;`author.image` 才是。
- `defaultSocialImage` 只是 OG 的**後備**——單頁的 `*featured*` / `*cover*` /
  `*thumbnail*` 資源會蓋過它。
- favicon 以固定檔名住在 **`static/`**,必須明確替換,沒有別的設定會動到它。
- `assets/` = 會被處理(縮放/裁切/指紋);`static/` = 原樣輸出。檔案放在它該被怎麼處理的
  地方。
- 當某樣東西「沒更新」,先懷疑**快取**,再懷疑設定。
