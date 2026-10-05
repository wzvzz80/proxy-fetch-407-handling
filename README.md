# node fetch proxy: how to route native fetch and node-fetch through a real proxy, handle 407s, and catch the silent bypass

Your code looks right. The request returns 200. And you're still hitting the target from your own IP.

That's the part of using a proxy with `fetch` in Node that trips people up: in most runtimes, a misconfigured proxy throws. In Node's built-in `fetch`, it usually doesn't. The request just goes direct, succeeds, and never tells you the proxy was skipped. `HTTP_PROXY` and `HTTPS_PROXY` sit in your `.env` file doing nothing at all.

So this isn't really an article about "add an agent." It's about which fetch implementation you're on, which of the three proxy mechanisms applies to it, and how to prove the traffic actually left through the proxy before you trust any of it.

## Two fetches, three mechanisms, and one nasty default

Node has had a global `fetch()` since v18 (experimental in 17.5). Separately, there's the `node-fetch` npm package, which has been doing the same job since long before that. Both are called "node fetch." They do not accept proxy configuration the same way, and mixing the instructions for one into the other is where most of the confusion starts.

| What you're calling | Under the hood | How it takes a proxy |
| --- | --- | --- |
| `fetch()` (Node 18+, global) | undici | `dispatcher` option, or `setGlobalDispatcher()` |
| `node-fetch` v2 / v3 | custom http/https agent | `agent` option (an `HttpsProxyAgent`) |
| Either, with `global-agent` | patches global agents | `GLOBAL_AGENT_HTTP_PROXY` env var, no code change |
| Recent Node releases | built-in proxy support | `--use-env-proxy` / `NODE_USE_ENV_PROXY` |

Two honest notes on that last row. Node has been rolling out native proxy support through an `--use-env-proxy` flag and a `NODE_USE_ENV_PROXY` variable, plus an `http.setGlobalProxyFromEnv()` API documented in the newer docs. It is the fix everyone has wanted for years, and it is also the reason any tutorial older than a few months may be wrong about "fetch can't do proxies." Check your actual `node -v` output before assuming either behavior.

The nasty default is undici's: it does not read `HTTP_PROXY`, `HTTPS_PROXY`, or `NO_PROXY`. Setting them changes nothing, and there is no warning. Same story for `node-fetch`, which also ignores those variables by design. Only Axios and Got among the common clients pick up proxy environment variables on their own.

## Native fetch with undici's ProxyAgent

For a single request, pass the dispatcher inline:

js
// npm i undici   (not needed on Node 18+, undici ships with it)
import { ProxyAgent } from 'undici';

const dispatcher = new ProxyAgent('http://USERNAME:PASSWORD@gw.dataimpulse.com:823');

const res = await fetch('https://httpbin.org/ip', { dispatcher });
console.log(await res.json()); // the egress IP, not yours


For a whole process — a scraper that makes thousands of calls and shouldn't thread an option through every function:

js
import { ProxyAgent, setGlobalDispatcher } from 'undici';

setGlobalDispatcher(new ProxyAgent('http://USERNAME:PASSWORD@gw.dataimpulse.com:823'));

// every fetch() in this process now exits through the proxy
const res = await fetch('https://httpbin.org/ip');


`dispatcher` is a Node-specific extension, not part of the standard Fetch API. If the same code has to run in a browser or in a Workers runtime, keep it behind a runtime check. Some non-Node runtimes ship their own `fetch` with their own proxy handling — Bun, for instance, accepts a `proxy` option, and it had a real bug where proxy credentials containing `+` or `/` in their base64 form got encoded with the wrong alphabet and the proxy closed the socket. If you're on a runtime other than Node, that's a category of bug worth knowing exists.

## node-fetch with https-proxy-agent

`node-fetch` takes an agent, so you build one. The npm package `https-proxy-agent` handles HTTPS targets through an HTTP(S) proxy, which is what you want most of the time.

js
// npm i node-fetch https-proxy-agent
import fetch from 'node-fetch';
import { HttpsProxyAgent } from 'https-proxy-agent';

