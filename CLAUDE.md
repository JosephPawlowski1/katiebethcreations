# katiebethcreations

Static one-page link site for KatiBeth Creative (formerly KatieBeth Creations). Plain HTML/CSS, no build step.

Live at **https://katibethcreative.com** (canonical since 2026-10-10). The old
`katiebethcreations.com` 301-redirects here. The repo name is still `katiebethcreations`.

## ⚠️ Migration in progress: GitHub Pages → Cloudflare Pages (started 2026-10-11)

The site is moving to **Cloudflare Pages**, with this repo connected over Git, so Cloudflare
builds and serves the files and GitHub only holds the source. The rest of this file describes
the **target** setup. Until every box below is ticked, the live site is still served by GitHub
Pages behind Cloudflare.

- [ ] Connect the GitHub repo in Cloudflare (Workers & Pages → Create → Pages → Import Git
      repository). Needs the account owner to approve Cloudflare's GitHub app.
- [ ] Create the Pages project: production branch `main`, framework none, build command empty,
      output directory `/`. Expected URL `katibethcreative.pages.dev` (Cloudflare adds a suffix
      if the name is taken).
- [ ] Check the `*.pages.dev` deploy matches the live site.
- [ ] Add custom domains `katibethcreative.com` and `www.katibethcreative.com` to the project.
      This replaces the four GitHub A records and the `www` CNAME (see DNS below).
- [ ] Check `https://katibethcreative.com` is served by Pages (`curl -sI` shows no
      `x-github-request-id` header).
- [ ] Turn off GitHub Pages (`gh api -X DELETE /repos/JosephPawlowski1/katiebethcreations/pages`)
      and delete `CNAME`.
- [ ] Set SSL mode to `Full (strict)` (the GitHub certificate problem no longer applies).
- [ ] Remove this section.

## Architecture

Two providers, each doing one job:

| Layer | Provider | Notes |
|-------|----------|-------|
| Registrar | **Squarespace** | Registration and renewal only |
| DNS, TLS, CDN, **hosting** | **Cloudflare** | Pages project built from this repo; Universal SSL |
| Source | **GitHub** | `JosephPawlowski1/katiebethcreations`; pushes trigger Cloudflare builds |

Traffic path: visitor → Cloudflare edge → Cloudflare Pages. Nothing visitor-facing touches GitHub.

Squarespace no longer serves DNS for these domains. **Its DNS panel still shows stale records —
ignore it.** Editing records there has no effect.

## Files

- `index.html` — the page (logo, four social link cards, footer)
- `styles.css` — palette sampled from the logos
- `logo.png` — stacked "KatiBeth Creative" wordmark on a pale star (hero)
- `icon.png` (192px), `favicon.png` (32px), `apple-touch-icon.png` (180px, white bg) — the "KC"
  monogram with its star, minus the tiny wordmark (unreadable at favicon size). Also the footer badge.
- `CNAME` — leftover from GitHub Pages. Keep it until GitHub Pages is turned off (see the
  migration checklist), then delete it. Cloudflare Pages ignores it.

The **whole repo root** is published as-is, `CLAUDE.md` included. It's harmless, but don't commit
anything here that shouldn't be public.

## Deploying

**Deploy = push to `main`.** Cloudflare Pages picks up the push and publishes it, usually in
under a minute. There's no build step: the build command is empty and the output directory is
the repo root.

```bash
git add -A && git commit -m "..." && git push
```

To check a deploy: Cloudflare dashboard → Workers & Pages → the project → Deployments. Every push
to another branch gets its own preview URL (`<branch>.<project>.pages.dev`), which is handy for
trying changes before they go live.

The custom domains are set in the Pages project (Custom domains tab), not in the repo. To change
the canonical domain, add the new one there and make the old one redirect.

Pages purges its cache on every deploy. If an old version still shows up, it's almost always the
browser cache or a local DNS cache (see Gotchas), not Cloudflare.

## DNS (Cloudflare)

