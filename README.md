# sj-wu.github.io

Personal site of SJ.Wu — built with [Hugo](https://gohugo.io/) and the
[Blowfish](https://blowfish.page/) theme, deployed to GitHub Pages via GitHub
Actions. Bilingual: English (default) + Traditional Chinese.

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

## Custom domain (later)

To attach a custom domain:

1. Add a `static/CNAME` file containing the domain (e.g. `www.example.com`).
2. Set the domain in Settings → Pages → Custom domain (creates the DNS check).
3. Update `baseURL` in `config/_default/hugo.toml`.