const agent = new HttpsProxyAgent('http://USERNAME:PASSWORD@gw.dataimpulse.com:823');

const res = await fetch('https://httpbin.org/ip', { agent });
console.log(await res.json());


If you're still on CommonJS, note that `node-fetch` v3 is ESM-only. Either move your project to `"type": "module"`, or pin `node-fetch@2` and use `const fetch = require('node-fetch')`. This bites people mid-refactor more than any proxy setting does.

You can also skip the agent entirely with `global-agent`, which patches the global HTTP/HTTPS agents so your request code never changes:

js
import fetch from 'node-fetch';
import { bootstrap } from 'global-agent';
bootstrap();
// then run: GLOBAL_AGENT_HTTP_PROXY=http://user:pass@gw.dataimpulse.com:823 node app.js


It's convenient for getting something running. It's also a global monkey-patch, which is a real tradeoff inside a larger application — every outbound HTTP call in the process now routes through the proxy, including the ones you didn't want routed.

## SOCKS5, when you need it

SOCKS5 proxies need a different agent and a different URL scheme:

js
// npm i socks-proxy-agent
import { SocksProxyAgent } from 'socks-proxy-agent';

const agent = new SocksProxyAgent('socks5://USERNAME:PASSWORD@gw.dataimpulse.com:824');
const res = await fetch('https://httpbin.org/ip', { agent });


SOCKS5 tunnels at a lower level than an HTTP proxy, so it plays better with non-HTTP traffic and some legacy tooling. For ordinary scraping over HTTPS, `https-proxy-agent` is fine and one dependency lighter.

## Credentials, rotation, and sticky sessions in practice

Everything above is provider-neutral, so here's what it looks like once a real account is behind it. DataImpulse's gateway is `gw.dataimpulse.com`, port **823** for HTTP/HTTPS and **824** for SOCKS5. Credentials go in the URL as `user:pass@host:port` and get applied automatically by the agent.

Two connection modes, and the difference matters for how you write your loop:

- **Rotating.** Each new request exits from a fresh IP. Port 823 (HTTP/HTTPS) or 824 (SOCKS5). Nothing to manage, no proxy list to shuffle.
- **Sticky.** The IP is bound to a specific port so it stays put. Sticky ports run from 10000 to 20000, rotation intervals go from 1 to 120 minutes, and if you don't set an interval the default is 30 minutes.

Targeting rides in the username, which is a neat trick once you've seen it:

js
// __cr.us pins the exit country to the United States
const agent = new HttpsProxyAgent(
  'http://YOUR_LOGIN__cr.us:YOUR_PASSWORD@gw.dataimpulse.com:823'
);


Country targeting is included in the base price. City, state, ZIP, and ASN targeting are paid add-ons on residential plans, and on the standard residential tier traffic routed through those advanced filters is billed at roughly double the per-GB rate — worth knowing before you build a city-level crawler and then wonder why the dashboard shows the traffic burning twice as fast.

If your password contains characters like `@`, `:`, `/`, `#`, or `>`, run it through `encodeURIComponent()` before it goes into the proxy URL. That's the single most common cause of a 407 with credentials that look obviously correct.

