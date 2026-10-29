# iherb-cli

Agent-native CLI for iHerb Taiwan (`tw.iherb.com`, TWD). Zero dependencies (Python 3.8+ stdlib).
Search, product, price checks, login and the cart. **No checkout.**

Portable: everything goes over plain HTTPS. These are the endpoints the web app uses (measured
2026-10-29):

| what | endpoint | needs |
|---|---|---|
| search | `GET tw.iherb.com/search?kw=&sr=&p=` (HTML, 48/page) | nothing |
| product / price | `POST graphql-catalog.app.iherb.com` `products(input:{ids})` | nothing |
| variants (specs) | `GET tw.iherb.com/pr/<slug>/<id>` (HTML) | Cloudflare: saved session, else Edge fallback |
| whoami | `GET catalog.app.iherb.com/catalog/currentUser` | login |
| cart | `checkout8.iherb.com/api/Carts/v2` (+ `v3/catalog/lineItems`, `v2/lineitem`, `v2/lineitems/products`) | login |

If Cloudflare challenges an anonymous read, the CLI retries it in a **background** tab of the
Edge/Chrome on CDP (`IHERB_CDP`, default `127.0.0.1:9222`). That tab is closed afterwards, and your
own tabs are never focused or touched.

## Login (no password, ever)

```
iherb login                      # = --from-browser: copies the session from Edge on CDP;
                                 #   if you are not signed in it opens the iHerb sign-in page in a
                                 #   background tab and waits (--wait 300) while you sign in there
iherb login --from-clipboard     # copy DevTools' Cookie request header first
iherb login --cookie-stdin       # paste or pipe it (raw tty, >4 KB is fine)
iherb login --keyring            # store it in the OS keyring instead of the file
iherb whoami | logout
```

Before saving, the session is checked over plain HTTP with `currentUser`. It is saved to
`~/.config/iherb/state.json` (0600, `IHERB_HOME` moves it), or to the keyring with `--keyring`
(`secret-tool`/`security`, chunked because `secret-tool` returns at most 8 KB). `IHERB_COOKIE`
overrides both. The Edge User-Agent is saved too, because Cloudflare's `cf_clearance` is bound to it.
`whoami` and the cart use only this saved session; they never borrow the browser. The session lasts
about 2 weeks (`expires` in `whoami`).

## Read

```
iherb doctor                                  # search, catalog, CDP, login, cart count
iherb search 魚油 [--limit 20] [--page 2] [--sort price|price-desc|bestselling|rating|reviews|discount|newest|relevance]
             [--min 200 --max 800] [--in-stock]
iherb product <id|url>      (alias: item)     # price, list price, promo code, stock, rating, flags,
                                              # free-shipping line, specs/variants with their ids
iherb price 61864 62118 https://tw.iherb.com/pr/.../111468   # price monitor, --json friendly
iherb raw /search?kw=zinc | raw https://checkout8.iherb.com/api/Carts/v2/cartcount   # GET only
```

On iHerb a variant is a separate product id. `product` lists every spec row (產品形式 / 劑量 /
包裝數量 ...) with the id of each option.

**Caveat (measured 2026-10-30):** some search results on every sort/page have no price at
all - discontinued listings and items iHerb blocks from purchase in this region/account
("無法購買" + a "查看類似商品" link instead of a price). Both are not purchasable, so the
CLI drops every result with `price: null` (the response reports how many under
`dropped_no_price`). This makes `--sort price`/`price-desc` reliable: the real prices left
on a sorted page were verified already in ascending/descending order once the priceless
items are removed, so no client-side re-sort is needed. `--min`/`--max`/`--limit`/`--page`
still apply only to the single remote search page fetched (`p=<page>`, 48 items/page before
filtering) - they are not a window over the whole catalog.

## Cart (login required)

```
iherb cart list
iherb cart add <id|url> [--qty N] [--spec '240 入' ...] --yes
iherb cart qty <line#|id> N --yes
iherb cart rm <line#|id|url> --yes
```

Without `--yes`, `add`/`qty`/`rm` only print exactly what they would do. After every write the CLI
reads the cart again. It reports the line it found there, and returns an error if the change did not
land. `--spec` resolves the variant through the product page and refuses sold-out or ambiguous
choices.

Every command takes `--json` and returns `{ok, data|error}`. Exit codes: 0 ok, 1 api, 2 usage,
3 auth, 4 net/blocked. `--via auto|http|browser` picks the transport for anonymous reads.

No checkout by design: no checkout, order or payment endpoint is ever called. This is an unofficial
client for a private API, so it may break at any time.
