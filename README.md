# hardstack.com

Marketing and ordering site for HardStack — remote, software-controlled
electronics testbenches, sold as a monthly subscription plus metered bench time,
alongside firmware, test-automation and bench-engineering services sold by the
engineer-day.

Live at **https://hardstack.com** (`www` 301s to the apex).

## Layout

```
public/               the site itself — edit this
  index.html          the page: platform, consulting, pricing, order form
  order-received.html post-order confirmation (303 target of /order)
  logo.svg            wordmark logo (PCB stackup with a plated through-hole)
  favicon.svg
  og.png              1200x630 social preview, rendered from the page
build.mjs             inlines public/ into dist/worker.js, plus /order handling
wrangler.jsonc        Cloudflare Worker config (account: Alpha21)
```

The site is a handful of small files, so `build.mjs` embeds them directly in the
Worker rather than using a separate asset store: one artifact to deploy and no
cold asset lookups. The Worker also handles the `www` redirect, ETags, caching
and the order form.

## The order form

`POST /order` is the only non-static route. The handler lives in the Worker
template in `build.mjs`. It validates the plan, name, company and email, drops
submissions that fill the hidden `fax` honeypot, then redirects 303 to
`/order-received`.

Every order is written to the Worker log as a single `ORDER {...}` line, so the
log is the backstop if forwarding is misconfigured:

```sh
npx wrangler tail --format pretty | grep ORDER
```

Set **`ORDER_WEBHOOK_URL`** to have each order POSTed as JSON somewhere useful —
an email relay, a Slack incoming webhook, a CRM endpoint:

```sh
npx wrangler secret put ORDER_WEBHOOK_URL
```

Until it is set, the Worker logs `ORDER_WEBHOOK_URL is not set` on every order
and the only record is the log line. **Set it before pointing anyone at the
form.**

The payload is:

```json
{
  "plan": "rack",
  "planLabel": "Rack - $1,200/month + $4/bench-hour",
  "name": "...", "email": "...", "company": "...",
  "benches": "3", "hardware": "...",
  "receivedAt": "2026-01-01T00:00:00.000Z", "country": "US"
}
```

## Prices

Everything orderable appears in the `<select>` in the order form and in the
`PLANS` map in `build.mjs`, keyed by the same `value`. An order naming a key
that is not in `PLANS` is rejected with a 400, so the two must be kept in step —
adding a service means editing both.

Prices are written out in four places and must agree, because the confirmation
email is generated from the label the Worker recorded:

1. `public/index.html` — the pricing cards (`.plan`) for the subscriptions
2. `public/index.html` — the rate table (`.bom`) for the services
3. `public/index.html` — the `<select>` in the order form, and the `From $299`
   line in the hero
4. `build.mjs` — the `PLANS` map, which is what gets logged and forwarded

```sh
# every <option> should have a PLANS entry
grep -o 'option value="[^"]*"' public/index.html
grep -A9 'const PLANS' build.mjs
```

## Develop

```sh
npm install
npm run dev          # wrangler dev on localhost
```

Or, for plain static preview without Wrangler:

```sh
cd public && python3 -m http.server 8787
```

## Deploy

```sh
npm run deploy       # builds, then wrangler deploy
```

Requires Wrangler to be authenticated against the **Alpha21** Cloudflare
account (`wrangler login`, or a `CLOUDFLARE_API_TOKEN` with Workers Scripts
edit permission).

## Regenerating og.png

`og.png` is a screenshot of the hero at exactly 1200x630, with the dot grid and
the travelling pulse hidden (both only add noise at card size and cost a lot of
bytes), then palette-reduced to stay under 30 KB:

```js
// in DevTools, with the page open at exactly 1200x630
document.head.insertAdjacentHTML("beforeend", `<style>
  body { background-image: none !important; height: 100vh; display: flex; flex-direction: column; overflow: hidden; }
  .loop .pulse { display: none !important; }
  .rail { position: static !important; border-bottom: none !important; flex: none; }
  .rail nav { display: none !important; }
  .rail .wrap { padding-block: 26px 0 !important; }
  .avail { display: none !important; }
  main, footer { display: none !important; }
  .hero { flex: 1; padding-block: 0 !important; display: flex; align-items: center; }
  .board { inset: 4px 26px 26px !important; }
</style>`);
```

```sh
magick og-raw.png -resize 1200x630^ -gravity center -extent 1200x630 \
  -dither None -colors 16 -strip PNG8:public/og.png
```

`-dither None` matters more than the colour count: with dithering on, the two
near-identical greens (`--substrate` and `--substrate-raised`) turn the YOU and
BENCH node fills into visible speckle. Keep it at 16 colours or above — below
that, quantisation breaks the 1px board outline and trace into dashes.

## Design notes

Palette and type are meant to read as test-bench hardware rather than generic
SaaS:

| Token | Value | Stands for |
| --- | --- | --- |
| `--substrate` | `#0A2226` | solder mask |
| `--copper` | `#C98B4B` | copper traces and pads |
| `--silkscreen` | `#E6EDE9` | silkscreen ink |
| `--signal` | `#A8E8F0` | the live signal on the loop |

Type is Archivo (wordmark and headline, on the width axis) with IBM Plex Mono
for board designators such as `BENCH` and `PROG`.

The hero diagram is the product thesis drawn literally: a closed loop where your
code and agents drive a bench in our lab and read measurements and video back.
The travelling pulse is the page's only ambient motion, and it is disabled under
`prefers-reduced-motion`.

The capability and pricing grids use fixed column counts rather than `auto-fit`.
`auto-fit` would let a fourth column appear at wide viewports and leave visible
empty cells in a grid whose gaps are drawn as hairlines.

## Trademark

The page is also the specimen of use supporting the HardStack trademark
application (Class 42). The mark is owned by **Alpha21, LLC**, which is the
applicant of record — the footer attribution has to name Alpha21, LLC and not
HardStack, or it contradicts the filing. For that it has to keep showing, together and on one
screen: the HardStack mark, a description of the services, and a direct way to
order them. Do not reintroduce "coming soon" wording, and do not remove the
order form or the pricing, without checking against the filing first.

The consulting section carries weight here out of proportion to its size:
engineering services can be rendered on the day the application is filed, which
is harder to argue for a subscription platform that is still being built.

## Contact

Orders: orders@hardstack.com · Everything else: info@hardstack.com