Account `f85a8fe3ad515231bebea3cb5ca5ffbb`, all zones on the Free plan. Every zone in this account
gets the same nameserver pair, set at Squarespace (DNS → Domain Nameservers → Use Custom
Nameservers; Squarespace forces DNSSEC off when you do this):

```
dax.ns.cloudflare.com
jacqueline.ns.cloudflare.com
```

| Zone | Role | Config |
|------|------|--------|
| `katibethcreative.com` | **Canonical site** | Records below, SSL `Full`, Always Use HTTPS |
| `katibethcreative.net` | Alias | Redirect Rule: all requests → 301 `https://katibethcreative.com` + path |
| `katibethcreative.store` | Alias | Same Redirect Rule |
| `katiebethcreations.com` (zone `3818b36ba3fcc78f8968ffae9753beda`) | Old name | Same Redirect Rule (deployed 2026-10-11); old GitHub records left in place |

The redirect-only zones keep the proxied Squarespace parking records from Cloudflare's import scan.
They are never reached (the Redirect Rule fires at the edge first) but a **proxied** `@` and `www`
must exist, or the rule has nothing to attach to.

`katibethcreative.com` records. Pages creates the `@` and `www` CNAMEs itself when the custom
domains are added; the apex works through Cloudflare's CNAME flattening:

| Type  | Name           | Value                                          | Proxy |
|-------|----------------|------------------------------------------------|-------|
| CNAME | @              | `<project>.pages.dev`                          | on    |
| CNAME | www            | `<project>.pages.dev`                          | on    |
| TXT   | @              | `v=spf1 -all`                                  | —     |
| TXT   | _dmarc         | `v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s` | —   |
| TXT   | _domainkey     | `v=DKIM1; p=`                                  | —     |
| CNAME | _domainconnect | _domainconnect.domains.squarespace.com         | on    |

Before the migration, `@` was four A records (`185.199.108–111.153`) and `www` was a CNAME to
`josephpawlowski1.github.io`. If those come back, the site is on GitHub Pages again.

No MX records — there is no email on this domain. The three TXT records are the SPF/DMARC/DKIM
policy and must survive any future move.

### SSL settings that matter

- **Encryption mode: `Full (strict)`** once the move is finished. Pages serves a valid
  certificate, so strict validation works. While GitHub Pages is still the origin it **must**
  stay at plain `Full`: GitHub's `*.github.io` certificate never matches this domain, and Strict
  breaks the site.
- **Always Use HTTPS: on.** `http://` → 301 → `https://`, apex and www.
- Universal SSL covers each zone's apex and `*.` wildcard. It is issued only once the zone goes
  **active**; until then HTTPS fails with a handshake error, not a mismatch.
- DNSSEC is **off** — Squarespace required disabling it to change nameservers.

### Verifying

`dig` against a public resolver; the apex should return **Cloudflare** IPs (104.x / 172.67.x):

```bash
dig +short katibethcreative.com A @1.1.1.1
curl -sI https://katibethcreative.com/ | head -3                       # 200, server: cloudflare
curl -sI https://katibethcreative.com/ | grep -ci x-github             # 0 once on Pages
curl -sI https://katibethcreative.net/x | grep -i location             # https://katibethcreative.com/x
curl -sI https://katiebethcreations.com/ | grep -i location            # https://katibethcreative.com/
```

## Alias domains

All registered at Squarespace. The three newest (`katibethcreative.net`, `.store`, and the old
`katiebethcreations.com`) redirect via **Cloudflare Redirect Rules** — see the DNS table above.

| Domain                   | Renews       |
|--------------------------|--------------|
| katibethcreative.com     | Oct 10, 2027 (canonical) |
| katibethcreative.net     | Oct 10, 2027 |
| katibethcreative.store   | Oct 10, 2027 |
| katiebethcreations.com   | Aug 8, 2027  |

Four older domains still **301-redirect** to `https://katiebethcreations.com` (which then redirects
again to the new name — two hops, harmless) via Squarespace *Domain Forwarding* (Website tab → Domain
Forwarding), each covering root (`@`) and `www`, path forwarding off:

