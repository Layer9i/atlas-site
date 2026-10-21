# Atlas Identity — IPv4 Geolocation Diagnostic

**Subject:** IPinfo places Layer9i regional IPv4 egress addresses in New York
**Network:** Layer9i, AS403034, 204.68.238.0/24 (IPv6 2602:f2be::/40)
**Investigation window:** 2026-10-21 14:26Z – 14:30Z (UTC)
**Mode:** Read-only. Nothing was changed: no routing, BGP, WireGuard, NAT, firewall, geofeed, DNS, interface or application changes, and no service restarts.

---

## 0. Read this first — what this investigation could and could not reach

This investigation ran in a sandboxed cloud container. It had **no access to Atlas infrastructure and no general Internet access**. I'm stating that plainly so no reader mistakes the gaps below for results.

| Access needed | Result | Evidence |
|---|---|---|
| SSH / console to Vultr instances (NY, Toronto, Amsterdam, London) | **Not available.** No credentials, hosts or inventory were provided, and this repository holds none. | — |
| Vultr API / portal | **Not available.** | — |
| Public web: layer9i.com, ipinfo.io, stat.ripe.net, rdap.arin.net, bgp.tools, bgp.he.net, bgpview, PeeringDB, RIPE Atlas, dns.google, MaxMind, ip-api, ipapi.is, ipwho.is, ipapi.co, Cloudflare Radar, lg.vultr.com | **Blocked by the sandbox egress proxy (HTTP 403 on CONNECT).** Every one was refused. | Proxy log in §0.1 |
| WebFetch tool (separate path) | **Failed: no DNS** (`getaddrinfo ENOTFOUND`) for layer9i.com, ipinfo.io, stat.ripe.net, rdap.arin.net, peeringdb.com | §0.1 |
| Browser automation tool | **Failed to start** (daemon did not come up) | §0.1 |
| ICMP / traceroute / dig / whois / mtr | **Not installed**, and raw sockets do not leave the sandbox anyway | `which` returned nothing for these |
| Other Layer9i repositories (e.g. `Layer9i/layer9i-research`, the backend holding `own_addresses`) | **Not attached.** The request to attach `layer9i-research` read-only was denied by the session's permission policy. The Atlas backend/ops repository is not in this account's repo list. | — |
| Web search (search-engine summaries only) | Available. Used only for published provider and Vultr documentation, not for live lookups. | §10 |
| GitHub: `Layer9i/atlas-site` git history and GitHub Pages deployment runs | **Available.** This is the main evidence source. | §4, §5 |

**So this report contains no live geolocation lookups, BGP data, traceroutes, latency figures or server-side command output.** Wherever a section asks for those, the report says they weren't collected and gives the exact command to run (§9). Nothing has been made up or estimated to fill those gaps.

### 0.1 Raw access evidence

```
$ curl -sS "$HTTPS_PROXY/__agentproxy/status" | jq '.recentRelayFailures[].host'   # 2026-10-21T14:26:51Z–14:27:02Z
"bgp.tools:443" "api.bgpview.io:443" "dns.google:443" "atlas.ripe.net:443"
"www.peeringdb.com:443" "ip-api.com:443" "api.ipapi.is:443" "geolite.info:443"
"www.maxmind.com:443" "api.radar.cloudflare.com:443" "lg.vultr.com:443"
"ipwho.is:443" "ipapi.co:443" "freeipapi.com:443" "api.iplocation.net:443"
"globe.adsbexchange.com:443" "bgp.he.net:443" "api.ipinfo.io:443"
"ipinfo.io:443" "layer9i.com:443"
  -> each: "gateway answered 403 to CONNECT (policy denial or upstream failure)"

$ curl -sS -v https://layer9i.com/geofeed.csv
> CONNECT layer9i.com:443 HTTP/1.1
< HTTP/1.1 403 Forbidden
curl: (56) CONNECT tunnel failed, response 403

WebFetch https://layer9i.com/geofeed.csv        -> getaddrinfo ENOTFOUND layer9i.com
WebFetch https://ipinfo.io/204.68.238.20/json   -> getaddrinfo ENOTFOUND ipinfo.io
WebFetch https://stat.ripe.net/...              -> getaddrinfo ENOTFOUND stat.ripe.net
WebFetch https://rdap.arin.net/registry/ip/...  -> getaddrinfo ENOTFOUND rdap.arin.net
```

