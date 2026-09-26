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
  15-60 min, then enable *Enforce HTTPS* in repo Settings > Pages. If it takes much longer than
  that, see below - it is probably stuck rather than slow.

## Known issue: GitHub Pages certificate stalls at "not requested"

Hit on 2026-09-26 during initial setup. HTTPS stayed dead for hours after DNS was already correct.

### Symptom

`https://katiebethcreations.com` does not load, while `http://` returns 200 normally. The TLS
handshake *succeeds* - the failure is a certificate mismatch, not a connection problem:

```
$ curl -sv https://katiebethcreations.com/ 2>&1 | grep -E 'subject:|subjectAltName'
*  subject: CN=*.github.io
*  subjectAltName does not match host name katiebethcreations.com
```

GitHub is serving its generic wildcard cert because no certificate exists for the custom domain.

### Diagnosis

Check the certificate state - this is the authoritative signal, not the browser:

```bash
gh api /repos/JosephPawlowski1/katiebethcreations/pages \
  --jq '{cname, state: .https_certificate.state, desc: .https_certificate.description}'
```

`state: null` means the certificate was **never requested**. That is different from a certificate
that is queued or pending: waiting will not fix it, because nothing is in flight. A healthy
request progresses `authorization_created` -> `authorization_pending` -> `authorized` -> `issued`.

### Fix

Clear and re-add the custom domain to force GitHub to start issuance over:

```bash
gh api -X PUT /repos/JosephPawlowski1/katiebethcreations/pages -f 'cname='
sleep 15
gh api -X PUT /repos/JosephPawlowski1/katiebethcreations/pages -f 'cname=katiebethcreations.com'
```

This moved `state` from `null` to `authorization_created` immediately.

**Afterwards, verify the `CNAME` file still exists on `main`** - clearing the domain through the
API can delete it, which would break the custom domain entirely:

```bash
gh api /repos/JosephPawlowski1/katiebethcreations/contents/CNAME --jq '.content' | base64 -d
```

The site stays up on HTTP throughout; this does not cause downtime.

### Gotchas within the fix

- Passing `https_enforced` while no certificate exists fails with a misleading
  `404 The certificate does not exist yet`. Set `cname` alone first, then enable
  *Enforce HTTPS* only once `state` is `issued`.
- Do not enable *Enforce HTTPS* before issuance completes - it makes the site unreachable rather
  than merely insecure.

### The fix rewrites the repo behind your back

Clearing and re-adding `cname` makes GitHub push two commits of its own - `Delete CNAME` and
`Create CNAME`. Your next `git push` will then be **rejected** as non-fast-forward, even though
you changed nothing locally. Fetch and rebase onto them; the net CNAME content is unchanged
(GitHub's version has no trailing newline, which is fine):

```bash
git fetch origin && git rebase origin/main
```

### If it stalls again after the fix

Suspect something interfering with GitHub's domain validation. The leftover
`_domainconnect` CNAME from Squarespace is the first thing to rule out.

## Alternative: Cloudflare (considered, not adopted)

Evaluated on 2026-09-26 and **declined** — the current setup was already working by then, and
switching would have meant hours of downtime to land in the same place. Documented here because
the tradeoffs still apply if the Squarespace side becomes painful.

Two separable decisions, often confused:

1. **Who answers DNS** — Squarespace (today) vs Cloudflare
2. **Who serves the files** — GitHub Pages (today) vs Cloudflare Pages

They can be mixed. **Cloudflare DNS + GitHub Pages hosting** is the cheapest useful change: it
removes the Squarespace DNS friction without touching the repo or deploy flow.

### What moving to Cloudflare DNS would fix

- **The repeated Google re-auth prompts.** Squarespace re-verifies roughly once per domain while
  editing records. This cost several interruptions during setup.
- **Squarespace re-adding its preset records.** The "Squarespace Defaults" preset comes back when
  you reconnect a site and silently breaks external hosting (see the DNS section above).
- **The 24-48 hour wait on alias redirects.** Cloudflare *Redirect Rules* apply in seconds and
  are free, replacing the four Squarespace forwarding rules.
- **Multiple custom domains.** Cloudflare Pages allows many per project; GitHub Pages allows
  exactly one, which is why the aliases need forwarding today.

### What it would cost

- Free. Cloudflare's DNS and Pages free tiers both cover this site.
- Squarespace stays the **registrar** — renewals and ownership are unaffected. Only the
  nameservers change.

### Why it was not done

- Nameserver changes propagate over hours, not minutes; the site can be intermittently
  unreachable during the switch.
- Requires creating a Cloudflare account.
- **All records must be recreated**, including the email TXT records (SPF/DMARC/DKIM). Cloudflare's
  onboarding scan imports most automatically, but a missed record breaks email delivery silently.
  Verify each one against the rollback list before flipping nameservers.
- The benefit is mostly one-time friction that has already been paid.

### When to revisit

If Squarespace clobbers the DNS records again, if the domains move off Squarespace entirely, or
if the site outgrows a single static page and needs redirect rules, custom headers, or staging.

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