👉 [Grab a DataImpulse account with $5 of pay-as-you-go traffic and test it against your own targets](https://bit.ly/dataimPulse)

## Verify the exit IP. Every time. Before you trust anything.

This is the step people skip, and it's the one that matters most, because a bypassed proxy produces no error. It looks exactly like a working setup, right up until the target starts rate-limiting your real IP.

js
const res = await fetch('https://api.ipify.org?format=json', { dispatcher });
console.log(await res.json()); // compare against your own IP


Run it twice — once with the dispatcher, once without — and compare. If the address doesn't change, your proxy isn't in the path, and you're debugging the wrong layer. `https://httpbin.org/ip` works just as well and shows the full header set.

## What the errors actually mean

Learn these six and proxy debugging stops being guesswork.

| Symptom | Real cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials missing, malformed, or not URL-encoded | `encodeURIComponent()` the password; check the username suffix isn't breaking parsing |
| `ECONNREFUSED` | Wrong host or port, proxy not reachable | Confirm port 823 (HTTP) vs 824 (SOCKS5) vs a sticky port in 10000–20000 |
| `ETIMEDOUT` | Proxy or target slow; sometimes the target quietly rate-limiting you | Pick a closer geo, raise the timeout, slow the request rate |
| `403` from the target | The proxy worked. The IP got flagged anyway | Switch proxy type, rotate per request, send realistic headers |
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `SELF_SIGNED_CERT_IN_CHAIN` | Something is re-signing TLS with its own CA | Load the real CA — see below |
| No error, no proxy | You passed `agent` to undici, or `dispatcher` to `node-fetch` | Use the option the specific client actually reads |

That TLS row has a wrong fix that circulates widely: `NODE_TLS_REJECT_UNAUTHORIZED=0`. It disables certificate verification for the whole process, which means any man-in-the-middle can swap a certificate and you'd never notice. Load the CA instead:

bash
NODE_EXTRA_CA_CERTS=/path/to/ca.crt node scraper.js


or pass it to the agent:

js
import fs from 'node:fs';
const ca = fs.readFileSync('/path/to/ca.crt');
const agent = new HttpsProxyAgent(proxyUrl, { ca });


Before you debug Node at all, test with curl on the same machine:

bash
curl -v -x "http://user:pass@gw.dataimpulse.com:823" https://api.ipify.org/


If curl fails, your code never had a chance.

## Which proxy type a Node scraper should actually pay for

Type doesn't matter much for local development and it matters enormously in production, because the price gap is real.

| Proxy type | DataImpulse entry price | Per GB | Where it fits |
| --- | --- | --- | --- |
| Datacenter | $5 for 10 GB | $0.50/GB | Public pages with no real anti-bot, internal APIs, high-volume jobs where speed beats stealth |
| Residential | $5 for 5 GB | $1/GB | E-commerce, SERPs, social — anything that inspects IP reputation |
| Mobile | $5 for 2.5 GB | $2/GB | The hardest targets, app and mobile-web data, where carrier NAT makes IPs expensive to block |
| Premium residential | $5 for 1 GB | $5/GB | High-trust residential traffic, all targeting included, dedicated account manager |

The rule of thumb that saves the most money: if datacenter IPs get through your target, keep using them. Paying mobile rates for datacenter-grade work is just an expensive habit.

## Full DataImpulse plan lineup and pricing

DataImpulse is pay-as-you-go per GB, with no subscription and traffic that doesn't expire. There's no free tier — the smallest commitment is the $5 entry pack on each product line. Volume tiers lower the per-GB rate; the figures below reflect published pricing, and since per-GB rates do shift, confirm the current number at checkout.

| Proxy type | Tier | Price | Per GB | Coverage | Buy |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | $5 / 5 GB | $1.00 | 195 countries, country targeting free | [Start with the $5 residential pack](https://bit.ly/dataimPulse) |
| Residential | Advanced | $800 / 1 TB | $0.80 | Same network, higher volume | [Check the Advanced residential tier](https://bit.ly/dataimPulse) |
| Datacenter | Intro | $5 / 10 GB | $0.50 | Cheapest tier, 99.9% uptime | [Try datacenter proxies from $0.50/GB](https://bit.ly/dataimPulse) |
| Datacenter | Volume | $50 / 100 GB | $0.50 | Randomized datacenter subnets | [Compare datacenter volume tiers](https://bit.ly/dataimPulse) |
| Datacenter | Volume | $450 / 1 TB | $0.45 | Same terms, better rate | [See the 1 TB datacenter rate](https://bit.ly/dataimPulse) |
| Datacenter | Custom | from $2,250 / 5 TB+ | Custom | Enterprise-style commit | [Request custom datacenter pricing](https://bit.ly/dataimPulse) |
| Mobile | Intro | $5 / 2.5 GB | $2.00 | 4G / 5G / LTE carrier IPs | [Add mobile proxies to your setup](https://bit.ly/dataimPulse) |
| Mobile | Volume | $50 / 25 GB | $2.00 | Same pool, larger commit | [See mobile volume options](https://bit.ly/dataimPulse) |
| Mobile | Volume | $1,600 / 1 TB | $1.60 | Volume discount at 1 TB+ | [Check the 1 TB mobile rate](https://bit.ly/dataimPulse) |
| Mobile | Custom | from $8,000 / 5 TB+ | Custom | Large-scale mobile work | [Talk to DataImpulse about mobile volume](https://bit.ly/dataimPulse) |
| Premium residential | Intro | $5 / 1 GB | $5.00 | All targeting included, no surcharge | [View premium residential pricing](https://bit.ly/dataimPulse) |
| Premium residential | Volume | $50 / 10 GB | $5.00 | High-speed pool, account manager | [Compare premium residential tiers](https://bit.ly/dataimPulse) |
| Premium residential | Custom | from $20,000 / 5 TB+ | Custom | Enterprise commit | [Request premium residential pricing](https://bit.ly/dataimPulse) |

What's included across all of it: HTTP/HTTPS and SOCKS5, rotating and sticky sessions, a pool of 90M+ ethically sourced IPs across 195 countries, country-level targeting at no extra cost, and no monthly minimum. DataImpulse publishes a 99.51% success rate and holds a 4.8/5 rating on G2 — both are the company's own numbers, so treat them as marketing until your own test confirms them.

## What the $5 entry pack is actually good for

Five gigabytes sounds like nothing. For the specific question of "does this work on my target," it's plenty — you're buying a measurement, not capacity.

Concretely: point your `HttpsProxyAgent` at the rotating gateway, run a few hundred requests against your real target with the headers you'd actually use, and log two things — how many came back with usable content, and how many GB it burned. Divide the second into the first. That's your cost per successful request, and it's the only number that lets you compare a $1/GB provider fairly against a $5/GB one. A cheap pool that gets blocked half the time costs more per usable record than a pricier pool that doesn't.

Traffic doesn't expire, so an inconclusive test is an inconvenience rather than wasted money. That's the part of pay-as-you-go that matters for the uneven workload most Node scrapers actually have.

## FAQ

**Does `node-fetch` read `HTTP_PROXY` or `HTTPS_PROXY`?**
No. Neither `node-fetch` nor Node's built-in `fetch` reads those variables by default. Build an agent explicitly — `https-proxy-agent` for node-fetch, undici's `ProxyAgent` for native fetch — or use `global-agent` to have them applied process-wide.

**Do I still need `node-fetch` on Node 18+?**
Usually not. Native `fetch` is fine for most scraping, and undici's `ProxyAgent` is well supported. Stick with `node-fetch` if you depend on its specific behaviors or you're maintaining code that already uses it.

**Why does my proxy work in curl but fail in Node?**
Almost always URL encoding of the credentials, or the wrong option name for the client. Node-specific special characters in a password need `encodeURIComponent()`, and passing `agent` to undici is a silent no-op.

**What's the difference between `agent` and `dispatcher`?**
`node-fetch` and Got use `agent`. undici and native Node `fetch` use `dispatcher`. Using the wrong one doesn't throw — the request just goes direct without the proxy.

**Rotating or sticky sessions for scraping?**
Rotating for breadth, sticky when state has to survive across requests — carts, multi-step flows, paginated sequences. On DataImpulse, rotating is port 823/824 and sticky uses ports 10000–20000 with a 1–120 minute rotation window.

**Is it worth paying for residential over datacenter?**
Only if datacenter IPs are getting blocked on your targets. Test with the cheap tier first; most public pages and internal APIs don't care.

Once the agent is right and you've confirmed the exit IP is someone else's, the rest of a Node scraping setup is ordinary engineering — concurrency, retries, parsing. The proxy stops being a mystery and becomes one line of config you never think about again.

👉 [Set up a DataImpulse account and put the exit-IP test on your own targets](https://bit.ly/dataimPulse)