To run the live parts from a session like this one, the cloud environment's network access must allow those hosts. The setting is in the environment settings → Network access ([docs](https://code.claude.com/docs/en/cloud-environments#network-access)). Server-side checks still need operator shell access to the Vultr nodes.

---

## 1. Verified Atlas network architecture

### 1.1 What could be verified

The only Atlas network facts that could be checked come from the geofeed and the commit messages in this repository. Those commit messages record decisions the owner made ("Owner, 2026-10-07: …"). They are **documented intent, not observed configuration.**

| Item | Source | Status |
|---|---|---|
| ASN AS403034, holder "Layer9i Inc." | geofeed header; PeeringDB search summary (net 43823, name Layer9i, 1 IPv4 + 1 IPv6 prefix, open policy, updated 2026-10-08T18:31:36Z) | Documented; PeeringDB not fetched directly |
| IPv4 204.68.238.0/24 | geofeed header, commit bac0c70 | Documented |
| IPv6 2602:f2be::/40 | commit 0072a5b ("validated: inside the /40") | Documented |
| Site address plan (IPv4) | commit 585ff1b | Documented intent: **site N = 204.68.238.(8×N)/29**; +1/+2 kept for a firewall in front; **+3 = "the egress that peers BGP"**; +4…+6 growth; N=0 (.0/29) reserved for a future single egress |
| Site address plan (IPv6) | commit 0072a5b | Documented intent: site N = 2602:f2be:N::/48; ::1/::2 firewall; ::3 egress that peers BGP |
| IPv6 announcement status | commit 0072a5b, 2026-10-07: "**IPv6 is not announced yet**" | Documented as of Oct 7; current state unknown |
| Each egress "peers BGP" | commits 585ff1b, 0072a5b | Documented intent, so each regional node is designed to run its own BGP session with Vultr |
| Live cities | commit 4b3c7e7 (2026-10-08): "5 cities, own network"; commit 431ad9b (2026-10-21): "only the four live cities" (NY, Toronto, Amsterdam, London) | Documented |
| Backend address list | commit 431ad9b: geofeed "Matches the backend's own_addresses (test passes)" | That backend repo wasn't accessible |

### 1.2 Discrepancies between the brief and the documented plan

These matter for root cause and should be cleared up with the operator.

1. **The prefix size is /29, not /28.** The brief describes "regional /28 blocks". Every published geofeed version since 2026-10-07 uses **/29 per region**. The one exception is New York, which today is a /28 (.0–.15, created by merging the old .0/29 and .8/29).
2. **The Toronto egress .20 is not at the planned position.** Toronto is site 2 (.16/29), so the plan puts its egress at **.19** (+3). The brief gives **.20** (+4, the "growth" slot). That is not wrong in itself. But if the node's BGP session, NAT rule or `own_addresses` entry was built from the plan (.19) while traffic is SNAT'd to .20, or the other way round, the address actually in use may not be the one configured locally.
3. **The New York egress address has changed.** Commit 585ff1b (2026-10-07) says ".0/29 (**New York's current .1 egress**…)". The brief gives **.11** (site 1, +3, which matches the plan). The move from .1 to .11 isn't recorded here, so it is unconfirmed.

### 1.3 Not verified (no access)

Vultr instance IDs and regions, interface configs, the Vultr-assigned primary IPs, how 204.68.238.x is bound (loopback / dummy / alias), WireGuard topology (hub-and-spoke versus per-region termination), SNAT/PAT rules, and the BGP daemon config and announcements for each region. The commands needed are in §9.1.

---

## 2. IPv4 and IPv6 egress validation (Toronto 204.68.238.20)

**None of the Toronto checks could run: no shell on the node, and no outbound access to IP-echo services.** Nothing below is a measured result.

| Check | Configured behaviour | Confirmed by test |
|---|---|---|
| .20 assigned to the Toronto node | Not visible | **Not tested** |
| Local routing table | Not visible | **Not tested** |
| SNAT/PAT rules | Not visible | **Not tested** |
| Actual public source IP of outbound traffic | Brief says .20 | **Not tested** |
| IPv4 outbound path exits locally in Toronto | Design intent (brief) | **Not tested** |
| IPv6 outbound path | 2602:f2be:2::/48 per plan; egress expected at 2602:f2be:2::3 | **Not tested** |

What would settle it is a set of read-only commands on the Toronto node (§9.1). The three that matter most:

