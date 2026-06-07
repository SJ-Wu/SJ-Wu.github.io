---
title: "Moving a GitHub Pages Site to a Custom Domain (Cloudflare Registrar)"
date: 2026-06-07T02:00:00+08:00
draft: false
summary: "Pointing a freshly registered apex domain at a GitHub Pages site, end to end: the repo changes, the Cloudflare DNS records, the one proxy setting that breaks HTTPS, and how to verify the cutover — plus domain verification to stop anyone hijacking the name."
tags: ["GitHub Pages", "Cloudflare", "DNS", "Hugo"]
---

A GitHub Pages site starts life at `<user>.github.io`. Attaching your own domain is
straightforward once you know the order, but a couple of steps have sharp edges —
especially if your registrar is Cloudflare, where one default setting will silently
stop HTTPS from ever working. Here's the whole flow, written with `example.com` so
you can drop in your own domain.

> Related: [Building This Site with AI in a Day](../building-this-site-with-ai/).

## The decision: apex vs www

You have to pick which form is **canonical** — the one the address bar settles on.

- **apex** (`example.com`) — shortest, cleanest. The cost is DNS: an apex can't be a
  `CNAME`, so you point it at GitHub's four IPs with `A` (and `AAAA` for IPv6).
- **www** (`www.example.com`) — DNS is one tidy `CNAME`, but the address is longer.

Going apex-canonical and letting `www` redirect to it is the common choice. GitHub
handles that redirect automatically once both are configured, so both addresses
work and the bar shows the bare apex.

## Step 1 — repo changes

Two things live in the repo, independent of DNS.

**A `CNAME` file** tells GitHub which domain this site answers to. With Hugo, drop
it in `static/` so it's copied verbatim into the build output:

```
static/CNAME      →  example.com
```

The file holds the **canonical** host only — the apex, not `www`.

**The `baseURL`.** Update it so generated absolute links, the sitemap, and RSS use
the new host:

```toml
# config/_default/hugo.toml
baseURL = "https://example.com/"
```

If you build in CI, check whether the workflow overrides `baseURL`. A common
Pages workflow does:

```yaml
run: hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
```

That `base_url` output becomes your custom domain *after* you set it in Pages, so
no workflow change is needed — but the value in `hugo.toml` still matters for local
builds, so update it anyway.

Push these, and the deploy publishes the `CNAME` into the live output.

## Step 2 — Cloudflare DNS

A domain registered with Cloudflare is already using Cloudflare DNS, so this is
just the **DNS → Records** tab. For an apex-canonical setup:

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

(These are GitHub's published Pages IPs — worth confirming against GitHub's docs in
case they ever change.)

### The gotcha: turn the orange cloud OFF

This is the one that bites. Cloudflare proxies records by default — the **orange
cloud**. With proxying on, GitHub can't complete the HTTP challenge it uses to
issue a Let's Encrypt certificate, so HTTPS never provisions; and if you force
Cloudflare's own TLS in front, you get a redirect loop. **Set every record to
"DNS only" — the grey cloud.** Click the cloud icon until it's grey.

Keep it grey at least until the GitHub certificate is issued and "Enforce HTTPS"
works. If you later *want* Cloudflare's proxy/CDN, you can turn it on — but then
set Cloudflare's **SSL/TLS mode to "Full"** (never "Flexible", which loops). The
simplest, most reliable setup is to just leave it grey.

## Step 3 — GitHub Pages + HTTPS

In the site repo: **Settings → Pages**.

1. **Custom domain** → enter `example.com` → **Save**. GitHub runs a DNS check;
   once your records have propagated it shows a green check.
2. Wait for the certificate. This takes minutes to tens of minutes — the
   **Enforce HTTPS** box is greyed out until the cert is ready.
3. Tick **Enforce HTTPS**. Now every `http://` hit 301s to `https://`.

Tip: push the `CNAME` (Step 1) *before* you fill in the custom domain, so the
domain is already present in the deployed output when the DNS check runs.

## Step 4 — verify the cutover

DNS first:

```bash
dig example.com +short          # expect the four GitHub IPs
dig www.example.com +short      # expect: <user>.github.io. then the IPs
```

Then every entry point should land on the canonical HTTPS URL:

```bash
curl -sI https://example.com        | head -1   # HTTP/2 200
curl -sI https://www.example.com    | head -1   # 301 -> https://example.com/
curl -sI http://example.com         | head -1   # 301 -> https (Enforce HTTPS)
curl -sI https://<user>.github.io   | head -1   # 301 -> https://example.com/
```

Four entry points — apex, www, plain http, and the old `github.io` — all funnel to
the canonical HTTPS URL. That's the cutover done.

## Step 5 — domain verification (stop hijacking)

There's a quieter risk worth closing. If you ever remove the domain from this repo
without removing the DNS records, someone else could claim it on *their* Pages
site. GitHub's **domain verification** prevents that: a verified domain can only be
used by your account.

It lives in your **account** settings, not the repo: **Settings → Pages →**
**"Add a domain"**. GitHub gives you a `TXT` record like:

```
_github-pages-challenge-<user>.example.com   TXT   <token>
```

Add that `TXT` in Cloudflare (grey cloud — TXT isn't proxied anyway), then click
**Verify**. Confirm it resolves with:

```bash
dig _github-pages-challenge-<user>.example.com TXT +short
```

Once verified, the name is locked to your account. You can leave the `TXT` record
in place.

## The order, in one breath

Repo `CNAME` + `baseURL` → push → Cloudflare `A`/`AAAA`/`www` records **grey
cloud** → Pages custom domain → wait for cert → Enforce HTTPS → verify with
`dig`/`curl` → add the domain-verification `TXT`. The single thing most likely to
waste your afternoon is the orange cloud. Turn it grey.
