# socks5 udp proxy: how to verify real UDP support, wire it into your stack, and what 9Proxy's plans cost

Here's the thing nobody tells you when you buy your first "SOCKS5 UDP proxy": SOCKS5 does not mean UDP. The protocol spec has a UDP command, plenty of providers advertise SOCKS5 in giant letters, and a large number of them will happily hand you a host:port that answers `CONNECT` and then does absolutely nothing when your client sends `UDP ASSOCIATE`. Your HTTP scraping works. Your game client, your WebRTC call, your QUIC-heavy HTTP/3 tooling, and your DNS-over-UDP queries all die with a timeout that looks like a network problem instead of a vendor problem.

That gap is what this article is about. Below: what the UDP part of SOCKS5 actually is, why it matters more in 2026 than it did five years ago, how to test it in under ten minutes before you spend money, and where 9Proxy's residential network fits into that picture (including the plans, the prices, and the one limitation worth knowing about).

## What "SOCKS5 UDP proxy" actually means

SOCKS5, defined in RFC 1928, supports three commands: `CONNECT` for TCP, `BIND` for inbound TCP, and `UDP ASSOCIATE` for UDP datagrams. That third command is the whole ballgame.

When a client sends `UDP ASSOCIATE`, the proxy server opens a UDP relay port and tells the client where to send datagrams. From then on, the client encapsulates each UDP packet with a small SOCKS5 header (the destination address and port) and fires it at the relay. The proxy strips the header, forwards the payload to the real destination, and sends replies back the same way.

Two consequences follow from that design, and both trip people up:

- **UDP support is a server-side feature, not a client-side one.** Your library can implement `UDP ASSOCIATE` perfectly. If the proxy on the other end returns a failure reply or silently drops the association, you get nothing.
- **DNS is part of the deal.** SOCKS5 lets the proxy resolve hostnames instead of your machine. With TCP that's routine. With UDP you need the resolver traffic to survive the relay too, which is why a proxy can pass a `curl` test and still fail a `dig` test.

SOCKS4 has no UDP command at all, and neither does HTTP/HTTPS proxying. So if a provider's protocol list says "HTTP/HTTPS and SOCKS5," the UDP question is entirely about whether their SOCKS5 endpoint implements `UDP ASSOCIATE`, and how well.

## Why UDP matters more than it used to

For years, UDP-over-proxy was a niche concern for gamers and torrent users. That's changed.

**QUIC and HTTP/3** run on UDP. A growing share of major sites serve traffic over QUIC by default, and clients that can't establish a QUIC connection fall back to TCP with an extra handshake. A proxy that only relays TCP quietly doubles your connection setup cost on those sites.

**WebRTC** for video calls, screen sharing, and browser-based voice is UDP-based. On a TCP-only proxy, call quality degrades or the connection simply fails to establish.

**DNS** defaults to UDP port 53. If you're doing geo-verification, ad verification, or any kind of regional content check, DNS resolution through the wrong path can leak or fail outright.

**Streaming, voice, and P2P** all lean on UDP because it doesn't wait for retransmits. Live video, VoIP, and BitTorrent all behave better when the proxy relays datagrams.

**Game clients** have used UDP forever, and modern game engines assume it.

If your workload is pure HTTP scraping with requests-style libraries, you can ignore all of this. If your workload involves any of the above, a TCP-only proxy is the wrong purchase regardless of how good the IP pool is.

## Four things to check before you buy any UDP-capable SOCKS5 proxy

Most plan pages won't give you straight answers on these. Ask support directly, or test.

1. **Does the SOCKS5 endpoint answer `UDP ASSOCIATE`?** Not "do you support UDP" as a marketing question. Ask whether the SOCKS5 listener implements the `UDP ASSOCIATE` command from RFC 1928. Vague answers are answers.
2. **Is the relay bidirectional?** Some implementations forward your outbound datagrams but drop the replies, which you'll only notice when a DNS lookup times out.
3. **Does the relay survive long sessions?** A UDP association bound to a residential IP that rotates or drops after three hours will cut your session mid-call. Check how the association handles IP churn.
4. **How does authentication work?** Username/password and IP whitelisting are the two common options. Datagram relays sometimes support only one of them, so confirm before you build a pipeline around it.

Anything you can't confirm from documentation, confirm with a test.

## Verifying UDP in about ten minutes

You don't need to write a load test. You need one test that fails loudly if UDP is dead. Three options, from easiest to most definitive:

**1. The TCP sanity check (control).** Any SOCKS5 endpoint should pass this. A `curl --socks5-hostname` request through the proxy should return a normal HTTP response. If this fails, the problem isn't UDP and you should stop here. Note that `curl`'s SOCKS5 support is TCP-only, so passing this test tells you nothing about UDP — it's your baseline, not your answer.