- `ip -4 route get 1.1.1.1 from 204.68.238.20`: does it leave via Toronto's own Vultr gateway, or via a tunnel (wg*) to another site?
- `curl -4 --interface 204.68.238.20 https://ipinfo.io/json`, plus a test from a real Atlas client pinned to the Toronto policy: what source address does the outside world actually see?
- `birdc show route export <peer>` (or the FRR equivalent): is the Toronto node announcing 204.68.238.0/24 to Vultr at all?

---

## 3. Public BGP findings

**No public BGP data could be retrieved.** RIPEstat, bgp.tools, bgp.he.net, bgpview, RouteViews and every looking glass were blocked, so origin AS, upstreams, visibility and RPKI status are all **unverified**.

### 3.1 Constraint that is documented (not a measurement)

Vultr's published BYOIP/BGP policy:

- **IPv4 prefixes must be /24 or shorter. IPv6 prefixes must be /48 or shorter.** ([Vultr BYOIP advisory](https://docs.vultr.com/platform/customer-advisory/byoip.md), [Vultr BGP docs](https://docs.vultr.com/products/network/bgp))
- Vultr announces the customer prefix to its upstreams **in whatever location the announcing instance is deployed**. For anycast across several locations, Vultr "cannot guarantee the even distribution of connections". ([Configuring BGP on Vultr](https://docs.vultr.com/configuring-bgp-on-vultr))

What follows from that (an inference, not a measurement):

- **The regional IPv4 /29s cannot be announced as separate routes.** On the public Internet, the only IPv4 route that can exist for 204.68.238.20 is the covering **/24**. Inbound packets for the Toronto /29 therefore go wherever the Internet's best path for the /24 points:
  - If only New York announces the /24, every inbound packet to .20 enters Vultr in New York.
  - If several sites announce the /24 (anycast), each source reaches its nearest announcing Vultr location. That site then has to own .20, or forward it to Toronto (over WireGuard or Vultr's backbone), or drop it.
- **The IPv6 /48s can be announced separately**, one per site. If they are (the brief says IPv6 geolocates correctly, but on Oct 7 IPv6 was "not announced yet"), inbound IPv6 for Toronto lands in Toronto, and measurement agrees with the geofeed.

This asymmetry matches the reported symptom exactly: **IPv6 is correct, IPv4 says New York.** It still has to be confirmed against real BGP and traceroute data (§9.2, §9.3).

### 3.2 What must be requested from Vultr

See §8.

---

## 4. Geofeed validation

### 4.1 Location and publication

- URL: `https://layer9i.com/geofeed.csv` (geofeed header; `CNAME` = `layer9i.com`; served by GitHub Pages from `main`).
- Commit #85 says Cloudflare sits in front of the site, with a 4-hour `max-age` seen on static files.
- **Live HTTP fetch: not possible (blocked).** HTTP status, `Content-Type`, TLS, redirects and Cloudflare cache headers are **unverified**.
- **ARIN `Geofeed:` reference: not checked** (rdap.arin.net and whois blocked).

### 4.2 Published versions, with deployment timestamps

The deployment times come from GitHub Pages "pages build and deployment" runs on `Layer9i/atlas-site` (all `conclusion: success`):

| Ver. | Commit | Live from (UTC) | Live until (UTC) | Duration | 204.68.238.20 mapped to |
|---|---|---|---|---|---|
| v1 | bac0c70 | 2026-10-06 21:05:04 | 2026-10-07 17:25:47 | ~20 h 21 m | **New York** (204.68.238.0/26) ⚠ |
| v2 | 585ff1b | 2026-10-07 17:25:47 | 2026-10-07 17:47:44 | ~22 m | Toronto (204.68.238.16/29) |
| v3 | 0072a5b | 2026-10-07 17:47:44 | 2026-10-21 12:49:29 | ~13 d 19 h | Toronto (204.68.238.16/29) |
| v4 | 431ad9b | 2026-10-21 12:49:29 | current (on `main`) | ~1.7 h at time of writing | Toronto (204.68.238.16/29) |

(The branch checked out for this investigation sits at v3. v4 is on `origin/main`, which is what Pages serves.)

**Finding: the first published geofeed told every consumer that the Toronto egress address .20 was in New York.** v1 split the /24 into /26s: .0/26 New York, .64/26 Toronto, .192/26 Amsterdam. Toronto's block started at .64, so .20 fell inside New York's /26. A provider that fetched the feed between Oct 6 21:05Z and Oct 7 17:25Z (plus any CDN cache time) received an authoritative "204.68.238.0–63 = New York, US-NY" record.

v1 content:

```
# Layer9i Inc. (AS403034) geofeed, RFC 8805. Published 2026-10-06.
# Contact: info@layer9i.com
# prefix,country,region,city,postal
204.68.238.0/26,US,US-NY,New York,
204.68.238.64/26,CA,CA-ON,Toronto,
204.68.238.192/26,NL,NL-NH,Amsterdam,
```

Current content (v4, `origin/main`):

```
# Layer9i Inc. (AS403034) geofeed, RFC 8805. Updated 2026-10-21.
# Contact: info@layer9i.com
# prefix,country,region,city,postal
204.68.238.0/28,US,US-NY,New York,
204.68.238.16/29,CA,CA-ON,Toronto,
204.68.238.24/29,NL,NL-NH,Amsterdam,
204.68.238.40/29,GB,GB-ENG,London,
2602:f2be:1::/48,US,US-NY,New York,
2602:f2be:2::/48,CA,CA-ON,Toronto,
2602:f2be:3::/48,NL,NL-NH,Amsterdam,
2602:f2be:5::/48,GB,GB-ENG,London,
```

### 4.3 Format and RFC 8805 validation (offline, from the git copies)

I checked every version with a local validator. It tests field count, strict CIDR form, ISO 3166-1 country, ISO 3166-2 region prefix, containment in the allocation, overlaps, BOM, CRLF and trailing newline.

```
### 431ad9b.csv  bytes=414 BOM=False CRLF=False trailing_newline=True non_ascii=False
  204.68.238.0/28      US US-NY   New York     hosts .0-.15
  204.68.238.16/29     CA CA-ON   Toronto      hosts .16-.23
  204.68.238.24/29     NL NL-NH   Amsterdam    hosts .24-.31
  204.68.238.40/29     GB GB-ENG  London       hosts .40-.47
  2602:f2be:1::/48 US-NY New York | 2602:f2be:2::/48 CA-ON Toronto
  2602:f2be:3::/48 NL-NH Amsterdam | 2602:f2be:5::/48 GB-ENG London
  204.68.238.11 (New York egress) -> 204.68.238.0/28  New York
  204.68.238.20 (Toronto egress)  -> 204.68.238.16/29 Toronto
  IPv4 addresses covered: 40/256; v4 entries 4, v6 entries 4
  ERRORS: none

### bac0c70.csv (v1)
  204.68.238.20 (Toronto egress)  -> 204.68.238.0/26  New York     <-- wrong for Toronto
  ERRORS: none (syntactically valid)
```

| Check (current v4) | Result |
|---|---|
| CSV structure: 5 fields, `#` comments, UTF-8/ASCII, LF, no BOM | Pass |
| Prefixes in canonical CIDR form | Pass |
| Country codes (US, CA, NL, GB) | Pass |
| ISO 3166-2 regions (US-NY, CA-ON, NL-NH, GB-ENG) | Pass |
| City names | Pass ("New York", "Toronto", "Amsterdam", "London") |
| Postal field empty | Allowed by RFC 8805 (and recommended against for privacy) |
| All entries inside 204.68.238.0/24 or 2602:f2be::/40 | Pass |
| Overlapping or conflicting entries | **None** |
| Coverage | 40/256 IPv4 addresses. The rest of the /24 (.32–.39, .48–.255) has no geofeed entry, so providers fall back to their own inference for it (often the /24 as a whole). |
| HTTP availability, content type, ARIN reference | **Not verified (blocked)** |

**Assessment:** the current geofeed is syntactically correct and internally consistent. That matches what IPinfo support reported. The problem is unlikely to be the current file's format. The problems are the **history** (v1's wrong mapping), the **churn** (four versions in 15 days, two of them listing cities that weren't serving), and the **gap between what the feed claims and where packets for the /24 actually land**.

---

## 5. Provider geolocation comparison

**No provider lookups could be run.** ipinfo.io, api.ipinfo.io, MaxMind (www.maxmind.com, geolite.info), ip-api, ipapi.is, ipwho.is, ipapi.co, freeipapi and iplocation.net were all blocked (§0.1). Search engines had indexed nothing for these addresses.

| Address | Expected | IPinfo | MaxMind | Others | Timestamp |
|---|---|---|---|---|---|
| 204.68.238.11 | New York, US-NY | not retrieved | not retrieved | not retrieved | — |
| 204.68.238.20 | Toronto, CA-ON | not retrieved (reported by Layer9i: New York) | not retrieved | not retrieved (reported by Layer9i: correct) | — |
| 2602:f2be:1::3 (NY, planned egress) | New York | not retrieved (reported: correct) | not retrieved | not retrieved | — |
| 2602:f2be:2::3 (Toronto, planned egress) | Toronto | not retrieved (reported: correct) | not retrieved | not retrieved | — |

The cells marked "reported" restate the problem statement. This investigation didn't observe them. The script to fill this table is in §9.4.

### 5.1 IPinfo's published methodology

From search-engine summaries of IPinfo publications. These are documented positions, not observations about Layer9i.

- IPinfo treats geofeeds as "**one input among many, weighted by how well they align with observable evidence**" ([IPinfo community: Trust-based vs evidence-based](https://community.ipinfo.io/t/trust-based-vs-evidence-based-geolocation/7208); [Why verification matters](https://community.ipinfo.io/t/ip-geolocation-and-geofeeds-why-verification-matters/7216)).
- **Active measurement is the primary source, and the geofeed is the fallback** "when measurement data is noisy or unavailable" ([IPinfo community: Geofeed as a fallback](https://community.ipinfo.io/t/why-geofeed-works-as-a-fallback-not-a-foundation/7209)).
- **When measurement and geofeed conflict, measurement wins.** IPinfo has also said that, because of peering and topology, active measurement "can be noisy for some ASNs", and it recommends **hosting an IPinfo probe server on your ASN** ([NANOG list, Feb 2026](https://seclists.org/nanog/2026/Feb/113), a secondary account).
- ProbeNet: 1,360+ points of presence; **every IP re-verified weekly** ([ipinfo.io/probenet](https://www.ipinfo.io/probenet)).

These policies explain the "correct for about an hour, then reverted" behaviour well (see hypothesis B in §6). A geofeed update can briefly take effect until the next measurement pass overrides it.

---

## 6. Hypotheses — confirmed findings versus open hypotheses

### 6.1 Confirmed findings (direct evidence in this investigation)

| # | Finding | Evidence |
|---|---|---|
| F1 | The first published geofeed (v1, live 2026-10-06 21:05Z → 2026-10-07 17:25Z) mapped **204.68.238.0/26, including the Toronto egress .20, to New York**. | git `bac0c70:geofeed.csv`; Pages runs 37531204367 and 37658877121 |
| F2 | The current geofeed (v4) is RFC 8805-valid, has no overlaps, and maps .20 → Toronto and .11 → New York. | §4.3 validator output |
| F3 | The geofeed has changed four times in 15 days. Two versions (v2 and v3, about 14 days in total) listed five cities that weren't serving (Montreal, Chicago, Los Angeles, Vancouver, Frankfurt). | git history and Pages runs |
| F4 | Regional IPv4 blocks are **/29**, not /28 as the brief says. | all geofeed versions ≥ v2 |
| F5 | Vultr accepts IPv4 announcements **no longer than /24**, so the /29s can't be routed separately on the Internet. IPv6 /48s can. | Vultr published documentation |
| F6 | The Toronto egress in use (.20, per the brief) is not the plan's egress slot (.19). New York's egress moved from .1 (Oct 7) to .11 (brief). | commit 585ff1b vs. brief |
| F7 | IPinfo's published policy is that measurement overrides a conflicting geofeed. | IPinfo publications (§5.1) |

### 6.2 Hypothesis evaluation

| Hyp. | Statement | Evidence for | Evidence against | Status |
|---|---|---|---|---|
| **A** | IPinfo ignores or aggregates /28 (/29) geofeed entries | Nothing direct. IPinfo's database is commonly organized around routed prefixes, and the only routed IPv4 prefix here is the /24. | IPinfo did show the correct regional location for about an hour, so it can represent the sub-/24 entries. | **Unlikely as the sole cause.** Ask IPinfo whether sub-/24 records exist for 204.68.238.0/24. |
| **B** | IPinfo's measurement system overrides the geofeed | Published policy (F7). Symptom pattern: correct for about an hour, then reverted, then stable on New York, which fits a feed update followed by a measurement re-verification. IPv6 is unaffected. | No ProbeNet RTT data obtained. | **Strongly supported, not confirmed.** Most likely mechanism. |
| **C** | IPv4 BGP architecture produces misleading measurements | F5: the /24 is the only IPv4 route, so all inbound IPv4 probes enter Vultr wherever the /24 is announced. If New York announces it (alone or nearest to most probes), probes to .20 get their lowest RTT from New York. IPv6 /48s can be per-site, which would explain why IPv6 measures correctly. | Announcement locations not observed. | **Strongly supported structurally, not confirmed.** Most likely underlying cause that **feeds B**. |
| **D** | Historical data overrides current data | F1: a provider that ingested v1 has an authoritative "Toronto egress = New York" record. F3: churn may have lowered trust in the feed. | v3/v4 have said Toronto for 14 days, so a weekly re-verify should have replaced history if measurement agreed. | **Plausible contributing factor.** Ask IPinfo whether v1 is still held. |
| **E** | Geofeed accepted but not incorporated into production | The roughly 1-hour correct display shows it *was* incorporated at least once. | Same observation: it went live, then was overridden. | **Weak alone.** Better described as "incorporated, then superseded" (B). |
| **F** | Regional IPv4 egress or return routing differs from the design | F6 (address slots differ from the plan). If replies or inbound to Toronto's .20 go through New York (a /24 announced only in New York with tunnelling to Toronto, or asymmetric return), the "Toronto" address really does behave like a New York host from outside. | No server-side or traceroute data. | **Plausible, not tested.** It's the same physical cause as C, seen from the operator's side. |

---

## 7. Most likely root cause

**Most likely (not confirmed):** a combination of **C + F producing B**, with **D** possibly contributing.

1. Vultr only routes IPv4 at /24 or shorter, so there is no public route that says "204.68.238.16/29 is in Toronto". Every inbound packet to 204.68.238.20, including every IPinfo ProbeNet ping or traceroute, follows the route for **204.68.238.0/24**.
2. If that /24 is announced from New York, or announced from several sites but answered or forwarded via New York, probes see the lowest latency from the New York area. Measurement-based geolocation then concludes "New York".
3. IPinfo's policy is to let measurement override a conflicting geofeed (F7). It briefly applied the feed (about an hour correct), then its measurement pass overrode it back to New York.
4. IPv6 doesn't have this problem because each site can announce its own /48, so measurement and feed agree. (This assumes the /48s are now announced per site; on 2026-10-07 they weren't announced at all.)
5. The v1 geofeed (F1), which told IPinfo for about 20 hours that the Toronto egress was in New York, and the heavy churn of reserved-city entries (F3) may have lowered the feed's weight and left a conflicting historical record.

Two checks would confirm or reject this, ideally run together:
- **(a)** BGP collector data showing *where* AS403034's /24 is announced, plus a Vultr confirmation of announcement locations per instance.
- **(b)** RTT to 204.68.238.20 from probes in Toronto and New York. If New York probes see lower RTT than Toronto probes, C is confirmed.

---

## 8. Evidence needed from Vultr and IPinfo

### 8.1 From Vultr (support ticket, BGP team)

1. For each Atlas instance (NY, Toronto, Amsterdam, London): instance ID, Vultr location, and whether a BGP session to AS64515 / Vultr's route servers is **established**.
2. Which instances are currently **announcing 204.68.238.0/24**, and which are announcing each **2602:f2be:N::/48**.
3. For the /24: does Vultr propagate it to upstreams **from every announcing location (anycast)** or only one? How does Vultr route traffic for the /24 *internally* between its locations?
4. Does Vultr accept more-specifics (/25–/32) **internally**, with no export to upstreams, so that traffic for 204.68.238.16/29 entering Vultr anywhere is carried to Toronto? If yes, which communities control this?
5. Vultr's upstream/transit providers at each location, and any BGP communities available for selective announcement, prepending or no-export.
6. Reverse DNS / SWIP delegation state for the /24 (only as supporting data).

### 8.2 From IPinfo (support / DevRel)

1. Their current record for **204.68.238.20** and **204.68.238.11**: city, *data source used* (geofeed, ProbeNet measurement, WHOIS or other), and record timestamp.
2. The **geofeed version they hold**: fetch timestamp and content. Specifically, do they still hold the v1 (Oct 6) entry `204.68.238.0/26,US,US-NY,New York`?
3. Whether the geofeed entry was **rejected or overridden by measurement**, and the measurement evidence (probe locations and RTTs) that drove the override.
4. Whether they keep **sub-/24 records** for 204.68.238.0/24 or treat it as a single routed unit.
5. What happened during the roughly 1-hour window when the correct location appeared, and why it reverted.
6. Whether hosting an **IPinfo probe server** in AS403034 (at the Toronto site) would anchor measurements, which is IPinfo's own recommendation.

---

## 9. Tests that could not be completed — ready-to-run commands

All of these are read-only and non-disruptive.

### 9.1 On each egress node (operator shell)

```bash
date -u +%FT%TZ; hostname; curl -s http://169.254.169.254/v1.json | jq '{region:.region,instanceid:."instance-v2-id"}'
ip -br addr; ip -4 route show table all; ip -6 route show table all; ip rule show
ip -4 route get 1.1.1.1; ip -4 route get 1.1.1.1 from 204.68.238.20
ip -6 route get 2606:4700:4700::1111
sudo wg show all dump | awk '{print $1,$3,$4,$5}'       # no private keys: field 2 omitted
sudo nft list ruleset | grep -nE 'snat|masquerade|dnat'  # or: sudo iptables -t nat -S
sudo birdc show protocols all; sudo birdc show route export <vultr_peer> all   # or: vtysh -c 'show bgp summary' -c 'show bgp ipv4 unicast neighbors <peer> advertised-routes'
curl -4s https://ipinfo.io/json; curl -6s https://v6.ipinfo.io/json
curl -4s --interface 204.68.238.20 https://ipinfo.io/ip   # Toronto: confirms which source IP the outside world sees
```

Repeat the source-IP check from a **real Atlas mobile client** pinned to each region's policy.

### 9.2 Public BGP

```bash
curl -s "https://stat.ripe.net/data/routing-status/data.json?resource=204.68.238.0/24"
curl -s "https://stat.ripe.net/data/looking-glass/data.json?resource=204.68.238.0/24" | jq '.data.rrcs[] | {rrc:.rrc,location:.location,paths:[.peers[].as_path]}'
curl -s "https://stat.ripe.net/data/bgp-state/data.json?resource=2602:f2be::/40"
curl -s "https://stat.ripe.net/data/rpki-validation/data.json?resource=AS403034&prefix=204.68.238.0/24"
curl -s "https://stat.ripe.net/data/asn-neighbours/data.json?resource=AS403034"
# Also: https://bgp.tools/prefix/204.68.238.0/24, https://bgp.he.net/AS403034, RouteViews telnet route-views.routeviews.org
```

Compare the AS paths seen at RRC collectors in Toronto (TorIX), New York, Amsterdam and London. A path through AS20473 alone doesn't reveal the announcing *city*. What does: Vultr's confirmation, or RTT from a collector-adjacent probe, or whether the last Vultr hop's rDNS/IXP is consistent everywhere.

### 9.3 Measurements (RIPE Atlas, one-off, no agents installed)

```bash
# ping + traceroute to 204.68.238.20 and 2602:f2be:2::3 from probes in CA-Toronto, US-NY, NL-Amsterdam, GB-London
ripe-atlas measure ping  --target 204.68.238.20 --from-area WW --probes 20 --from-country CA
ripe-atlas measure traceroute --target 204.68.238.20 --protocol ICMP --from-country US --probes 10
# Interpretation: if NY probes RTT < Toronto probes RTT for .20, inbound IPv4 for Toronto lands in NY (confirms C/F).
```

### 9.4 Provider lookups (record timestamp and raw JSON)

```bash
for ip in 204.68.238.11 204.68.238.20 2602:f2be:1::3 2602:f2be:2::3; do
  date -u +%FT%TZ
  curl -s "https://ipinfo.io/$ip/json?token=$IPINFO_TOKEN"
  curl -s -u "$MM_ACCT:$MM_KEY" "https://geolite.info/geoip/v2.1/city/$ip" | jq '{city:.city.names.en,sub:.subdivisions[0].iso_code,country:.country.iso_code}'
  curl -s "https://api.ipapi.is/?q=$ip" | jq '.location'
done
curl -sI https://layer9i.com/geofeed.csv; curl -s https://layer9i.com/geofeed.csv | sha256sum   # compare with git 431ad9b
whois -h whois.arin.net "n 204.68.238.0" | grep -i -E 'geofeed|comment'
```

---

## 10. Recommended corrective actions (not implemented)

Ranked by expected effectiveness, with operational risk for each. **None of these were carried out.**

| Rank | Action | Expected effectiveness | Operational risk |
|---|---|---|---|
| 1 | **Collect the missing evidence first** (§9.1–9.4, §8). Confirm where the /24 is announced and where probes to .20 actually land before changing anything. | Turns the hypotheses into facts. Prevents a fifth geofeed change made blind. | None (read-only) |
| 2 | **Ask IPinfo to purge the v1 record and re-evaluate**, giving them the v1→v4 timeline (§4.2), the current file's SHA-256, and the explanation that the regional /29s sit inside one /24. | High, if D or E is part of the cause. Low effect if measurement (B) keeps overriding. | None |
| 3 | **Host an IPinfo probe server in AS403034 at the Toronto site** (IPinfo's own recommendation for ASNs where measurement is noisy). | Medium–high. Gives IPinfo a Toronto vantage point inside the network. | Low. A new outward-facing service; needs a firewall review. |
| 4 | **Freeze the geofeed.** Stop changing it, and list only blocks that are actually serving from the stated city. v4 already does this; keep it stable for several re-verification cycles (IPinfo says weekly). | Medium. Restores feed credibility (D). Doesn't fix measurement disagreement on its own. | None |
| 5 | **Make IPv4 measurement agree with the feed by giving each region its own announceable /24.** Since Vultr's IPv4 minimum is /24, the only clean equivalent of the IPv6 per-site /48 design is one /24 per region (a new allocation or lease), announced only from that region's instance. | **Highest for the underlying cause (C/F).** | Medium–high. New address space, RPKI/ROA, IRR, geofeed and NAT changes in every region. Needs a change window. |
| 6 | **If the /24 must stay shared:** announce it from *every* region and make sure each site answers inbound for its own /29 locally (Vultr-internal more-specifics, if Vultr supports them per §8.1 Q4). Avoid hairpinning through New York. | Medium. Anycast routing is uneven (Vultr says so), so measurement can still disagree for some probes. | Medium–high. Changes BGP in production. Risk of blackholing or asymmetric return. |
| 7 | **Align the addresses in use with the documented plan** (Toronto .19 vs .20) and with the backend `own_addresses`, or update the plan. | Low for geolocation. High for operational clarity and avoiding SNAT/BGP mismatches. | Low–medium. Changes NAT; affects live sessions. |
| 8 | Publish or verify the ARIN `Geofeed:` reference (RFC 9632) for the /24 and the /40, if not already done. | Low–medium (discovery only). The file was already accepted. | None |

---

## 11. Sources

**Repository and GitHub (primary evidence)**
- `Layer9i/atlas-site` git: `bac0c70`, `585ff1b`, `0072a5b`, `431ad9b` (`geofeed.csv`); `4b3c7e7` (#292); `cc5e1f6` (#85, Cloudflare caching).
- GitHub Pages deployment runs: [37531204367](https://github.com/Layer9i/atlas-site/actions/runs/37531204367) (v1), [37658877121](https://github.com/Layer9i/atlas-site/actions/runs/37658877121) (v2), [37661726465](https://github.com/Layer9i/atlas-site/actions/runs/37661726465) (v3), [39602620247](https://github.com/Layer9i/atlas-site/actions/runs/39602620247) (v4).

**Published documentation, via search-engine summaries (pages not fetched directly)**
- Vultr: [BYOIP advisory](https://docs.vultr.com/platform/customer-advisory/byoip.md) · [BGP product docs](https://docs.vultr.com/products/network/bgp) · [Configuring BGP on Vultr](https://docs.vultr.com/configuring-bgp-on-vultr) · [AS20473](https://docs.vultr.com/products/network/bgp/asn-information/as20473.md)
- IPinfo: [Trust-based vs evidence-based geolocation](https://community.ipinfo.io/t/trust-based-vs-evidence-based-geolocation/7208) · [Why verification matters](https://community.ipinfo.io/t/ip-geolocation-and-geofeeds-why-verification-matters/7216) · [Geofeed as a fallback](https://community.ipinfo.io/t/why-geofeed-works-as-a-fallback-not-a-foundation/7209) · [ProbeNet](https://www.ipinfo.io/probenet) · [NANOG 96: IP Geofeeds — Trust, Accuracy, and Abuse](https://prod.nanog.revsys.com/events/nanog-96/content/5678/) · [NANOG list, Feb 2026](https://seclists.org/nanog/2026/Feb/113)
- PeeringDB: [net 43823 (Layer9i, AS403034)](https://www.peeringdb.com/net/43823)
- Standards: RFC 8805 (geofeed format), RFC 9632 (finding and using geofeed data)

*Report generated 2026-10-21T14:30Z. All "not retrieved" and "not tested" cells were blocked by access limits, not by negative results.*
