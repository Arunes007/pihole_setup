# Pi-hole + Unbound + WireGuard on an Ubuntu VPS
## Complete build log, rationale, gotchas, and rebuild manual

**Server:** `51.79.145.116` (hostname `vps-7d3414ff`)
**OS:** Ubuntu 26.04.1 LTS (Resolute Raccoon)
**Built on:** 2026-10-03
**Login:** `ssh ubuntu@51.79.145.116`
**This document lives at:** `/home/ubuntu/workspace/pihole_setup/README.md`
(a `clients/` subfolder next to it holds the ready-to-scan WireGuard profiles)

**Goal of this document:** if this VPS ever dies, you should be able to rebuild
the *entire* setup — Pi-hole, Unbound, WireGuard, NAT, lockdown — using **only**
this file. Every step explains *what*, *why*, and *what went wrong the first
time* (the gotchas are the most valuable part; read them before re-running
anything).

**Companion files in this folder:**

| File | Contents | Permissions |
|---|---|---|
| `README.md` (this file) | The whole manual | `644` |
| `clients/phone.conf` | Full-tunnel WireGuard profile for the phone (all traffic via VPS) | `600` |
| `clients/phone-dnsonly.conf` | DNS-only WireGuard profile (only DNS via VPS, web traffic direct) | `600` |

> **Security note:** the `clients/` files contain WireGuard **private keys**.
> Anyone holding them can join your VPN. `600` permissions + `700` on the
> folder is the minimum. Do not publish this folder anywhere. If a key ever
> leaks, generate a fresh keypair and replace the peer (procedure in §9.5).

---

## Table of contents