**2. The UDP association test (the real one).** Use a client library that actually implements UDP over SOCKS5 — PySocks exposes it through a `SOCK_DGRAM` socket, and SOCKS5-aware clients like tun2socks send `UDP ASSOCIATE` when routing traffic. Point a UDP socket at a public DNS resolver through the proxy and send a query. If you get an answer back, the relay is genuinely bidirectional. If you get a timeout while the TCP test passed, you've just proven the endpoint is TCP-only.

**3. The QUIC check.** If you have a client that can force HTTP/3 over a SOCKS5 proxy, try a QUIC-serving endpoint. This is a harder test because client support for UDP-over-SOCKS5 is patchy — some programs accept a SOCKS5 proxy and then quietly downgrade everything to TCP. If JSON-over-HTTP/3 fails but plain HTTPS works, find out whether the client or the proxy is at fault before blaming the vendor.

The one thing worth remembering: proxychains-style tools that only implement `CONNECT` won't tell you anything about UDP. A TUN-based redirector that speaks `UDP ASSOCIATE` will.

## Where 9Proxy fits

9Proxy is a residential proxy network with 20M+ IPs across 90+ locations, city/ZIP/ISP-level targeting, and two very different billing models. It's the kind of service people reach for when they want residential IPs without a monthly subscription.

On protocols, the company's own help center FAQ says it supports both HTTP and SOCKS5 and is "fully compatible with TCP and UDP connections." That's the vendor's position, and it's the claim you should test rather than trust, because it matters a lot for anything UDP-shaped. Worth flagging: at least one third-party review aggregator listing for 9Proxy states there's no UDP support at all, which is the opposite of the FAQ. That contradiction is a good reason to run the ten-minute test above on a small package before you commit to a five-figure IP order.

Two product structures, and the distinction matters for UDP work:

- **Residential Proxy by IPs** bills per IP with unlimited bandwidth on that IP. Unused IPs don't expire. You get the IP via the 9Proxy desktop app, which does local port forwarding and can add proxy authentication.
- **Residential Proxy by GB** bills by traffic, lets you generate unlimited endpoints from the dashboard, and supports either username/password auth or IP whitelisting. Traffic is valid for 180 days, or unlimited on enterprise tiers.

The desktop-app requirement on the IP-based product is the part I'd watch. Local port forwarding is exactly the layer where UDP can get dropped in ways that never show up in a browser test, so if UDP is central to your use case, validate it on the IP-based product specifically rather than assuming a working SOCKS5 TCP tunnel proves the rest.

9Proxy is also SOCKS5-compatible with the anti-detect browser crowd (Dolphin{anty}, AdsPower, and similar), which is the more common use case: multi-account work, ad verification, and geo-checked browsing rather than datagram relaying.

## 9Proxy plans and pricing

Prices below reflect the June 1, 2026 pricing update, which raised IP-based packages and bundles by roughly 20% while leaving GB-based packages untouched. All purchases are one-off balance top-ups, not subscriptions.

### IP-based residential packages (unlimited bandwidth per IP)

