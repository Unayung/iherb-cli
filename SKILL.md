---
name: iherb
description: Use when searching iHerb Taiwan (tw.iherb.com) products, checking TWD prices/promo codes/stock, or viewing/changing the user's iHerb cart (no checkout).
---

# iherb — iHerb Taiwan from a shell (no checkout)

`~/.local/bin/iherb` → `~/Projects/iherb-cli/iherb`. Stdlib Python over plain HTTPS. The Edge browser
on CDP 9222 is only used for the opt-in `login --from-browser` and as a fallback when Cloudflare walls an anonymous read.
Start with `iherb doctor`.

- `iherb search 魚油 --json [--sort price|price-desc|bestselling|rating|discount] [--min/--max TWD] [--page N]`
  returns items[] with id, title, brand, price, list_price, discount, rating, reviews, in_stock, url
  CAVEAT (measured 2026-10-30): discontinued listings and items iHerb blocks from purchase
  in this region/account ("無法購買") render with no price at all on every sort/page. Both
  are not purchasable, so the CLI drops every result with price=null (count reported in
  `dropped_no_price`). The remaining real prices on a `--sort price`/`price-desc` page were
  verified already in sorted order once those are removed - no client-side re-sort needed.
  `--min`/`--max`/`--limit`/`--page` only filter/window the single remote page fetched
  (48 items/page before filtering), not the whole catalog.
- `iherb product <id|url> --json` returns price, list_price, promo_code/promo_pct (e.g. OCT26DAILY -20%),
  stock, rating, flags, shipping (free over NT$790), and `variants[]` (each spec row → options with
  product `id`). On iHerb a variant is a different product id.
- `iherb price <id> <id>... --json` is for price monitoring (price, list_price, promo, in_stock, checked_at)
- Login: never ask for or type passwords/OTP, and never ask the user to paste cookies into chat. Tell
  the user: DevTools > Network > click any iherb.com request > copy the `Cookie:` request header to the
  clipboard, then run `iherb login` themselves (or pipe it in on stdin). No usable Cookie header = exit 2
  with those steps. `login --from-browser` is opt-in only (Edge on CDP 9222; opens the sign-in page in a
  background tab and waits for the USER). `whoami` checks the saved session over HTTP (email, name).
  Exit 3 = run `iherb login` again (sessions last about 2 weeks, and an expired `cf_clearance` also
  needs a fresh login).
- Cart (needs login): `cart list`; `cart add <id> [--qty N] [--spec '240 入']`; `cart qty <line> N`;
  `cart rm <line|id>`. Run first WITHOUT `--yes` to get the dry run, show it to the user, and add
  `--yes` only after they agree. The CLI re-reads the cart after each write and reports the verified line.
- Never try to check out: the CLI has no checkout or order command by design.

Pitfalls: anonymous product HTML (`/pr/...`) gets Cloudflare 403. Variants then come from the
background Edge tab, or are skipped with `--no-variants`. Repeated cookieless GraphQL calls sometimes
get a CF challenge; the CLI waits 3 s and retries once. The cart line price already includes the
public promo code (e.g. NT$207 → NT$166).

CDP setup: if `iherb doctor` shows `cdp` failing and a command needs Edge, ask the user to start it as in README "Browser on CDP 9222" (separate `--user-data-dir`, `--remote-debugging-port=9222`). Never launch a browser yourself.
