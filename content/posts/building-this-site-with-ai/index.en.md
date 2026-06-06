---
title: "Building This Site with AI in a Day: From Résumé and Notes to an Automated Deploy"
date: 2026-06-07T01:00:00+08:00
draft: false
summary: "Using Claude Code with Hugo + Blowfish + GitHub Actions to turn a PDF résumé and a pile of scattered tech notes into this bilingual, auto-deployed personal site — in a day. The workflow, the disclosure discipline, and the gotchas."
tags: ["AI", "Hugo", "GitHub Actions", "Claude Code"]
---

This site is itself the product of a day of AI-assisted work (with Claude Code).
The starting point was just a PDF résumé, a few scattered tech notes, and a talk
write-up; the end point is this **bilingual site that goes live the moment I
push**. Here's the workflow, the discipline, and the gotchas — in case you want to
do something similar.

> Related: [A Practical Guide to AI-Assisted Development](../ai-collaboration-guide/).

## The stack

- **Claude Code** — the AI pair, doing most of the mechanical work, translation,
  and consistency checks.
- **Hugo (extended) + the Blowfish theme** — static site generator; Blowfish
  installed as a Hugo Module.
- **GitHub Actions → GitHub Pages** — push to `main` and it builds and deploys
  automatically.

## The workflow

### 1. Positioning first, then the README

The first step wasn't code — it was getting the **positioning** straight. A GitHub
profile README is a business card, so I first decided the audience (job search /
overseas), the angle (backend + applied ML/CV), and the depth (a minimal card,
detail left to the site). I used a "drill down every decision branch" process to
settle each choice before building — so I wouldn't finish and then realize the
direction was wrong.

The result is a minimal card: a one-line tagline, grouped tech badges, three
contact links, and everything else routed to this site.

### 2. Scaffold the site with Hugo + Blowfish

- Install Blowfish via **Hugo Modules** (`hugo mod init` → import the theme →
  `hugo mod get`); upgrading later is just `hugo mod get -u`.
- **Bilingual** via Hugo's built-in i18n: language definitions in
  `languages.toml`, content with `.en.md` / `.zh-tw.md` suffixes.
- Use Blowfish's **profile homepage** layout, surfacing the résumé highlights
  (backend, ML/CV, talks, publications).

### 3. Automate the deploy with GitHub Actions

One workflow: push to `main` → install Hugo extended → `hugo mod get` →
`hugo --gc --minify` → publish to GitHub Pages with the official
`upload-pages-artifact` / `deploy-pages` actions. After that, publishing a post is
just a push, live in a minute or two.

> The one manual step: in the repo, **Settings → Pages → Source = "GitHub
> Actions"**. Set it once and forget it.

### 4. Turning existing docs into posts — discipline is the point

This is the part that most needs a human in the loop; the AI shouldn't decide it
alone. I converted the résumé, a talk write-up, self-study notes, and work notes
into bilingual posts, holding to a few rules:

- **De-identify (most important):** for anything from work, describe only the
  **domain and the technology** — no employer name, no internal service names, no
  business specifics. For example, an internal Feign development guide whose
  example was a sensitive business case got its entire example domain swapped to a
  neutral "e-commerce order," keeping only the generic architectural pattern.
- **Strip private links:** reference links pointing to a personal knowledge base
  are broken for readers and leak internal IDs — remove them all.
- **Strip tracking params:** when pasting course/external links, cut the long
  `gclid` / `utm_*` tail and keep just the clean URL.
- **Bilingual:** every post ships in both English and Chinese.

After each deploy I run a quick audit (e.g. `grep -ri` over the published output
for any leftover sensitive terms) and only call it done once the live site is
clean.

## Gotchas

- **Match the theme's Hugo version:** Blowfish has a tested maximum Hugo version;
  a newer Hugo throws a compatibility warning. Pinning both local and CI to the
  theme's supported version is the simplest fix.
- **Future dates get skipped:** Hugo doesn't build "future" posts by default. When
  I dated a post "today," my machine's timezone (UTC+8) meant that "today at UTC
  midnight" was actually still in the future, so the post wasn't built. Fix: give
  the date an explicit timezone (or use the previous day).
- **Config file naming:** language *definitions* go in `languages.toml`; a filename
  like `languages.en.toml` makes Hugo treat the `.en` as a language qualifier and
  drop the keys. The `.<lang>` suffix is only for `menus.<lang>.toml`.

## Takeaways

AI did the mechanical work — scaffolding, the workflow, translation, formatting,
consistency audits — quickly and reliably, which is what made "live in a day"
realistic. But two things stayed firmly my responsibility: **the positioning
judgment**, and **policing the disclosure boundary** — what can be public and what
must be de-identified. The AI can flag it and do it, but you make the final call.