| Domain                   | Renews       |
|--------------------------|--------------|
| creationsbykatiebeth.com | Aug 16, 2027 |
| creationsbykatiebeth.net | Aug 16, 2027 |
| katiebethcreations.net   | Aug 23, 2027 |
| katiebethcreations.store | Aug 23, 2027 |

These still use **Squarespace** nameservers — only katiebethcreations.com moved to Cloudflare.
Their forwarding rules are unaffected by that move.

Note `creationsbykatiebeth.*` is legacy — it predates the rename and matches no current handle.

To move one of these to Cloudflare, add it as a zone, switch its nameservers at Squarespace, and
give it the same Redirect Rule as `.net`. Redirect Rules apply in seconds, unlike Squarespace's
24–48 hours.

## History: why GitHub Pages was dropped

The site started on GitHub Pages with DNS at Squarespace. **GitHub never issued a TLS certificate
for the custom domain.** It sat at `state: null` (never requested, not merely queued) for two days,
even though DNS, CAA and the ACME challenge path all checked out. Clearing and re-adding the domain
moved it to `authorization_created`, where it stalled again.

Cloudflare was then put in front (2026-09-28) to terminate TLS, with GitHub Pages kept as the
origin. That worked, but it meant three providers, an SSL mode pinned to `Full` around GitHub's
mismatched certificate, a `CNAME` file GitHub could rewrite behind your back, and the
one-custom-domain-per-repo limit. Moving the files to Cloudflare Pages (2026-10-11) removes all
of that.

**Don't move the site back to GitHub Pages without solving the certificate problem first.**

## Gotchas

- **Squarespace re-prompts for Google sign-in** roughly once per domain while editing DNS or
  forwarding. Records saved before the prompt are kept.
- **Squarespace forms ignore programmatically-set values.** Setting an input's value directly
  leaves their React state empty, so Save silently does nothing and no rule is created. Type real
  keystrokes instead, then reload to confirm the rule actually exists.
- **Squarespace forwarding rules take 24–48 hours** to start working, per their own notice.
- **A local DNS cache will lie to you.** macOS `dscacheutil` held the old GitHub IPs for hours
  after the switch, making the site look broken locally while it worked everywhere else. Check
  `dig @1.1.1.1` before believing a failure, and use
  `curl --resolve host:443:<ip>` to bypass the cache.
- **The Squarespace website trial** (separate from the domain) expires **Oct 10, 2026**. It was
  never used. Cancel it so it does not convert to a paid plan; do not cancel the domain.

## Conventions

- Brand colors (from the logos): pink `#bd4a77`, olive `#919d3d`, pale sage `#e9ebd8`. They are
  CSS custom properties on `:root`; reuse them rather than introducing new hex values. `--olive` is
  `#8e9a3b`, a hair darker than the logo, so white icons on it clear WCAG 3:1. Olive is too light
  for small text — use `--olive-deep` or `--ink-soft` for that.
- The logo PNGs have **real transparency**. The designer's originals ship on opaque white; they
  were keyed by projecting each pixel onto the white→brand-color line. If new art arrives on
  white, re-key it the same way rather than reaching for `mix-blend-mode: multiply`.
- The pale sage star nearly vanishes on a sage background, which is why the card stays **white**
  and only the page behind it is tinted.
- Hover lift on `.link` is wrapped in `@media (hover: hover)` — without it the state sticks
  after a tap on touchscreens.
- Page must stay usable at 320px wide with no horizontal scroll.

## Links on the page

| Channel   | URL                                          |
|-----------|----------------------------------------------|
| YouTube   | https://www.youtube.com/@katiebethcreations   |
| TikTok    | https://www.tiktok.com/@katibethcreative      |
| Instagram | https://www.instagram.com/katibethcreative    |
| Etsy      | https://katiebethworkshop.etsy.com            |

TikTok and Instagram moved to `@katibethcreative` on 2026-10-11. YouTube is still
`@katiebethcreations`, and the Etsy shop is **katiebethworkshop** — neither was renamed.
