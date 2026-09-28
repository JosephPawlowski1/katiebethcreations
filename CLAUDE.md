# katiebethcreations

Static one-page link site for KatieBeth Creations. Plain HTML/CSS, no build step.

Live at **https://katiebethcreations.com**.

## Architecture

Three providers, each doing one job. Keeping them straight is the key to not breaking this:

| Layer | Provider | Notes |
|-------|----------|-------|
| Registrar | **Squarespace** | Owns the registration + renewal only |
| DNS | **Cloudflare** | Authoritative since 2026-09-28 |
| TLS + CDN | **Cloudflare** | Universal SSL, proxies all traffic |
| Origin (files) | **GitHub Pages** | Serves the repo; never visitor-facing directly |

Traffic path: visitor → Cloudflare edge (TLS terminates here) → GitHub Pages origin.

Squarespace no longer serves DNS for this domain. **Its DNS panel still shows stale records —
ignore it.** Editing records there has no effect.

## Files

- `index.html` — the page (logo, four social link cards, footer)
- `styles.css` — palette sampled from the logo
- `logo.webp` — logo artwork
- `CNAME` — holds `katiebethcreations.com`; GitHub Pages needs it to route the host header to
  this repo. Still required even though Cloudflare fronts the site. Do not delete.

## Deploying

**Deploy = push to `main`.** GitHub Pages rebuilds automatically; no CI, no build step.

```bash
git add -A && git commit -m "..." && git push
```

Check a deploy:

```bash
gh api /repos/JosephPawlowski1/katiebethcreations/pages/builds/latest --jq '.status, .error.message'
```

Cloudflare caches aggressively. After a deploy, purge the cache if a change does not appear
(Cloudflare dashboard → Caching → Purge Everything).

## DNS (Cloudflare)

Zone ID `3818b36ba3fcc78f8968ffae9753beda`, account `f85a8fe3ad515231bebea3cb5ca5ffbb`, Free plan.

Nameservers set at Squarespace (DNS → Domain Nameservers → Use Custom Nameservers):

```
dax.ns.cloudflare.com
jacqueline.ns.cloudflare.com
```

Records — the 4 A records and `www` are **proxied** (orange cloud); the TXT records are DNS-only:

| Type  | Name           | Value                                          | Proxy |
|-------|----------------|------------------------------------------------|-------|
| A     | @              | 185.199.108.153                                | on    |
| A     | @              | 185.199.109.153                                | on    |
| A     | @              | 185.199.110.153                                | on    |
| A     | @              | 185.199.111.153                                | on    |
| CNAME | www            | josephpawlowski1.github.io                     | on    |
| TXT   | @              | `v=spf1 -all`                                  | —     |
| TXT   | _dmarc         | `v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s` | —   |
| TXT   | _domainkey     | `v=DKIM1; p=`                                  | —     |
| CNAME | _domainconnect | _domainconnect.domains.squarespace.com         | on    |

No MX records — there is no email on this domain. The three TXT records are the SPF/DMARC/DKIM
policy and must survive any future move.

### SSL settings that matter

- **Encryption mode: `Full`**, pinned explicitly — *not* `Automatic`. GitHub's origin serves a
  `*.github.io` certificate that will never match this domain, so `Full (Strict)` **breaks the
  site**. `Automatic` can upgrade itself to Strict, which is why it is pinned.
- **Always Use HTTPS: on.** `http://` → 301 → `https://`, apex and www.
- Universal SSL covers `katiebethcreations.com` and `*.katiebethcreations.com`.
- DNSSEC is **off** — Squarespace required disabling it to change nameservers.

### Verifying

`dig` against a public resolver; the apex should return **Cloudflare** IPs (104.x / 172.67.x),
not GitHub's 185.199.x:

```bash
dig +short katiebethcreations.com A @1.1.1.1
curl -sI https://katiebethcreations.com/ | head -3
curl -sv https://katiebethcreations.com/ 2>&1 | grep -E 'subject:|issuer:'
```

Expect `CN=katiebethcreations.com`, issuer Let's Encrypt, `server: cloudflare`.

## Alias domains

Four other domains, all registered at Squarespace, **301-redirect** to
`https://katiebethcreations.com` via Squarespace *Domain Forwarding* (Website tab → Domain
Forwarding), each covering root (`@`) and `www`, path forwarding off:

| Domain                   | Renews       |
|--------------------------|--------------|
| creationsbykatiebeth.com | Aug 16, 2027 |
| creationsbykatiebeth.net | Aug 16, 2027 |
| katiebethcreations.net   | Aug 23, 2027 |
| katiebethcreations.store | Aug 23, 2027 |

These still use **Squarespace** nameservers — only katiebethcreations.com moved to Cloudflare.
Their forwarding rules are unaffected by that move.

`katiebethcreations.com` renews Aug 8, 2027 ($20) and is canonical.

Note `creationsbykatiebeth.*` is legacy — it predates the rename and matches no current handle.

**Do not point aliases at GitHub's IPs.** GitHub Pages serves exactly one custom domain per repo
(whatever is in `CNAME`); any other hostname resolving to its IPs gets a 404. If you ever want
them served rather than redirected, move them to Cloudflare and use Redirect Rules, which apply
in seconds instead of Squarespace's 24–48 hours.

## Why Cloudflare (history — read before "simplifying" this)

The site originally ran on GitHub Pages with DNS at Squarespace. **GitHub never issued the TLS
certificate.** It sat at `state: null` — meaning the request was never created, not queued — for
two full days, while DNS, CAA and the ACME challenge path were all verifiably correct. Clearing
and re-adding the custom domain moved it to `authorization_created`, where it stalled again.

Cloudflare issued a working certificate within minutes of the zone activating.

So: **the GitHub Pages certificate is still broken and always was.** It does not matter, because
Cloudflare terminates TLS at the edge and GitHub is only the origin. Do not "fix" this by pointing
DNS straight at GitHub again — that is the broken configuration this setup exists to route around.

### Diagnosing a stuck GitHub Pages certificate

Symptom: HTTPS fails with a certificate *mismatch* (handshake succeeds), origin serves
`CN=*.github.io`:

```bash
gh api /repos/JosephPawlowski1/katiebethcreations/pages \
  --jq '{cname, state: .https_certificate.state}'
```

`state: null` = never requested; waiting will not help. Healthy progression is
`authorization_created → authorization_pending → authorized → issued`.

The clear/re-add cycle (which did **not** ultimately work here):

```bash
gh api -X PUT /repos/JosephPawlowski1/katiebethcreations/pages -f 'cname='
sleep 15
gh api -X PUT /repos/JosephPawlowski1/katiebethcreations/pages -f 'cname=katiebethcreations.com'
```

**This rewrites the repo behind your back** — GitHub pushes `Delete CNAME` and `Create CNAME`
commits, so your next `git push` is rejected as non-fast-forward despite no local changes:

```bash
git fetch origin && git rebase origin/main
```

Also: passing `https_enforced` while no certificate exists fails with a misleading
`404 The certificate does not exist yet`.

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
