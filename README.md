# hardstack.com

Coming-soon landing page for HardStack — hardware-in-the-loop testing.

Live at **https://hardstack.com** (`www` 301s to the apex).

## Layout

```
public/          the site itself — edit this
  index.html     single page, styles inlined
  logo.svg       wordmark logo (PCB stackup with a plated through-hole)
  favicon.svg
  og.png         1200x630 social preview, rendered from the page
build.mjs        inlines public/ into dist/worker.js
wrangler.jsonc   Cloudflare Worker config (account: Alpha21)
```

The site is four small files, so `build.mjs` embeds them directly in the Worker
rather than using a separate asset store: one artifact to deploy and no cold
asset lookups. The Worker also handles the `www` redirect, ETags and caching.

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

`og.png` is a screenshot of the page itself at 1200x630, with the dot grid and
the travelling pulse hidden (both only add noise at card size and cost a lot of
bytes), then palette-reduced to 12 colours to stay around 15 KB:

```js
// in DevTools, with the page open at 1200x630
document.head.insertAdjacentHTML("beforeend",
  "<style>body{background-image:none!important}.loop .pulse{display:none!important}</style>");
```

```sh
magick og-raw.png -resize 1200x630^ -gravity center -extent 1200x630 \
  -colors 12 -strip PNG8:public/og.png
```

Keep it at 12 colours or above — below that, quantisation breaks the 1px board
outline and trace into dashes.

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
for board designators such as `HOST` and `DUT`.

The hero diagram is the product thesis drawn literally: a closed loop where a
host sends stimulus into a device under test and reads measurements back. The
travelling pulse is the page's only ambient motion, and it is disabled under
`prefers-reduced-motion`.

## Contact

info@hardstack.com