| Package | Price per IP | Total | Purchase |
| --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | [Get the 100-IP package](#) |
| 500 IPs | $0.144 | $72 | [Get the 500-IP package](#) |
| 1,000 IPs + 500 bonus | $0.084 | $126 | [Get the 1,500-IP package](#) |
| 2,500 IPs | $0.084 | $210 | [Get the 2,500-IP package](#) |
| 5,000 IPs | $0.072 | $360 | [Get the 5,000-IP package](#) |
| 15,000 IPs | $0.048 | $720 | [Get the 15,000-IP package](#) |
| 25,000 IPs | $0.035 | $863 | [Get the 25,000-IP package](#) |
| 50,000 IPs | $0.029 | $1,438 | [Get the 50,000-IP package](#) |
| Business 100,000 IPs | $0.023 | $2,300 | [Get the 100,000-IP business package](#) |
| Business 200,000 IPs | $0.021 | $4,140 | [Get the 200,000-IP business package](#) |
| Business 500,000 IPs | $0.018 | $8,625 | [Get the 500,000-IP business package](#) |

### GB-based residential packages (pay per gigabyte)

| Package | Price per GB | Total | Validity | Purchase |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [Get the 5 GB pack](#) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [Get the 55 GB pack](#) |
| 100 GB | $1.50 | $150 | 180 days | [Get the 100 GB pack](#) |
| 200 GB | $1.00 | $200 | 180 days | [Get the 200 GB pack](#) |
| 1,000 GB | $0.80 | $800 | 180 days | [Get the 1,000 GB pack](#) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [Get the 2,000 GB pack](#) |
| Enterprise 3,000 GB | $0.72 | $2,160 | No expiry | [Get the 3,000 GB enterprise pack](#) |
| Enterprise 6,000 GB | $0.70 | $4,200 | No expiry | [Get the 6,000 GB enterprise pack](#) |
| Enterprise 10,000 GB | $0.68 | $6,800 | No expiry | [Get the 10,000 GB enterprise pack](#) |

### Bundle packages (IPs + traffic)

| Bundle | Contents | Price | Purchase |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Get the Starter bundle](#) |
| Popular | 1,500 IPs + 50 GB | $180 | [Get the Popular bundle](#) |
| Pro | 5,000 IPs + 500 GB | $720 | [Get the Pro bundle](#) |

Bundled traffic is valid for 180 days.

The Starter bundle is the one worth doing arithmetic on. Bought separately, 100 IPs costs $24 and 5 GB costs $15, so $39 against $30 bundled. That's a real discount rather than a marketing rounding trick, and it's a cheap way to get both billing models in front of you at once while you run the UDP test.

## Which plan makes sense for UDP-heavy work

The two billing models reward different workloads, and UDP usage tends to be bursty in a way that makes the choice fairly clear.

**Long UDP sessions on a stable IP** — a game client held open for hours, a VoIP bridge, a persistent WebRTC connection — favour the IP-based model. Bandwidth is unlimited while the IP is active, so a voice call or a video stream doesn't burn a traffic counter you're watching nervously. The catch is IP lifetime: residential IPs naturally hold for a few hours to about 24 hours, so plan for reconnection logic.

**Short, high-rotation UDP requests** — DNS verification across many regions, checking a QUIC endpoint from fifty cities, quick geo-probes — favour the GB-based model. Each request uses kilobytes, you're rotating constantly, and paying per IP for that pattern is wasteful. The 5 GB pack at $15 is a reasonable testing budget: enough for thousands of small datagram tests, cheap enough that discovering the endpoint is TCP-only costs you a lunch rather than a month's budget.

If you want to test the UDP question seriously before committing, that's the order I'd do it in: 👉 [start with the smallest balance top-up](#), confirm the association with a datagram test, then size the package to whatever you learned.

## Limits worth knowing before you buy

- **IP lifetime is not permanent.** Residential IPs on the IP-based product last anywhere from a few hours to roughly 24 hours. Sticky-session work needs a reconnection path.
- **The IP-based product requires the desktop app.** You can't just point a script at host:port; traffic goes through local port forwarding on Windows.
- **The UDP claim is unverified in public documentation.** 9Proxy's FAQ says TCP and UDP are both fine, and at least one third-party listing says the opposite. Test first.
- **Reliability gets flagged in outside reviews.** Independent trackers and at least one user review reported a service outage in mid-2026 (a couple of weeks, then a second shorter one), attributed to an infrastructure migration. Vendor-side and third-party reviews are generally positive on IP quality and pricing, but if you're running production workloads, that's a genuine consideration.
- **HTTP and SOCKS5 only.** There's no SOCKS4 or dedicated datagram-only endpoint to fall back to.

## FAQ

**Is 9Proxy SOCKS5 UDP support confirmed?**
The company's help center FAQ states that HTTP and SOCKS5 are supported and fully compatible with TCP and UDP. A third-party listing claims the opposite, and 9Proxy's public documentation doesn't detail the UDP relay configuration. Treat it as a vendor claim and verify with a datagram test on a small package.

**Can I use a SOCKS5 proxy for HTTP/3?**
Architecturally yes, if the proxy implements `UDP ASSOCIATE` and your client sends UDP through SOCKS5. In practice many clients accept a SOCKS5 proxy and then route only TCP through it, so HTTP/3 silently falls back to TCP. Check the client before blaming the proxy.

**Will proxychains give me UDP over SOCKS5?**
No. Standard proxychains-style setups implement the TCP `CONNECT` path only. For UDP you need a TUN-based redirector or a client library that genuinely implements `UDP ASSOCIATE`.

**Do I need unlimited bandwidth for UDP?**
Rarely. Voice, DNS, and most datagram traffic are small. What UDP workloads do need is a proxy that keeps the association alive for the length of the session, which is why IP lifetime matters more than the traffic counter here.

**What's the cheapest way to test whether a proxy does UDP?**
The 5 GB pack at $15, or the $30 Starter bundle if you also want IPs in the same purchase. Either gives you enough headroom to run datagram tests against several regions and still have balance left if the answer is yes.
