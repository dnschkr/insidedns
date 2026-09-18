# CLAUDE.md — insidedns.com

Handover notes. Everything a fresh session needs to work on this without
rediscovering it.

**This repo is PUBLIC.** No secrets in here, ever. Where a credential is
needed, this file says where it lives, not what it is.

---

## What it is right now

A single static page at <https://insidedns.com/> with one working instrument on
it: a probe that reports what your recursive resolver discloses to an
authoritative nameserver. Below that, a guide to what DNS and IP diagnostics
measure and what they cannot tell you, with links to the DNS Checker tools.

Written in Ishan's voice, first person, and it discloses in the byline that the
tools it discusses are his own.

**The direction is unsettled.** See [Open question](#open-question-what-this-should-actually-be)
at the bottom before building anything substantial. Ishan pushed back that the
page still orbits "is my VPN leaking", which is a commodity genre, and wants
something that earns the domain name. That decision has not been made.

---

## Where the code is

| Thing | Location |
|---|---|
| This site | `~/dev/node/2026/dnschkr/insidedns` → `github.com/dnschkr/insidedns` (public) |
| The nameserver behind it | `~/dev/node/2026/dnschkr/dnschkr-dns-leak` → `github.com/dnschkr/dnschkr-dns-leak` (private) |
| DNS Checker itself | `~/dev/node/2026/dnschkr/dnschkr-site` → `dnschkr.com` |

The site is one file. `index.html` contains the markup, all CSS, and the probe
script. No build step, no dependencies, no framework. Keep it that way unless
there is a reason not to.

---

## Hosting

GitHub Pages, serving `master` from the repo root.

- `CNAME` — the custom domain. GitHub rewrites this file when the domain is
  changed through the API, which will make your next push a non-fast-forward.
  Rebase, do not force.
- `.nojekyll` — skip Jekyll.
- HTTPS is enforced, certificate auto-renews.

### The certificate gotcha, because it will happen again

GitHub silently never started provisioning the certificate. `cert: not started`
for 40 minutes with correct DNS and no error anywhere. The fix is to clear the
custom domain and immediately re-add it:

```bash
gh api -X PUT repos/dnschkr/insidedns/pages --input - <<< '{"cname":null}'
sleep 10
gh api -X PUT repos/dnschkr/insidedns/pages --input - <<< '{"cname":"insidedns.com"}'
gh api -X PUT repos/dnschkr/insidedns/pages --input - <<< '{"https_enforced":true}'
```

It went from "not started" to "issued" in about fifteen seconds after that.

---

## DNS

The zone is on **Route 53**, hosted zone `Z0578495FJOUJ3XC9HH1`, AWS account
`571471188499`, profile `dnschkr-cli`. The registrar is GoDaddy and its
nameservers point at Route 53.

```
insidedns.com        A      185.199.108-111.153      GitHub Pages
insidedns.com        AAAA   2606:50c0:8000-8003::153 GitHub Pages
www                  CNAME  dnschkr.github.io
probe                NS     leak-ns.dnschkr.com      the listener below
                            leak-ns2.dnschkr.com
v4                   A      54.235.89.97             IPv4-only on purpose
v6                   AAAA   2600:1f18:3d95:f900:2f4d:70a4:a42b:7410
```

**Do not add an AAAA to `v4`, or an A to `v6`.** The whole point of those two
names is that each resolves on exactly one address family. They back the
dual-stack test on dnschkr.com's what-is-my-ip page.

**Do not touch `probe`.** It is the delegation the resolver probe depends on.

### Why the zone is not on Cloudflare

It was, briefly. Two dead ends, both worth knowing:

1. **Cloudflare accepted an NS record for a subdomain and never served the
   referral.** API returned `success: true`, record visible in the dashboard,
   resolvers got `NOERROR` with the parent SOA. I originally wrote that this
   was a plan restriction. **That was wrong** — Cloudflare
   [documents outgoing delegation on Free](https://developers.cloudflare.com/dns/manage-dns-records/how-to/subdomains-outside-cloudflare/).
   The failure was real, the cause is still unknown. Two things never ruled
   out: I tested within a minute of creating the record, and the nameserver
   was inside the same zone.
2. **GoDaddy's API rejected every custom nameserver hostname** with
   "nameserver hostname is not available", including ones that resolved. No
   public endpoint for the registry host records it seems to want.

What works: registrar → Route 53 → NS delegation → the listener. Route 53
serves subdomain delegations and registrars accept AWS nameservers without
host registration.

---

## The nameserver behind the probe

Repo `dnschkr-dns-leak`, running on the AWS **error-simulator** box.

| | |
|---|---|
| Host | `54.235.89.97`, us-east-1, `i-0553fb9c35e01705b`, Elastic IP so it is stable |
| SSH | `ssh -o IdentitiesOnly=yes -i ~/.ssh/dnschkr-key.pem ubuntu@54.235.89.97` |
| Service | `systemd`, unit `dnschkr-dns-leak`, user `dnsleak` |
| Path | `/opt/dnschkr-dns-leak` |
| Zone | `probe.insidedns.com` (`LEAK_ZONE` in `/opt/dnschkr-dns-leak/.env`) |
| Logs | `/var/log/dnschkr-dns-leak.log` |

`IdentitiesOnly=yes` is not optional. Without it the 1Password agent offers
other keys first and the connection fails on "too many authentication
failures".

The box **also runs the Error Simulator** (pm2 `error-simulator`, port 3002)
and `webcheck`. Do not disturb them. Check `pm2 list` restart counts before and
after any deploy.

### Deploy

```bash
cd ~/dev/node/2026/dnschkr/dnschkr-dns-leak
pnpm build
rsync -avz --rsync-path="sudo rsync" --exclude node_modules --exclude .git --exclude .env \
  ./ ubuntu@54.235.89.97:/opt/dnschkr-dns-leak/ \
  -e "ssh -o IdentitiesOnly=yes -i ~/.ssh/dnschkr-key.pem"
ssh -o IdentitiesOnly=yes -i ~/.ssh/dnschkr-key.pem ubuntu@54.235.89.97 \
  'sudo chown -R dnsleak:dnsleak /opt/dnschkr-dns-leak && sudo systemctl restart dnschkr-dns-leak'
```

`--rsync-path="sudo rsync"` is required: the directory is owned by `dnsleak`
and `ubuntu` cannot write to it. `--exclude .env` is load-bearing; the API key
is only on the box.

Per the workspace rule, run SSH through the `remote-executor` agent in the
background rather than inline.

### Endpoints

| URL | Purpose |
|---|---|
| `https://v4.insidedns.com/ip` | echoes the caller's address, IPv4 only |
| `https://v6.insidedns.com/ip` | same, IPv6 only |
| `https://v4.insidedns.com/resolver/<token>` | what the resolvers disclosed, CORS `*` |
| `http://54.235.89.97:3200/observations/<token>` | older readback, bearer auth, source-IP restricted to the dnschkr origin |

nginx terminates TLS (certbot, auto-renew, expires 2026-12-17) and proxies
`/resolver/` to `127.0.0.1:3200`. Config at
`/etc/nginx/sites-available/dualstack`, backups alongside it. The service sets
its own CORS header — **do not add one in nginx too**, two headers breaks the
browser check.

### Security posture, do not weaken

An authoritative server that answers for names it does not own is a DDoS
reflector. This one was, and it was found in review:

- It echoed the caller's question section back in refusals, and the DNS library
  re-encodes compressed names uncompressed. A reviewer reproduced **33.9x
  amplification** with 180 compressed questions in one packet.
- Now: a packet must be a query, opcode QUERY, exactly one question, no answer
  or authority sections. Anything else gets **no reply at all**. Replies carry
  only the validated question, never the caller's list. UDP replies over 512
  bytes are dropped. [RFC 9619 §4](https://www.rfc-editor.org/rfc/rfc9619#section-4)
  requires `FORMERR` when `QDCOUNT` > 1.
- Out-of-zone names and `ANY` are `REFUSED`.
- Response rate limiting: 25 replies per source address per second.

Regression test it after any change to `src/dns-server.ts`:

```bash
# expect: responses 0, bytes 0
dig +notcp @54.235.89.97 example.com A | grep status          # expect REFUSED
dig +notcp @54.235.89.97 probe.insidedns.com ANY | grep status # expect REFUSED
dig +notcp @54.235.89.97 a.b.c.tok12345678.probe.insidedns.com A +short  # expect 54.235.89.97
```

The amplification harness is in the git history of `dnschkr-dns-leak`.

### What it records, and for how long

In memory only, three minute TTL, swept on a 30 second timer. Capped at 5,000
tokens and 50 resolvers per token, oldest evicted first. Per observation:
resolver address, transport, EDNS Client Subnet if volunteered, DNSSEC DO bit,
EDNS buffer size, query type, and the full query name.

The full query name is what makes QNAME minimisation detectable, and it is more
than the earlier version kept. Nothing is persisted. If you add aggregation,
aggregate counters and never store addresses.

---

## How the probe works

1. The browser generates a random token and fetches five deliberately deep
   names: `<hex>.<hex>.<hex>.<token>.probe.insidedns.com`. The HTTPS request is
   expected to fail; only the DNS lookup matters, and it happens first.
2. Whatever recursive resolver reaches the nameserver is recorded.
3. The page polls `https://v4.insidedns.com/resolver/<token>`.

QNAME minimisation is read off the **shape** of the names. A resolver that
walks down the tree sends short names first; one that does not sends all seven
labels every time. Three verdicts, all verified against a local listener before
deploy:

| Observed | Verdict |
|---|---|
| shortest 4 labels, longest 7 | `yes` |
| three queries, all 7 labels | `no` |
| one query only | `unknown` |

`unknown` is a real answer. One query cannot distinguish the two cases and the
page says so rather than guessing. A probe that never arrives is a blocked
probe, not a clean result.

---

## Credentials

Nothing here is a secret; this is where to find them.

| Need | Where |
|---|---|
| `API_KEY` for the readback | `/opt/dnschkr-dns-leak/.env` on the box, mode 600, owned `dnsleak` |
| AWS | `AWS_PROFILE=dnschkr-cli`, account `571471188499`. Always pass the profile |
| GoDaddy | `GODADDY_PAT` in `~/.env.local` |
| Cloudflare | `CLOUDFLARE_EMAIL` / `CLOUDFLARE_API_KEY` in `dnschkr-site/.env.local` (zone `dnschkr.com` only; this domain is not on Cloudflare) |
| SSH | `~/.ssh/dnschkr-key.pem` |

**GitHub: check the account before every push.** It silently reverted to
`ishanrmn` three times in one session, and that account has no dnschkr access,
so failures present as `403` or `Repository not found` rather than anything
informative.

```bash
gh auth switch --user ishankaru
```

---

## House rules for this page

- No em dashes anywhere, including `&mdash;` and `—`.
- No "we", "our", "us". One person runs this.
- Never state a cause for something you only observed. The Cloudflare mistake
  above is exactly this failure and it shipped publicly.
- Never claim a measurement the code does not make.
- Design: run the slop detector before committing.
  ```bash
  node ~/dev/node/2026/dnschkr/dnschkr-site/.agents/skills/impeccable/scripts/detect.mjs \
    --json ~/dev/node/2026/dnschkr/insidedns/index.html
  ```
  It flagged Instrument Serif as the saturated AI default serif, which is why
  the page uses IBM Plex Serif against the system sans.
- Every JSON-LD `citation` must appear visibly in the body. This has already
  drifted once.

---

## Verify the whole chain

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://insidedns.com/       # 200
dig +short NS probe.insidedns.com                                     # the two leak-ns names
dig +short A v4.insidedns.com                                         # 54.235.89.97, no AAAA
dig +short AAAA v6.insidedns.com                                      # the v6 address, no A
curl -4 -s https://v4.insidedns.com/ip                                # {"ip":"..."}
dig +short a.b.c.tok11111111.probe.insidedns.com A                    # 54.235.89.97
curl -s https://v4.insidedns.com/resolver/tok11111111 | python3 -m json.tool
```

The last two together prove the delegation, the listener and the readback are
all alive.

Browser testing goes through `playwright-cli` with CloakBrowser, headed, per
the workspace rule. Note CloakBrowser suppresses WebRTC at the API level, so it
cannot be used to test anything WebRTC-dependent.

---

## Open question: what this should actually be

Unresolved, and worth settling before more building.

The page began as a guide to DNS Checker's tools. An independent review called
it "a promotional satellite page with useful technical content, not a
demonstrated PBN", and argued the tool survey was the weak half while the
measurement work was the half anyone would actually link to. So the probe
became the lede.

Ishan's objection is that it still orbits "is my VPN leaking", which is a
commodity genre, and the domain deserves better. That is a fair reading.

The unusual assets, for whoever picks this up:

- An **authoritative nameserver**, so resolver behaviour can be observed rather
  than asked about. Very few people running a public page have this.
- Through DNS Checker: zone data for 1,103 TLDs, ~258M domains, ~231M WHOIS
  records, and Project Echo's continuous DNS measurement in ClickHouse.

Directions sketched but not chosen:

1. **A resolver census.** Aggregate anonymously across visitors into a live
   dataset on how the world's recursive resolvers actually behave: what share
   minimise, what share send ECS and at what prefix, DNSSEC, buffer sizes.
   Compounds with traffic. Mostly built already.
2. **Watch a name resolve for real.** Root to TLD to authoritative, actual
   packets and timings. Enormous evergreen search demand. Biggest build; needs
   a recursion path that reports each step.
3. **Oddities from the zone files.** Longest TXT, deepest CNAME chain, most MX
   records, from the real corpus. Shareable, does not compound.

On SEO, so it is not relitigated: a page built to pass equity to a co-owned
domain is a link scheme and gets discounted. The version worth building is one
that ranks and earns links on its own, with a few honest links out. The links
here are natural and descriptive with no exact-match anchors, and that is
deliberate.
