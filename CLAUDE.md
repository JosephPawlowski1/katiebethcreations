# katiebethcreations

Static one-page link site for KatieBeth Creations. Plain HTML/CSS, no build step.

## Files

- `index.html` — the page (logo, four social link cards, footer)
- `styles.css` — palette sampled from the logo
- `logo.webp` — logo artwork
- `CNAME` — **required by GitHub Pages**; holds `katiebethcreations.com`. Do not delete.

## Hosting

Hosted free on **GitHub Pages** from [JosephPawlowski1/katiebethcreations](https://github.com/JosephPawlowski1/katiebethcreations),
branch `main`, folder root. Public repo (required for Pages on a free account).

**Deploy = push to `main`.** Pages rebuilds automatically; no CI, no build step.

```bash
git add -A && git commit -m "..." && git push
```

Check a deploy:

```bash
gh api /repos/JosephPawlowski1/katiebethcreations/pages/builds/latest --jq '.status, .error.message'
```

## DNS

Registrar and DNS are **Squarespace**; hosting is **GitHub**. The two are separate — the
Squarespace *website* was never used and its trial was left to lapse (expired Oct 10, 2026).

Custom records at Squarespace (DNS Settings > Custom records):

| Type  | Name | Data                       |
|-------|------|----------------------------|
| A     | @    | 185.199.108.153            |
| A     | @    | 185.199.109.153            |
| A     | @    | 185.199.110.153            |
| A     | @    | 185.199.111.153            |
| CNAME | www  | josephpawlowski1.github.io |

The **"Squarespace Defaults" preset was deleted** — its `A @ -> 198.185.x.x` records pointed at
the Squarespace site and conflict with GitHub's. If it ever reappears (Squarespace re-adds it when
you reconnect a site), the site breaks; delete it again.

Email TXT records (SPF/DMARC/DKIM) and the `_domainconnect` CNAME are unrelated — leave them.

## Alias domains

Four other domains are owned (all registered at Squarespace) and **301-redirect** to
`https://katiebethcreations.com` via Squarespace *Domain Forwarding* (Website tab > Domain
Forwarding), each covering both the root (`@`) and `www`, with path forwarding off:

| Domain                   | Renews       |
|--------------------------|--------------|
| creationsbykatiebeth.com | Aug 16, 2027 |
| creationsbykatiebeth.net | Aug 16, 2027 |
| katiebethcreations.net   | Aug 23, 2027 |
| katiebethcreations.store | Aug 23, 2027 |

`katiebethcreations.com` itself renews Aug 23, 2027 and is the canonical domain — it is the only
one pointed at GitHub with A records.

**Do not point aliases at GitHub's IPs.** GitHub Pages serves exactly one custom domain per repo
(whatever is in `CNAME`); any other hostname resolving to its IPs gets a 404. Forwarding is the
only correct mechanism for aliases.

Note `creationsbykatiebeth.*` is legacy — it predates the rename and matches no current handle.

### Gotchas

- **Squarespace DNS re-prompts for Google sign-in** partway through editing records, roughly once
  per domain. Expect to re-authenticate mid-session; records saved before the prompt are kept.
- **Squarespace forms ignore programmatically-set values.** Setting an input's value directly
  (e.g. via JS or a form-fill tool) leaves their React state empty, so Save silently does nothing
  and no rule is created. Click the field and type real keystrokes instead, then confirm the rule
  appears after a reload.
- **Forwarding rules take 24-48 hours** to start working, per Squarespace's own notice.
- **HTTPS is issued by GitHub only after DNS resolves to it.** After any DNS change, allow
  15-60 min, then enable *Enforce HTTPS* in repo Settings > Pages.

## Conventions

- The logo art has an **opaque white background**, not transparency. `styles.css` uses
  `mix-blend-mode: multiply` on `.logo` to drop the white into the cream card. This works only
  because the card background is a flat color — on a gradient or photo, export the logo with a
  real alpha channel instead.
- Colors are CSS custom properties on `:root`, sampled from the logo. Reuse them rather than
  introducing new hex values.
- Hover lift on `.link` is wrapped in `@media (hover: hover)` — without it the state sticks
  after a tap on touchscreens.
- Page must stay usable at 320px wide with no horizontal scroll.

## Links on the page

| Channel   | URL                                          |
|-----------|----------------------------------------------|
| YouTube   | https://www.youtube.com/@katiebethcreations   |
| TikTok    | https://www.tiktok.com/@katiebethcreations    |
| Instagram | https://www.instagram.com/katiebethcreations/ |
| Etsy      | https://katiebethworkshop.etsy.com            |

Note the Etsy shop is **katiebethworkshop**, not katiebethcreations.
