# sj-wu.github.io

Personal site of SJ.Wu, live at **[sj-wu.com](https://sj-wu.com)** — built with
[Hugo](https://gohugo.io/) and the [Blowfish](https://blowfish.page/) theme,
deployed to GitHub Pages via GitHub Actions. Bilingual: English (default) +
Traditional Chinese.

## Local development

Requires **Hugo extended** (pinned to the version in `.github/workflows/hugo.yml`)
and **Go** (for Hugo Modules).

```bash
hugo mod get          # fetch / update the Blowfish theme module
hugo server           # http://localhost:1313
hugo --gc --minify    # production build into ./public
```

## Structure

- `config/_default/` — Hugo + Blowfish configuration (split config).
  Languages live in `languages.toml`; per-language nav in `menus.<lang>.toml`.
- `content/` — bilingual content via `.en.md` / `.zh-tw.md` suffixes
  (`_index`, `about`, `projects`, `posts`).
- `.github/workflows/hugo.yml` — build + deploy to GitHub Pages.

## Deployment

Push to `main` triggers the Actions workflow, which builds with Hugo and deploys
to GitHub Pages. **Pages source must be set to "GitHub Actions"** (Settings →
Pages → Build and deployment → Source).

## Custom domain

The site is served at the apex `sj-wu.com` (canonical), with `www` and the old
`sj-wu.github.io` redirecting to it. Configured via:

1. `static/CNAME` holds the canonical host (`sj-wu.com`).
2. `baseURL` in `config/_default/hugo.toml` points at `https://sj-wu.com/`.
3. Domain set in Settings → Pages → Custom domain, with **Enforce HTTPS** on.
4. DNS lives in Cloudflare: apex `A`/`AAAA` → GitHub Pages IPs, `www` `CNAME` →
   `sj-wu.github.io`, all records **DNS only (grey cloud)**.

See the post *Moving a GitHub Pages Site to a Custom Domain* for the full walkthrough.