1. [What lives on this server (read first — do not disturb)](#1-what-lives-on-this-server-read-first--do-not-disturb)
2. [Architecture: what we built and how traffic flows](#2-architecture-what-we-built-and-how-traffic-flows)
3. [What this setup does — and the YouTube asterisks](#3-what-this-setup-does--and-the-youtube-asterisk)
4. [Glossary (concepts you need before touching anything)](#4-glossary-concepts-you-need-before-touching-anything)
5. [Prerequisites and port map](#5-prerequisites-and-port-map)
6. [Step 1 — SSH in and update the system](#6-step-1--ssh-in-and-update-the-system)
7. [Step 2 — Install Pi-hole (incl. the stuck-installer gotcha)](#7-step-2--install-pi-hole-incl-the-stuck-installer-gotcha)
8. [Step 3 — Answer the Pi-hole installer prompts correctly](#8-step-3--answer-the-pi-hole-installer-prompts-correctly)
9. [Step 4 — Install and configure Unbound (private DNS)](#9-step-4--install-and-configure-unbound-private-dns)
10. [Step 5 — Pi-hole v6 specifics: web port, listening mode, the Docker port conflict](#10-step-5--pi-hole-v6-specifics-web-port-listening-mode-the-docker-port-conflict)
11. [Step 6 — WireGuard: server discovery, peers, client profiles](#11-step-6--wireguard-server-discovery-peers-client-profiles)
12. [Step 7 — The "VPN connects but no internet" NAT fix](#12-step-7--the-vpn-connects-but-no-internet-nat-fix)
13. [Step 8 — Lockdown: closing the public DNS resolver](#13-step-8--lockdown-closing-the-public-dns-resolver)
14. [Verification checklist (run after every rebuild or reboot)](#14-verification-checklist-run-after-every-rebuild-or-reboot)
15. [Daily operation handbook](#15-daily-operation-handbook)
16. [Rebuild-from-scratch: the short ordered version](#16-rebuild-from-scratch-the-short-ordered-version)
17. [Troubleshooting matrix](#17-troubleshooting-matrix)
18. [Credentials and file inventory](#18-credentials-and-file-inventory)

---

## 1. What lives on this server (read first — do not disturb)

When we arrived, this VPS was **already running production services in
Docker**. Everything we built had to fit *around* them. This is the single
most important context for the whole build — half the gotchas below come
directly from this table:

| Container | Ports it owns | What it is |
|---|---|---|
| `caddy` | `80/tcp`, `443/tcp`, `443/udp`, `2019/tcp` (bound to all interfaces, `0.0.0.0`) | Reverse proxy / HTTPS front door |
| `navidrome` | `4533/tcp` (internal) | Music server (reached through Caddy) |
| `songbot-botapi-1` | `127.0.0.1:8081->8081/tcp` | A bot API, localhost-only |

How we know (run any time):

```bash
docker ps --format "{{.Names}}: {{.Ports}}"
sudo ss -tlnp | grep -E ':80|:443|:53|:8081|:5335|:8888'
```

**Rules that follow from this:**

1. **Never bind anything new to ports 80 or 443.** They belong to the
   `caddy` container (via `docker-proxy`). Pi-hole's web UI defaults to port
   80 — that had to move (§10).
2. **Never put firewall rules in Docker's chains** (`DOCKER-USER` excepted —
   that chain exists *specifically* for admin custom rules; §12 explains).
3. **Port 8081 is taken** (songbot, localhost-only). Our Pi-hole web UI went
   to **8888** instead.
4. **UFW is inactive** (`sudo ufw status` → `Status: inactive`). Packet
   filtering is done with raw `iptables`, and Docker manages its own chains.
   Do not enable UFW casually — its default-deny posture can break Docker
   networking. All our rules are plain `iptables` + WireGuard `PostUp`.

---

## 2. Architecture: what we built and how traffic flows

Three cooperating pieces, each doing exactly one job:

```
                        ┌──────────────────────────────────────────────┐
                        │              VPS 51.79.145.116               │
                        │                                              │
   PHONE (anywhere)     │   WireGuard 10.77.0.1/24  (UDP 51820)        │
        │               │        │                                     │
        │  encrypted    │        ▼                                     │
        └────tunnel────►│   Pi-hole  (DNS, port 53)                    │
                        │        │  blocks ads/trackers/telemetry      │
                        │        │  upstream: 127.0.0.1#5335            │
                        │        ▼                                     │
                        │   Unbound  (127.0.0.1:5335)                  │
                        │        │  recursive resolver, talks to root   │
                        │        │  servers directly — no Google/CF    │
                        │        ▼                                     │
                        │   Internet (via ens3, NAT-masqueraded)       │
                        └──────────────────────────────────────────────┘
```

- **Pi-hole** = the *policy* layer. It receives DNS questions ("what IP is
  `ads.example.com`?") and answers `0.0.0.0` for blocked domains, otherwise
  forwards to Unbound. It also logs every query to its dashboard.
- **Unbound** = the *resolution* layer. A recursive resolver: starting from
  the DNS root servers, it walks the DNS tree itself. Nobody upstream
  (Google, Cloudflare, your ISP's resolver) ever sees the query.
- **WireGuard** = the *transport* layer. An encrypted tunnel that carries
  your phone/laptop traffic to the VPS so Pi-hole can filter it *even on
  mobile data*, and so your ISP sees only ciphertext.

Two phone profiles exist (§11.6) — same keys, different routing:

```
FULL TUNNEL (clients/phone.conf)
  AllowedIPs = 0.0.0.0/0, ::/0
  → EVERYTHING (web + DNS) exits at the VPS. Websites see 51.79.145.116.
  → ISP sees nothing. Some sites (banks, Netflix) may block datacenter IPs.

DNS-ONLY (clients/phone-dnsonly.conf)
  AllowedIPs = 10.77.0.0/24
  → ONLY traffic for 10.77.0.x (i.e. DNS to 10.77.0.1) uses the tunnel.
  → Web traffic goes out your normal connection. Websites see your home IP.
  → You keep DNS-level ad blocking. ISP sees destination IPs but not DNS.
```

---

## 3. What this setup does — and the YouTube asterisk

### It DOES

- Block ads, trackers, and telemetry **network-wide** for every device that
  uses it (phones, laptops, smart TVs, IoT) with zero per-device software.
- Act as a **DNS sinkhole**: when e.g. a smart TV asks for
  `telemetry.manufacturer.com`, Pi-hole answers `0.0.0.0`, so the TV's
  "phone-home" connection dies before it starts. Same mechanism kills
  malware command-and-control lookups.
- Show a **live query dashboard** (who asked for what, what got blocked).
- Give you **private DNS** (Unbound — no third-party resolver sees queries)
  and, over WireGuard, **ISP-proof transport** (ISP sees only an encrypted
  tunnel to your VPS).
- Offer parental-control-style blocking via blocklists, per-client rules,
  custom allow/denylist entries.

### It does NOT (read before being disappointed)

- **It cannot block ads inside the YouTube / Facebook / Instagram *apps***
  (and only partially on their websites). Reason: those apps serve ads from
  the *same domains* as the content (`googlevideo.com` for YouTube). DNS
  blocking cannot distinguish "ad bytes" from "video bytes" coming from one
  domain — blocking the domain kills the video too. This is a fundamental
  DNS-layer limit, not a misconfiguration. No blocklist fixes it.
- Workarounds that *do* work for YouTube:
  - **uBlock Origin** browser extension (best for desktop/laptop browsers),
  - **SmartTube** (Android TV / Firestick),
  - **ReVanced** (patched YouTube app, Android phones),
  - **YouTube Premium** (paid, official).
- It is not a full privacy suite by itself: without WireGuard active, your
  ISP still sees destination IPs and unencrypted traffic. WireGuard is the
  part that hides things from the ISP.

---

## 4. Glossary (concepts you need before touching anything)

- **DNS (Domain Name System).** The internet's phone book: translates names
  (`google.com`) to IPs. Almost everything you do online starts with a DNS
  question. Whoever answers those questions sees *every* site you visit —
  which is why controlling your DNS matters.
- **DNS sinkhole.** A DNS server that answers queries for *bad* domains with
  a fake dead address (`0.0.0.0`) instead of the real IP. The client tries to
  connect to nothing and gives up. Pi-hole *is* a sinkhole with a nice UI.
- **Upstream DNS.** Who your DNS server asks when it doesn't know the answer
  itself. Default Pi-hole setups use Google (`8.8.8.8`) or Cloudflare
  (`1.1.1.1`) — meaning those companies see all your queries. We replaced
  that with Unbound on `127.0.0.1#5335`.
- **Recursive resolver (Unbound).** Instead of asking Google, Unbound asks
  the DNS root servers → TLD servers → authoritative servers *itself*,
  following referrals down the tree. Slower on first lookup (~300 ms,
  cached afterwards), but no third party is involved at all.
- **`root.hints`.** The file containing the IP addresses of the 13 DNS root
  servers — the starting point of every recursive lookup. Unbound ships
  expecting it at `/var/lib/unbound/root.hints`; on our box it was **missing**
  and Unbound refused to start (§9.3 gotcha).
- **WireGuard.** A modern VPN protocol (~4,000 lines of code, in the Linux
  kernel). Creates an encrypted tunnel with virtual IPs (`10.77.0.x` here).
  Anything sent through the tunnel is opaque to the local network/ISP.
- **AllowedIPs.** The single most misunderstood WireGuard setting. It is a
  *routing* decision on the client: "which destination IPs go through the
  tunnel". `0.0.0.0/0` = everything (full tunnel). `10.77.0.0/24` = only the
  VPN subnet itself (DNS-only split tunnel). On the server side it doubles as
  an access list (which source IPs a peer may send from).
- **NAT / MASQUERADE.** Our VPN clients have private IPs (`10.77.0.x`) that
  are meaningless on the public internet. When their packets leave the VPS,
  the VPS must rewrite the source address to its own public IP
  (`51.79.145.116`) and remember the mapping so replies find their way back.
  That rewrite is NAT; the iptables target is called MASQUERADE. **Missing
  MASQUERADE was our Day-1 outage** (§12).
- **IP forwarding.** The kernel switch (`net.ipv4.ip_forward`) that allows
  the VPS to pass packets *between* interfaces (`wg0` ↔ `ens3`). A VPS is a
  router for its VPN clients; without this, packets arrive and die. Ours was
  already `1`.
- **listeningMode (Pi-hole v6).** Who Pi-hole answers DNS for:
  `LOCAL` = only clients on local subnets (safe default);
  `ALL` = the whole internet (needed temporarily during setup, dangerous to
  keep — see *open resolver*). There is no per-interface picker in v6; the
  old `pihole -a -i all` CLI from v5 guides **does not exist anymore**.
- **Open resolver.** A DNS server that answers queries from the entire
  internet. Attackers abuse these for **DNS amplification DDoS** (tiny forged
  query in, huge response out to a victim). Scanners find new open resolvers
  within *hours*. Hosts (OVH/Hetzner-style) will suspend you for it. Our box
  was briefly an open resolver during setup and was locked down in §13.
- **Split tunnel.** Routing only *some* traffic through the VPN (here: only
  DNS). Trade-off between privacy (full tunnel) and compatibility (sites that
  block datacenter IPs see your residential IP instead).
- **qname minimisation.** Unbound privacy feature: when walking down the DNS
  tree, it reveals only the minimum necessary part of the name at each level
  (e.g. asks root only about `.com`, not the full hostname). Enabled in our
  config.

---

## 5. Prerequisites and port map

- A VPS running Ubuntu (ours: 26.04.1 LTS), with `sudo` access as `ubuntu`.
- SSH key auth working (`ssh ubuntu@51.79.145.116` with no password).
- Outbound internet from the VPS (for `apt`, blocklists, root servers).
- No service already using the ports below (we checked with `ss`).

| Port | Proto | Service | Exposed to | Notes |
|---|---|---|---|---|
| 22 | TCP | SSH | Internet (you) | How you administer the box |
| 53 | UDP/TCP | Pi-hole DNS | **VPN + localhost only** (locked down, §13) | Was briefly public during setup — never again |
| 51820 | UDP | WireGuard | Internet (your devices) | UDP only; TCP probes to it mean nothing |
| 5335 | UDP/TCP | Unbound | localhost only (`127.0.0.1`) | Pi-hole's upstream; never public |
| 8888 | TCP | Pi-hole web UI | Internet (password-protected) | Moved off 80/443 — Docker owns those |
| 80/443 | TCP | Caddy (Docker) | Internet | **NOT OURS. Do not touch.** |
| 8081 | TCP | songbot (Docker, localhost) | localhost | Reason 8888 was chosen over 8081 |
| 4533 | TCP | Navidrome (Docker) | via Caddy | **Do not touch.** |

---

## 6. Step 1 — SSH in and update the system

**Rationale:** every guide assumes this, and an outdated package cache is the
#1 cause of mysterious `apt` failures mid-install. Do it first, always.

```bash
ssh ubuntu@51.79.145.116
sudo apt update && sudo apt upgrade -y
```

No gotchas here. (`sudo` will ask for *your* password only if key+sudo
require it; on this box the `ubuntu` user has passwordless sudo.)

---

## 7. Step 2 — Install Pi-hole (incl. the stuck-installer gotcha)

**Rationale:** Pi-hole is the policy layer (§2). Install it *before*
Unbound, but point it at Unbound as upstream during the installer prompts
(§8) — order matters because the installer asks for the upstream DNS server
exactly once.

### 7.1 Method A (official one-liner)

```bash
curl -sSL https://install.pi-hole.net | bash
```

> The pipe-to-bash pattern is safe *here* because the URL is Pi-hole's
> official installer domain, served over HTTPS. As a rule, never pipe random
> URLs to bash.
>
> You do **NOT** need `sudo` in front: the script detects non-root, prints
> "Script called with non-root privileges … requires elevated privileges",
> and re-elevates itself with `sudo` where needed.

### 7.2 GOTCHA — the installer hangs installing `pihole-meta.deb`

**What happened to us:** the installer printed

```
[✓] Building dependency package pihole-meta.deb
[i] Installing Pi-hole dependency package...
```

…and sat there for 13+ minutes. Diagnosis:

```bash
ps aux | grep pihole
# → apt-get -qq --no-install-recommends install /tmp/pihole-meta.deb  (stuck)

ls -lh /tmp/pihole-meta.deb
# → 658 bytes. A real .deb is megabytes. 658 bytes = a FAILED download
#    (an error page saved as a .deb). apt then choked on it forever.
```

**Fix:**

1. `Ctrl+C` in the installer terminal. If that doesn't respond, open a
   **second** SSH session and:
   ```bash
   sudo kill <PID>        # PID from the ps output, e.g. 1011260
   sudo kill -9 <PID>     # if plain kill doesn't work
   ```
   (Closing the SSH session entirely and reconnecting also works.)
2. Delete the corrupt file: `sudo rm -f /tmp/pihole-meta.deb`
3. Re-run the installer. It usually succeeds on retry (transient mirror
   hiccup). If Method A keeps failing, use Method B.

### 7.3 Method B (git clone fallback — what actually worked for us)

```bash
git clone --depth 1 https://github.com/pi-hole/pi-hole.git Pi-hole
bash "Pi-hole/automated install/basic-install.sh"
```

**GOTCHA — the space in the path.** The repo directory is literally named
`automated install` (with a space), *not* `automated-install`. The naïve
command fails:

```bash
bash Pi-hole/automated-install/basic-install.sh
# → bash: Pi-hole/automated-install/basic-install.sh: No such file or directory
```

Fix: quote the path (`"Pi-hole/automated install/basic-install.sh"`) or
Tab-complete it. Verify with `ls Pi-hole/` first — you'll see
`'automated install'` listed.

---

## 8. Step 3 — Answer the Pi-hole installer prompts correctly

The installer is interactive (dialog menus). Each answer and *why*:

### 8.1 Network interface

It asks which interface Pi-hole should bind to. Identify yours first:

```bash
ip addr
```

Pick the interface holding your **public IP**. Ours:

```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 51.79.145.116/32 ... scope global dynamic ens3
```

→ Select **`ens3`**. (Ignore `lo`, `docker0`, `br-*`, `veth*`, and `wg0`.)
**Rationale:** Pi-hole must listen where client queries arrive. `lo` is
localhost-only; docker bridges are container networks. `ens3` is the VPS's
front door. (VPN clients arrive on `wg0`, which Pi-hole v6 handles via
`listeningMode`, §10.3 — not via this prompt.)

> Observation at the time: a `wg0` interface (`10.77.0.1/24`) already
> existed — WireGuard had been set up on this box earlier (§11).

### 8.2 Upstream DNS provider → Custom: `127.0.0.1#5335`

**Rationale:** this is the single setting that wires Pi-hole → Unbound. The
`#5335` is a port suffix (not a comment): "ask the resolver at 127.0.0.1 on
port 5335". Unbound doesn't exist yet at this point — that's fine, we
install it next (§9) and Pi-hole starts working once Unbound answers. If you
picked Google/Cloudflare here instead, you'd hand your entire query history
to them; the whole point of Unbound is avoiding exactly that.

### 8.3 Query logging → Yes

**Rationale:** enables the dashboard's live query log — your main debugging
tool ("is device X actually using Pi-hole?", "what got blocked?"). Costs
negligible disk on a VPS. Disable later in the UI if you ever care.

### 8.4 Success output — SAVE THIS

```
[i] Pi-hole DNS (IPv4): 51.79.145.116
[i] View the web interface at http://pi.hole:80/admin or http://51.79.145.116:80/admin
[i] Web Interface password: V8WF-2pK
[i] The install log is located at: /etc/pihole/install.log
[✓] Installation complete!
```

- `http://pi.hole/admin` only resolves *for clients using Pi-hole as DNS*
  (it's a fake local domain Pi-hole answers itself). From outside, use the
  IP URL — which later moved to port 8888 (§10.2).
- The password can be changed any time: `sudo pihole setpassword`.
- Full installer log: `/etc/pihole/install.log` (invaluable when something
  fails months later and you've forgotten what you clicked).

---

## 9. Step 4 — Install and configure Unbound (private DNS)

**Rationale:** Pi-hole blocks; Unbound resolves. Without Unbound, Pi-hole
forwards every allowed query to Google/Cloudflare, who then see your full
browsing profile. With Unbound, queries go to root/TLD/authoritative
servers directly — no middleman. Cost: ~10–30 ms extra on *uncached*
lookups (cached answers are instant); negligible on a VPS.

```bash
sudo apt install unbound -y
```

Config file (create it):

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Canonical minimal config (this exact text is live on the server — verified
working). Every line is annotated:

```ini
server:
    interface: 127.0.0.1        # listen on localhost ONLY — never public
    port: 5335                   # must match Pi-hole's upstream 127.0.0.1#5335
    access-control: 127.0.0.0/8 allow   # only localhost may query
    do-daemonize: yes
    username: "unbound"          # drop root after binding (least privilege)
    verbosity: 1
    use-syslog: yes
    root-hints: "/var/lib/unbound/root.hints"  # seed list of root servers
    harden-glue: yes             # ignore out-of-zone "glue" (spoof hardening)
    harden-dnssec-stripped: yes  # treat stripped DNSSEC as bogus
    do-not-query-localhost: no   # MUST be 'no': our upstream chain ends here,
                                 # and Pi-hole lives on localhost
    prefetch: yes                # refresh popular entries before they expire
    qname-minimisation: yes      # leak minimal query info up the tree (§4)
    auto-trust-anchor-file: "/var/lib/unbound/root.key"  # DNSSEC trust anchor
```

**Rationale for what is NOT in this file:** widely-copied "hardened"
configs append hundreds of `local-zone: "xxx.in-addr.arpa." nodefault` lines
(one per private-IPv4 reverse zone). Those only suppress pointless reverse
lookups of private IPs leaking to root servers — a micro-optimisation. We
deliberately run **without** them: fewer lines, fewer typos, identical
blocking behavior. (What those lines mean, if you ever meet them: "answer
NXDOMAIN immediately for reverse lookups of RFC-1918 addresses instead of
bothering upstream".)

### 9.1 GOTCHA — `root.hints: No such file or directory`

**What happened:** `sudo systemctl restart unbound` failed:

```
Job for unbound.service failed because the control process exited with error code.
```

`journalctl -xeu unbound.service` showed crash-looping ("Start request
repeated too quickly"), and the real cause came from:

```bash
sudo unbound-checkconf
# /var/lib/unbound/root.hints: No such file or directory
# fatal error: file with root-hints ... does not exist
```

**Rationale:** the Ubuntu `unbound` package does not always ship the root
hints file, but our config *references* it, so Unbound aborts at parse
time. Fix — download the canonical copy from IANA/InterNIC:

```bash
sudo wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
sudo systemctl restart unbound
```

**Lesson (general):** when a systemd service crash-loops, `systemctl status`
tells you *that* it died; the service's own config checker (`unbound-checkconf`,
`nginx -t`, …) tells you *why*. Always run the checker.

### 9.2 GOTCHA — the giant config with typos (historical, do not reuse)

An earlier draft of this config (a ~700-line AI-generated variant) contained
**four classes of typos**, each fatal or silently wrong:

1. `local: "172.22.in-addr.arpa." nodefault` — key must be `local-zone:`
   (missing `-zone` → parse error, Unbound won't start).
2. `"172.157.in-addr.arpa." noddefault` — value must be `nodefault`
   (missing `e` → parse error).
3. `"72.46.in-addr.arpa."` / `"72.199.in-addr.arpa."` — dropped the leading
   `1` (`72.` instead of `172.` → wrong zone, silently useless).
4. The whole block pasted **twice** (duplicates are tolerated, but it shows
   the text was never validated).

Any *one* parse error kills Unbound at startup. Moral: after every edit, run
`sudo unbound-checkconf` **before** restarting, and prefer the short config
above — surface area for typos is everything.

### 9.3 Verify Unbound

```bash
dig @127.0.0.1 -p 5335 google.com
```

Expected (first query ≈ 300–400 ms while it walks the tree; repeats are
instant from cache):

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: ...
;; ANSWER SECTION:
google.com.     300  IN  A  142.251.12.138
...
;; Query time: 369 msec
;; SERVER: 127.0.0.1#5335(127.0.0.1) (UDP)
```

`status: NOERROR` + real `A` records = Unbound resolves. If it times out,
re-check §9.1 and `systemctl status unbound`.

---

## 10. Step 5 — Pi-hole v6 specifics: web port, listening mode, the Docker port conflict

Pi-hole v6 stores **everything** in one TOML file:

```
/etc/pihole/pihole.toml       # (root-owned — read with sudo)
```

Key settings and their live values on this box:

| TOML key | Value | Meaning |
|---|---|---|
| `dns.upstreams` | `[ "127.0.0.1#5335" ]` | Pi-hole → Unbound (§8.2) |
| `dns.interface` | `"ens3"` | Install-time choice (§8.1) |
| `dns.listeningMode` | `"LOCAL"` | Answer local subnets only (§10.3, §13) |
| `dns.port` | `53` | Standard DNS port |
| `webserver.port` | `"80o,443os,[::]:80o,[::]:443os,8888o"` | Web UI ports (§10.2) |
| `webserver.acl` | `""` | Empty = web UI reachable from anywhere (password-gated) |

Change values with (no manual TOML editing needed):

```bash
sudo pihole-FTL --config <dotted.key> "<value>"
sudo systemctl restart pihole-FTL
```

### 10.1 GOTCHA — `pihole -a -i all` doesn't exist on v6

Countless tutorials say `pihole -a -i all` to make Pi-hole listen
everywhere. On v6 it prints the help text and does nothing — that flag was
removed. Interface behavior is now `dns.listeningMode` in `pihole.toml` (or
the web UI: **Settings → DNS → Interface settings**). Stop following v5
guides for interface settings.

### 10.2 The web UI vs Docker-Caddy port fight (why port 8888)

**Symptoms we saw:** `http://51.79.145.116/admin` in a browser →
`ERR_SSL_PROTOCOL_ERROR`. From the server, `curl -I` revealed:

```
HTTP/1.1 308 Permanent Redirect
Location: https://51.79.145.116/admin
Server: Caddy
```

…and `https://…` then died with a TLS `internal error` (Caddy has
certificates for its *domains*, not for a bare IP, so the handshake aborts).

**Root cause:** `sudo ss -tlnp` showed `docker-proxy` (the `caddy`
container, §1) owning ports 80 and 443 on all interfaces. Pi-hole's default
`webserver.port` starts with `80o,443os,…` — and the trailing **`o` means
*optional***: if FTL can't bind a port, it silently skips it instead of
crashing. So Pi-hole lost 80/443 without a single error, while Caddy
answered (and redirected) all port-80 traffic first — including `/admin`.

Port suffix letters in `webserver.port` (from the commented `pihole.toml`):

| Suffix | Meaning |
|---|---|
| (none) | mandatory — FTL fails to start if unavailable |
| `o` | optional — skip silently if unavailable |
| `s` | TLS/SSL on this port |
| `r` | redirect to the first SSL port |

**Fix (leaves Caddy/Navidrome 100% untouched):** append a free plain-HTTP
port for Pi-hole. We verified `8888` and `8090` were free
(`sudo ss -tlnp | grep -E ':8888|:8090'` → empty) and chose **8888**
— *not* 8081, because songbot already owns `127.0.0.1:8081`:

```bash
sudo pihole-FTL --config webserver.port "80o,443os,[::]:80o,[::]:443os,8888o"
sudo systemctl restart pihole-FTL
```

Admin UI: **`http://51.79.145.116:8888/admin`** (plain HTTP; the `:8888`
bypasses Caddy entirely). A `30x` status from curl is normal (redirect to
`/admin/` login page = the server is alive).

### 10.3 `listeningMode`: LOCAL vs ALL (and the open-resolver trap)

- `LOCAL` (default, and our final state): answer only queries whose source
  is on a local subnet — localhost, `ens3`'s network, and the WireGuard
  subnet `10.77.0.0/24`. **Internet strangers are ignored.** Safe.
- `ALL`: answer anyone on the internet. We used this *temporarily* during
  setup to prove DNS worked externally, then reverted (§13). Keeping `ALL`
  with a public port 53 = **open resolver** = amplification-DDoS abuse =
  possible VPS suspension (§4).

How we proved the modes from the outside (from a laptop, not the VPS):

```bash
dig +short +time=4 +tries=1 @51.79.145.116 google.com
# ALL   → returns IPs (working, but dangerous to keep)
# LOCAL → "connection timed out; no servers could be reached" (desired!)
```

Timeout here is *success* — it means Pi-hole drops strangers.

### 10.4 GOTCHA — `systemctl restart pihole-FTL` hangs your SSH session

Twice during this build, `sudo systemctl restart pihole-FTL` over SSH
**never returned** — the terminal froze until we aborted it. The restart
itself *always completed fine in the background* (new FTL PID, new config
live). It's the SSH channel that wedges, not the service.

**Rule from now on — never plain-restart FTL over SSH. Use:**

```bash
sudo systemctl --no-block restart pihole-FTL   # returns immediately
sleep 8
systemctl is-active pihole-FTL                 # separate check: expect 'active'
```

`--no-block` tells systemd "start the job, don't wait for it". Then poll.
This pattern ended all the "is it stuck?" episodes.

---

## 11. Step 6 — WireGuard: server discovery, peers, client profiles

**Rationale refresher:** without a VPN, your phone on mobile data sends DNS
to your carrier, not Pi-hole — and your ISP sees everything. WireGuard puts
a private encrypted road between your devices and the VPS so Pi-hole
filters you *everywhere*.

### 11.1 Discovery: the server already existed

`ip addr` (§8.1) showed a live `wg0` (`10.77.0.1/24`). Inspection:

```bash
sudo wg show
sudo cat /etc/wireguard/wg0.conf
```

```
interface: wg0
  public key: uGM/3c6XTSNyU/ItYV8jBWPMiI1glOadBHOiuS5zHyU=
  listening port: 51820
peer: X5SRQeES3oiuIjEy0wyZ2AJz1vJrWXGLKYg6w/paJRY=   # "Mac client"
  endpoint: 122.172.81.221:29303
  allowed ips: 10.77.0.2/32
  latest handshake: 20 days ... ago                  # stale = offline for weeks
  transfer: 16.75 MiB received, 69.35 MiB sent       # ...but it worked back then
```

How to read this (you'll do it for every future debug):

- `latest handshake` = last time this peer completed WireGuard's handshake.
  Recent = reachable + keys correct. Stale/missing = peer offline, wrong
  endpoint, or wrong keys. (UDP has no "connection refused" — silence is the
  only error message, so handshake age is your connectivity signal.)
- `transfer: RX TX` = bytes received from / sent to the peer. A peer with
  growing RX but flat TX is *reaching* the server but getting nothing back —
  the classic NAT symptom (§12). Our phone showed `796420 received, 9260
  sent` while broken, then megabytes both ways once fixed.

> Note: `sudo cat /etc/wireguard/wg0.conf` once printed the server's
> **private key** into chat. Treat that key as exposed: if this history ever
> leaves your machine, rotate the server keypair (`wg genkey` × 2, update
> `wg0.conf` + all peers). Clients keep working only if you *don't* rotate;
> weigh accordingly. A backup of the pre-NAT config lives at
> `/etc/wireguard/wg0.conf.bak`.

### 11.2 WireGuard and reboots

```bash
systemctl is-enabled wg-quick@wg0   # → enabled (starts at boot) ✓
```

`SaveConfig = false` in `wg0.conf` means: live `wg set` changes are
**forgotten** on restart — the file is the truth. So every peer must exist
**both** live (for now) **and** in the file (for after reboot). The
procedure below does both, in that order.

### 11.3 Add a peer (phone, `10.77.0.3`) — the exact procedure

```bash
# 1. Generate the phone's keypair (private stays secret to the phone afterwards)
wg genkey | tee /tmp/phone.key | wg pubkey | tee /tmp/phone.pub
chmod 600 /tmp/phone.key

# 2. Teach the LIVE server about the new peer (no restart, no disruption)
PUB=$(cat /tmp/phone.pub)
sudo wg set wg0 peer "$PUB" allowed-ips 10.77.0.3/32

# 3. Persist it in wg0.conf (or the next reboot forgets the phone)
printf '\n[Peer]\n# Phone\nPublicKey = %s\nAllowedIPs = 10.77.0.3/32\nPersistentKeepalive = 25\n' \
  "$PUB" | sudo tee -a /etc/wireguard/wg0.conf

# 4. Validate the file parses (catches quoting/typo damage immediately)
sudo wg-quick strip /etc/wireguard/wg0.conf >/dev/null && echo CONF_OK

# 5. Confirm the peer is live
sudo wg show wg0 peers
```

Field rationale:

- `AllowedIPs = 10.77.0.3/32` (**server side**): this peer may only send
  from `.3`; each peer gets its own `/32` so they can't spoof each other.
- `PersistentKeepalive = 25`: phone is usually behind NAT that forgets idle
  UDP mappings; a 25-second heartbeat keeps the return path open. (Server
  doesn't need it — it has a public IP.)
- Server keypair never changes here — clients address the server by its
  *public* key + `Endpoint = 51.79.145.116:51820`.

### 11.4 Install `qrencode` (for phone onboarding)

```bash
which qrencode || sudo apt install -y qrencode   # ~52 KB, instant
```

### 11.5 Build the two client profiles

Same keys, different `AllowedIPs` (**client side** = routing decision, §4):

`clients/phone.conf` — **full tunnel** (privacy + blocking):

```ini
[Interface]
PrivateKey = <PHONE_PRIVATE_KEY from /tmp/phone.key — see clients/ folder>
Address = 10.77.0.3/24
DNS = 10.77.0.1

[Peer]
PublicKey = uGM/3c6XTSNyU/ItYV8jBWPMiI1glOadBHOiuS5zHyU=
Endpoint = 51.79.145.116:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

`clients/phone-dnsonly.conf` — **DNS-only split tunnel** (residential IP +
blocking). Identical except one line:

```ini
AllowedIPs = 10.77.0.0/24
```

Line-by-line rationale:

- `Address = 10.77.0.3/24`: the phone's VPN identity. Must be inside the
  server's `/24` and unique per device (Mac = `.2`, phone = `.3`).
- `DNS = 10.77.0.1`: **the whole point** — the phone sends DNS to Pi-hole
  *through the tunnel*. Without this line, the phone would use the
  carrier/default DNS and Pi-hole would see nothing.
- `Endpoint`: server's public IP + WireGuard's UDP port. If the VPS IP ever
  changes, this is the one line to update on every client.
- Full-tunnel `AllowedIPs = 0.0.0.0/0, ::/0`: default-route everything via
  VPS → websites see `51.79.145.116`, ISP sees ciphertext. Downside: sites
  that block datacenter IPs (some banks, Netflix) may refuse you.
- DNS-only `AllowedIPs = 10.77.0.0/24`: only VPN-subnet traffic uses the
  tunnel → websites see your **residential IP**, ad blocking still works
  (DNS still goes to Pi-hole), ISP sees destination IPs but not query
  contents. Generated via:
  ```bash
  sed 's|AllowedIPs = 0.0.0.0/0, ::/0|AllowedIPs = 10.77.0.0/24|' \
    /tmp/phone.conf > /tmp/phone-dnsonly.conf
  ```

### 11.6 Onboard the phone (QR scan)

On the VPS terminal (you must be looking at it — QR is a picture):

```bash
qrencode -t ansiutf8 < /tmp/phone.conf          # full tunnel
qrencode -t ansiutf8 < /tmp/phone-dnsonly.conf  # DNS-only
```

In the WireGuard app: `+` → *Scan from QR code*. Both profiles can coexist
in the app (activate one at a time). **Third option, no extra profile:**
Android WireGuard → tunnel → ⋮ → *Exclude applications* (e.g. banking,
Netflix bypass the VPN with residential IP while the rest stays tunneled).

> Housekeeping already done: the key material was copied to
> `clients/` and `/tmp/phone.key` + `/tmp/phone.pub` deleted (the `.conf`
> files in `/tmp` remain until reboot; the `clients/` copies are the
> permanent ones).

---

## 12. Step 7 — The "VPN connects but no internet" NAT fix

### 12.1 Symptoms (learn to recognise this failure mode)

Phone shows WireGuard **connected** (fresh handshake in `sudo wg show wg0
latest-handshakes`) but **nothing loads**. Transfer counters tell the story:

```
# BROKEN:  phone sent 796 KB up, got 9 KB back  (requests leave, replies lost)
xfXe...  796420  9260
# FIXED:   megabytes both directions
xfXe...  1303076 8562188
```

**Rationale:** handshake OK + RX growing + TX flat ≈ always NAT/forwarding.
The tunnel (crypto, keys, UDP reachability) is proven by the handshake; the
*routing* past the tunnel is what's broken.

### 12.2 Root causes found (two, both required fixing)

1. **No MASQUERADE for `10.77.0.0/24`.** `iptables -t nat -L POSTROUTING`
   showed MASQUERADE rules only for Docker's `172.x` subnets. VPN packets
   left `ens3` still sourced `10.77.0.x` — unroutable on the internet, so
   replies never came back.
2. **FORWARD chain defaults to DROP** (Docker sets that) with no accept path
   for `wg0 ↔ ens3`. Even with NAT, packets would be filtered between
   interfaces.

(`net.ipv4.ip_forward` was already `1` — if it were `0`, that would be a
third cause. Check it first in any future debug: `sysctl
net.ipv4.ip_forward`.)

### 12.3 The fix (live rules + reboot persistence)

```bash
# NAT: rewrite VPN-client sources to the VPS public IP on the way out
sudo iptables -t nat -A POSTROUTING -s 10.77.0.0/24 -o ens3 -j MASQUERADE

# Forwarding: accept VPN↔internet in BOTH directions.
# DOCKER-USER is Docker's designated admin chain — custom rules here survive
# 'docker' restarts and never touch container networking (Caddy/Navidrome safe).
sudo iptables -I DOCKER-USER -i wg0 -o ens3 -j ACCEPT
sudo iptables -I DOCKER-USER -i ens3 -o wg0 -m conntrack \
  --ctstate RELATED,ESTABLISHED -j ACCEPT
```

**Rationale for each rule:**

- The MASQUERADE is `-o ens3`-scoped (not global) so Docker's own NAT is
  unaffected.
- The `wg0 → ens3` ACCEPT lets client traffic out.
- The `ens3 → wg0` rule allows **only reply traffic** (`RELATED,
  ESTABLISHED`) back in — the VPS still can't be *entered* via `wg0` from
  the internet side. This asymmetry is deliberate.
- `-I` (insert at top) puts our rules before Docker's jumps.

**Persistence across reboots** — live `iptables` rules evaporate on reboot,
and Docker re-creates its chains at startup, so we re-apply via WireGuard
itself. Added under `[Interface]` in `/etc/wireguard/wg0.conf` (inserted
right after `ListenPort`):

```ini
PostUp = iptables -t nat -A POSTROUTING -s 10.77.0.0/24 -o ens3 -j MASQUERADE; iptables -I DOCKER-USER -i wg0 -o ens3 -j ACCEPT; iptables -I DOCKER-USER -i ens3 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.77.0.0/24 -o ens3 -j MASQUERADE; iptables -D DOCKER-USER -i wg0 -o ens3 -j ACCEPT; iptables -D DOCKER-USER -i ens3 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

**Rationale:** `wg-quick@wg0` is `enabled`, so every boot runs `PostUp`
*after* the interface exists — self-healing by construction, no
`iptables-persistent` package needed (which could fight Docker's dynamic
rules). `PostDown` (`-D` = delete) keeps `wg0` restarts idempotent — no
duplicate rules stacking up. Validate after any edit:
`sudo wg-quick strip /etc/wireguard/wg0.conf >/dev/null && echo CONF_OK`.

Verify the fix any time:

```bash
sudo iptables -t nat -L POSTROUTING -n | grep 10.77   # expect MASQUERADE line
sudo iptables -L DOCKER-USER -n | head -6             # expect our 2 ACCEPTs
sudo wg show wg0 transfer                            # both counters growing?
```

---

## 13. Step 8 — Lockdown: closing the public DNS resolver

**Rationale:** during setup we set `dns.listeningMode = "ALL"` to prove
external DNS worked. That made the box an **open resolver** (§4) — an
abuse magnet that can get the VPS suspended. The fix is one setting, because
our threat model doesn't need public DNS at all: the phone uses the tunnel,
and a home router would too (§15.4). Nobody legitimate queries port 53 from
the raw internet.

```bash
sudo pihole-FTL --config dns.listeningMode "LOCAL"
sudo systemctl --no-block restart pihole-FTL   # --no-block: see §10.4 hang gotcha
sleep 8
systemctl is-active pihole-FTL                 # expect: active
```

**Why the tunnel keeps working:** `LOCAL` means "answer sources on local
subnets". `10.77.0.3` (phone via `wg0`) *is* a local subnet — same as
localhost. Only random internet IPs get dropped. Both phone profiles
(full-tunnel and DNS-only) send DNS to `10.77.0.1` through the tunnel, so
both are unaffected.

Prove it (from your laptop, not the VPS):

```bash
dig +short +time=4 +tries=1 @51.79.145.116 google.com
# → "connection timed out; no servers could be reached"  = SUCCESS (strangers dropped)
```

And on the VPS, local resolution still works:

```bash
dig +short @127.0.0.1 google.com   # → real IPs
```

> If you ever need public DNS again (e.g. a device that can't run
> WireGuard): prefer a router-side WireGuard client (§15.4) over reopening
> port 53. If you must reopen, allowlist specific IPs in iptables — never
> `listeningMode ALL` naked.

---

## 14. Verification checklist (run after every rebuild or reboot)

On the VPS:

```bash
systemctl is-active pihole-FTL unbound          # → active / active
systemctl is-enabled wg-quick@wg0 pihole-FTL unbound  # → enabled × 3
sysctl net.ipv4.ip_forward                      # → 1
sudo ss -tlnp | grep -E ':8888'                 # → pihole-FTL on 0.0.0.0:8888
sudo ss -ulnp | grep -E ':53|:51820'           # → pihole-FTL :53, kernel :51820
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8888/admin  # → 30x
dig +short @127.0.0.1 google.com                # → real IPs (Pi-hole→Unbound chain)
dig +short @127.0.0.1 -p 5335 google.com       # → real IPs (Unbound alone)
sudo wg show wg0 latest-handshakes              # → fresh timestamps for online peers
sudo iptables -t nat -L POSTROUTING -n | grep 10.77  # → MASQUERADE line
sudo pihole-FTL --config dns.listeningMode     # → LOCAL (not ALL!)
docker ps --format "{{.Names}}: {{.Status}}"    # → caddy/navidrome/songbot Up
```

From the outside (laptop):

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://51.79.145.116:8888/admin/  # → 30x
dig +short +time=4 +tries=1 @51.79.145.116 google.com  # → TIMEOUT (good!)
```

On the phone (either WireGuard profile active): browse — pages load, ads
gone from ad-heavy sites (test with a banner-farm news site, not YouTube —
§3). Pi-hole dashboard **Query Log** should show the phone's queries
arriving from `10.77.0.3` in real time.

---

## 15. Daily operation handbook

### 15.1 Admin UI

- URL: `http://51.79.145.116:8888/admin` (note `:8888`, plain `http` — typing
  `https` or omitting the port reproduces the original `ERR_SSL_PROTOCOL_ERROR`).
- Password: set at install; change any time with `sudo pihole setpassword`.
- Useful pages: *Dashboard* (blocked % over time), *Query Log* (live per-device
  stream — your #1 debugging view), *Group Management → Domains* (manual
  allow/deny), *Settings → DNS* (interface mode — leave it).

### 15.2 Which phone profile when

| Situation | Use | Why |
|---|---|---|
| Public Wi-Fi / mobile data, general browsing | Full tunnel | ISP-proof + blocking |
| Bank / Netflix / site blocks datacenter IPs | DNS-only (or per-app exclusion) | Residential IP + blocking |
| Home Wi-Fi with router VPN (§15.4) | WireGuard **off** | Router already covers you; avoid double-tunnel weirdness |

### 15.3 Keeping blocklists fresh

```bash
pihole -g          # update gravity (adlists) — run monthly or when ads reappear
pihole status      # health of FTL/DNS subsystems
pihole -t          # live-tail the query log in the terminal
```

### 15.4 Optional: whole-home coverage via the router

Instead of configuring every device, run **one** WireGuard client for the
home (many routers support it natively; else a cheap travel router or
Raspberry Pi as the LAN's WireGuard gateway + DNS forwarder to
`10.77.0.1`). Then every device at home gets Pi-hole with zero per-device
setup — and port 53 stays closed to the world (§13) because home queries
arrive via the tunnel.

### 15.5 Backups (what to copy off the box before any risky change)

```bash
/etc/wireguard/wg0.conf            # server keys + peers + PostUp/PostDown
/etc/pihole/pihole.toml            # ALL Pi-hole settings (v6 single-file config)
/etc/unbound/unbound.conf.d/pi-hole.conf
/var/lib/unbound/root.hints        # re-downloadable (§9.1), but copy anyway
~/workspace/pihole_setup/          # this manual + clients/
```

### 15.6 Reboot survival (why a reboot is a non-event now)

| Thing | Survives reboot because |
|---|---|
| WireGuard server | `wg-quick@wg0` is `enabled` |
| NAT/forward rules | `PostUp` in `wg0.conf` re-applies them |
| Pi-hole settings | `pihole.toml` on disk |
| Unbound config | files on disk |
| `root.hints` | file on disk |
| Phone profiles | `clients/` folder (NOT `/tmp` — those vanish!) |
| Docker services | Docker restart policy (untouched by us) |

---

## 16. Rebuild-from-scratch: the short ordered version

Assumes fresh Ubuntu, SSH access, nothing else installed. (Details and
*whys* are in the sections referenced.)

```bash
# 0. In and updated (§6)
ssh ubuntu@<NEW_VPS_IP>
sudo apt update && sudo apt upgrade -y

# 1. Pi-hole (§7–§8). If the installer sticks at pihole-meta.deb: kill, rm the
#    658-byte file, retry; else git method (quote the space in the path!).
curl -sSL https://install.pi-hole.net | bash
# installer prompts: interface=<public NIC, see `ip addr`>,
#   upstream=Custom 127.0.0.1#5335, logging=Yes. Save the admin password.

# 2. Unbound (§9)
sudo apt install unbound -y
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf   # paste §9 config
sudo wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
sudo unbound-checkconf                                # must pass BEFORE restart
sudo systemctl restart unbound
dig @127.0.0.1 -p 5335 google.com                     # expect NOERROR + answers

# 3. dodge port conflicts (§10.2): check owners FIRST
sudo ss -tlnp | grep -E ':80|:443|:8081'
sudo pihole-FTL --config webserver.port "80o,443os,[::]:80o,[::]:443os,8888o"
# (pick any free port instead of 8888 if taken)

# 4. listeningMode ALL only to test, then back to LOCAL (§10.3, §13)
sudo pihole-FTL --config dns.listeningMode "ALL"
sudo systemctl --no-block restart pihole-FTL; sleep 8; systemctl is-active pihole-FTL
dig +short @<PUBLIC_IP> google.com                    # should answer (temporary!)

# 5. WireGuard server (§11.1–11.3)
sudo apt install wireguard -y
# (or reuse existing wg0 — check `sudo wg show` first!)
# generate server keys if new: wg genkey | sudo tee /etc/wireguard/server.key ...
# create /etc/wireguard/wg0.conf with Address/Port/PostUp (§12.3!) + peers

# 6. NAT + forwarding (§12.3) — do it IN wg0.conf PostUp from the start,
#    plus apply live once. Verify with `sudo wg show wg0 transfer`.

# 7. Client profiles (§11.5–11.6): phone.conf (full) + phone-dnsonly.conf,
#    qrencode, scan, test browsing.

# 8. Lockdown (§13): listeningMode back to LOCAL, non-blocking restart,
#    external dig must TIME OUT, tunnel browsing must WORK.

# 9. Enable at boot + final checklist (§14)
sudo systemctl enable --now wg-quick@wg0   # if newly created
sudo apt install -y qrencode
```

---

## 17. Troubleshooting matrix

| Symptom | Likely cause | Fix (§) |
|---|---|---|
| Installer stuck at `pihole-meta.deb` for 10+ min | 658-byte failed download; apt wedged | Kill, `rm /tmp/pihole-meta.deb`, retry or git method (§7.2–7.3) |
| `basic-install.sh: No such file` | Space in `automated install/` path | Quote the path (§7.3) |
| `pihole -a -i all` prints help, changes nothing | v5 flag removed in v6 | Use `listeningMode` / web UI (§10.1) |
| `unbound.service failed`, crash-loop | Missing `root.hints` (or config typo) | `unbound-checkconf`, wget hints (§9.1); fix typos (§9.2) |
| Browser `ERR_SSL_PROTOCOL_ERROR` on `/admin` | Caddy owns :80 and redirects; bare-IP TLS fails | Use `http://IP:8888/admin` (§10.2) |
| `curl :80/admin` → `Server: Caddy` 308 | Same port fight | Same fix — never fight Docker for 80/443 |
| `systemctl restart pihole-FTL` freezes SSH | systemd holds the channel | `--no-block` + poll (§10.4) |
| External `dig @VPS` times out | `listeningMode LOCAL` (correct!) or FTL down | If lockdown intended: success. If not: check FTL + mode (§10.3) |
| VPN handshake fresh, no internet | Missing MASQUERADE / FORWARD DROP | §12 (check `transfer` counters) |
| VPN no handshake at all | Wrong keys, wrong `Endpoint`, UDP 51820 blocked, peer offline | Compare `wg show` both ends; `Endpoint` = current public IP:51820 |
| Internet works, ads NOT blocked on phone | Phone DNS bypassing tunnel (no `DNS = 10.77.0.1`, or app uses DoH) | Check profile's `DNS` line; test with `dig @10.77.0.1` on tunnel |
| YouTube app still shows ads | Fundamental DNS limit, not a bug | SmartTube / ReVanced / uBlock / Premium (§3) |
| Site blocks you / demands captcha | It dislikes datacenter (VPS) IPs | DNS-only profile or per-app exclusion (§11.6) |
| Rules/peers gone after reboot | Only applied live, never persisted | Peers → `wg0.conf`; iptables → `PostUp` (§11.2, §12.3) |
| Web UI login loops / 404 | Wrong URL (missing `:8888` or trailing path) | Exact URL §15.1; `curl -v` to see redirect chain |

---

## 18. Credentials and file inventory

| Item | Value / location | Notes |
|---|---|---|
| VPS SSH | `ssh ubuntu@51.79.145.116` | Key auth |
| Pi-hole admin | `http://51.79.145.116:8888/admin` | Password set at install (`sudo pihole setpassword` to rotate) |
| Install log | `/etc/pihole/install.log` | What the installer did/chose |
| Pi-hole config | `/etc/pihole/pihole.toml` | v6 single-file config (root-owned) |
| Unbound config | `/etc/unbound/unbound.conf.d/pi-hole.conf` | Short canonical config (§9) |
| Root hints | `/var/lib/unbound/root.hints` | From `https://www.internic.net/domain/named.root` |
| WireGuard server conf | `/etc/wireguard/wg0.conf` | Includes PostUp/PostDown; backup `wg0.conf.bak` |
| WireGuard server pubkey | `uGM/3c6XTSNyU/ItYV8jBWPMiI1glOadBHOiuS5zHyU=` | Goes in every client `[Peer]` |
| WireGuard endpoint | `51.79.145.116:51820` (UDP) | Update clients if VPS IP changes |
| VPN subnet | `10.77.0.0/24` (server `.1`, Mac `.2`, phone `.3`, PC `.4`, TV `.5`) | Next device takes `.6` |
| Phone profiles | `~/workspace/pihole_setup/clients/*.conf` | `600` perms; QR via `qrencode -t ansiutf8` |
| This manual | `~/workspace/pihole_setup/README.md` | You are here |

**Environment snapshot (2026-10-03):** Ubuntu 26.04.1 LTS · Unbound 1.24.2 ·
wireguard-tools v1.0.20250521 · Pi-hole v6 (official installer) ·
`wg-quick@wg0`, `pihole-FTL`, `unbound` all `enabled`+`active` · UFW
inactive · `net.ipv4.ip_forward=1` · Docker: `caddy` (:80/:443), `navidrome`
(:4533), `songbot-botapi-1` (localhost :8081) — all untouched.

---

*End of manual. If you're rebuilding and something here is stale, update this
file as you go — future-you is the primary reader.*

### Client profiles added 2026-10-03 (PC + smart TV)
| `clients/pc.conf` | Full-tunnel profile for the PC (10.77.0.4) — import into WireGuard for Mac | `600` |
| `clients/tv.conf` | DNS-only profile for the smart TV (10.77.0.5) — blocking without routing streaming via VPS | `600` |

### More profiles added 2026-10-03 (every device now has both modes)

Each device below has two interchangeable profiles (same keys, toggle in the app):

- `clients/phone.conf` / `clients/phone-dnsonly.conf` (10.77.0.3)
- `clients/pc.conf` / `clients/pc-dnsonly.conf` (10.77.0.4)
- `clients/tv.conf` (DNS-only) / `clients/tv-full.conf` (full tunnel) (10.77.0.5)
