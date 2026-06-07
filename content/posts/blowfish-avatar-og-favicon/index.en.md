---
title: "Avatar, OG Image, and Favicon in Blowfish: Three Things, Three Params"
date: 2026-06-07T03:00:00+08:00
draft: false
summary: "The profile avatar, the social-share OG image, and the browser favicon look like one 'site picture' — but in Blowfish they are three independent settings in three different places. Here's which param does what, why homepageImage is none of them, and how I generated an SJ favicon from one command."
tags: ["Hugo", "Blowfish", "Open Graph", "Favicon"]
---

When I finally gave this site a face — a profile avatar, a social-share preview
image, and a favicon — I assumed it would be one setting. It is not. In the
**Blowfish** theme these are *three independent things*, configured in *three
different places*, and the one param whose name sounds like "the homepage picture"
(`homepageImage`) controls none of them.

This post is the map I wish I'd had.

## The trap: one mental model, three real settings

| What you see | Blowfish param | Lives in | Resolved from |
| --- | --- | --- | --- |
| Profile **avatar** | `author.image` | `languages.toml` (per language) | `assets/` via `resources.Get` |
| **OG / social** share image | `defaultSocialImage` | `params.toml` | `assets/` via `resources.Get` |
| Browser **favicon** | *(no param)* — fixed filenames | `static/` | copied verbatim |

The lesson up front: **`homepageImage` is a red herring.** It feeds the
`background` / `hero` homepage layouts as a banner image. If your homepage layout
is `profile` (the portfolio-style landing), the avatar does *not* come from
`homepageImage` — it comes from the author image. I confirmed this by reading the
theme's own partial, `layouts/partials/home/profile.html`, which pulls
`.Site.Params.Author.image` and nothing else.

## 1. The avatar — `author.image`, per language

Blowfish's author lives under each language block, so the image is set per
language (point both at the same file):

```toml
# config/_default/languages.toml
[en.params.author]
  name  = "SJ.Wu"
  image = "img/avatar.jpeg"   # relative to assets/
  headline = "Software Engineer"

[zh-tw.params.author]
  name  = "SJ.Wu"
  image = "img/avatar.jpeg"
  headline = "軟體工程師"
```

The path is relative to **`assets/`**, so the file lives at
`assets/img/avatar.jpeg`. The theme runs it through Hugo's image pipeline —
`Fill` to a 288×288 square, quality 96 — so a roughly-square source is all you
need; it crops to the short side automatically.

## 2. The OG image — `defaultSocialImage`, not the avatar

The social-share preview (the card Facebook / LinkedIn / Slack render from your
link) is a *separate* image and a separate param:

```toml
# config/_default/params.toml
defaultSocialImage = "img/og.png"   # 1.91:1, relative to assets/
```

Two things worth knowing about how Blowfish picks the OG image (from
`layouts/partials/head.html`):

1. **Per-page wins first.** If a page bundle contains a resource named
   `*featured*`, `*cover*`, or `*thumbnail*`, that image becomes the page's
   `og:image`. `defaultSocialImage` is only the **fallback** when no such resource
   exists — which is exactly what you want for the homepage and most posts.
2. **Aspect ratio is 1.91:1.** The social standard is **1200×630**. Make the
   image that shape from the start; cropping after the fact wastes the sides.

A practical tip for *generating* an OG image that matches a stylized avatar: feed
the avatar into an image model as a reference and prompt for a **1200×630 banner,
same flat-illustration style, character on the left, negative space on the
right** — and **bake no text into the image**. Generators mangle small text; if
you want a title, overlay it later in Figma/Canva where it stays crisp.

## 3. The favicon — it does *not* come along for free

This is the one that surprises people: setting `author.image` and
`defaultSocialImage` changes **nothing** about the favicon. Blowfish serves the
favicon from a fixed set of filenames in **`static/`**, and a file you place there
shadows the theme's default:

```
static/favicon.ico          # multi-size: 48 / 32 / 16
static/favicon-32x32.png
static/favicon-16x16.png
static/apple-touch-icon.png # 180×180
```

I verified the site was still on the theme default by comparing checksums — the
built `public/favicon.ico` had the *same* md5 as the theme's bundled file. Until
you drop your own files into `static/`, you're shipping Blowfish's icon.

### Generating an "SJ" favicon from one command

A half-body avatar is unreadable at 16×16, so for the favicon I made a simple
letter mark instead. ImageMagick produces the whole set from a single master:

```bash
FONT=/usr/share/fonts/truetype/liberation/LiberationSans-Bold.ttf

# 512px master: rounded square + centered "SJ"
magick -size 512x512 xc:none \
  -fill "#1f2a37" -draw "roundrectangle 0,0 511,511 96,96" \
  -font "$FONT" -fill "#f4f1ea" -gravity center -pointsize 250 \
  -annotate +0-10 "SJ" master.png

# derivatives
magick master.png -resize 180x180 static/apple-touch-icon.png
magick master.png -resize 32x32   static/favicon-32x32.png
magick master.png -resize 16x16   static/favicon-16x16.png
magick master.png -define icon:auto-resize=48,32,16 static/favicon.ico
```

The colours are pulled straight from the avatar — deep slate-navy for the
background (the hair/outline colour), warm off-white for the letters (the tee /
backdrop) — so the favicon and the avatar read as one brand.

## `assets/` vs `static/` — why the split matters

This is the quiet reason the three settings live where they do:

- **`assets/`** is Hugo's *processing* pipeline. `resources.Get` reads from here,
  and the theme can resize, crop, and fingerprint the output (that's why the
  built avatar URL looks like `avatar_hu_9d0…jpeg`). The avatar and OG image go
  here because Blowfish *transforms* them.
- **`static/`** is copied to the site root **verbatim**, no processing. Favicons
  must keep exact filenames and byte content, so they belong here.

Put a favicon in `assets/` and the theme won't find it; put the avatar in
`static/` and you lose the automatic resizing. Match the file to the folder.

## Verifying the live site

After the GitHub Actions deploy, I checked the real output rather than trusting
the build — including a cache-busting query so I saw the *current* asset:

```bash
# OG meta tag and that the image actually serves
curl -s https://sj-wu.com/ | grep -oE '<meta property="og:image"[^>]*>'
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' https://sj-wu.com/img/og.png

# favicon is no longer the theme default
curl -s "https://sj-wu.com/apple-touch-icon.png?nocache=$RANDOM" -o live.png
```

All three returned `200`, the OG tag pointed at the right URL, and the downloaded
favicon was the new SJ mark.

## The caches that make it *look* broken

Everything can be correct on the server and still look stale, because two caches
sit in front of you:

- **Social platforms cache OG previews.** Facebook / LinkedIn keep an old card
  until you re-scrape it in their post-inspector / sharing-debugger.
- **Browsers cache favicons aggressively** — often beyond a normal refresh. Use a
  private window, or append `?v=2` to force a re-fetch, before concluding anything
  is wrong.

## Takeaways

- Avatar, OG image, and favicon are **three settings in three places** — don't
  expect one to imply the others.
- `homepageImage` is **not** the avatar on a `profile` layout; `author.image` is.
- `defaultSocialImage` is only the OG **fallback** — page-level `*featured*` /
  `*cover*` / `*thumbnail*` resources override it.
- Favicons live in **`static/`** as fixed filenames and must be replaced
  explicitly; nothing else touches them.
- `assets/` = processed (resize/crop/fingerprint); `static/` = verbatim. Put each
  file where its handling lives.
- When something "didn't update," suspect the **cache** before the config.
