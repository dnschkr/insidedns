# insidedns.com

A one-page field guide to what DNS and IP diagnostics actually measure, and
what each one cannot tell you.

Written by Ishan Karunaratne, who also builds [DNS Checker](https://dnschkr.com/).
The page says so plainly: reviewing your own tools is worth less than an
independent review, and pretending otherwise would be the only dishonest thing
on it.

## Why this domain exists

`probe.insidedns.com` is the zone the DNS Checker leak test delegates to. A
browser cannot see which recursive resolver serves it; only the authoritative
server for a name can. So the test needs a zone, and this is it. The public
page is a second use for a domain that was already doing work.

## Hosting

Static single file, served by GitHub Pages from `master`. No build step.

- `index.html` — the whole site
- `CNAME` — custom domain
- `.nojekyll` — skip Jekyll processing

DNS lives in Route 53 (hosted zone `Z0578495FJOUJ3XC9HH1`). The apex points at
GitHub Pages; `probe`, `v4` and `v6` point at measurement infrastructure and
must not be touched when changing the site.
