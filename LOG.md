# Debug log

### CC-02 — "Can't read anything in dark mode"

**Reproduced:** Started the server, opened localhost:3000 and switched to dark mode.
On every dish card the name and price were dark brown on a dark card, almost invisible.

**Cause:** `frontend/style.css:294` had `.dish-body { color: #2b2118 }` — a fixed
dark-brown colour. The name and price set no colour of their own, so they inherit it.
In dark mode `--ink` flips to light, but this fixed value never changes.

**Fix:** Changed `.dish-body`'s colour to `var(--ink)`, the theme text colour. It is
the one rule both texts inherit from, so both are fixed and follow the theme.

**Checked:** Hard-refreshed. Names and prices are readable in dark mode and light mode.

**Time:** ~15 minutes.



### CC-01 — "The search suggestions are behind everything"

**Reproduced:** Opened localhost:3000, went to the menu and started typing a dish name
(e.g. "sam"). The suggestion list showed up under the search box, but its lower rows sat
behind the category bar. Only the thin strip right under the input could be clicked;
clicking the rest did nothing.

**Cause:** `.suggest-box` had `z-index: 100`, which looks big enough, but its parent
`.search-wrap` had `z-index: 1`. A positioned element with a z-index creates its own
stacking context, so the dropdown's `100` was only ever compared with things inside
`.search-wrap`, never with the `.cat-tabs` bar next to it, which is `z-index: 40` inside the
same `.filters` box. The category bar painted on top and swallowed the clicks.

**Fix:** Raised `.search-wrap` to `z-index: 45` in `frontend/style.css:756`, so the whole
search area (and the dropdown inside it) stacks above the category bar's `40`. Changed one
value, nothing else.

**Checked:** Reloaded and typed again. The suggestions now sit over the category bar and
every row is clickable. Arrow keys and Escape still behave.

**Time:** ~20 minutes.



### CC-03 — "The menu is wider than my phone"

**Reproduced:** Opened the menu in Chrome at phone widths (about 500-620px) and measured the
page. `document.documentElement.scrollWidth` was 615px while the window itself was only 512px
wide, so the page scrolled sideways and the right-hand dish cards (with their "Add to Cart"
buttons) were cut off the edge. At 760px+ and at 480px or less there was no overflow, which is
why it only showed on certain phone widths.

**Cause:** Two rules inside the `@media (max-width: 760px)` block of `frontend/style.css`
were asking for more width than a card has:
  - `.dish-card img { width: auto; min-width: 100%; height: 170px; aspect-ratio: 3 / 2; }` —
    `height: 170px` together with `aspect-ratio: 3/2` makes `width: auto` resolve to 255px. On
    a phone the card is narrower than 255px, so the image forced the grid column to 255px and
    pushed the whole two-column grid past the screen edge.
  - `.name-btn { width: max-content; }` — this stopped long dish names from wrapping, so a
    column with a long name (e.g. "Paneer Butter Masala with Butter Naan") grew even wider.
  Added together, the grid was wider than the viewport.

**Fix:** In the same media block I removed `width: auto; min-width: 100%; aspect-ratio: 3 / 2`
from the image rule, leaving `height: 170px`. The base `.dish-card img` rule already sets
`width: 100%` and `object-fit: cover`, so the image now fills its card instead of stretching
it. I also deleted the `.name-btn { width: max-content; }` line so names wrap like they do on
desktop. CSS only, no markup or JS touched.

**Checked:** Re-measured with Chrome: `scrollWidth` now equals the window width at 500, 560,
620, 800 and 850px (was 615 vs 512). No sideways scroll and no cut-off buttons. Images still
fill the card. The picks strip is still a horizontal scroller (that is intentional).

**Time:** ~45 minutes, most of it setting up a headless Chrome measurement to see the real
card widths instead of guessing.



### CC-05 — "The filter bar scrolls away on my phone"

**Reproduced:** Emulated a phone in Chrome (390px wide), loaded the menu and scrolled down
1500px. The top site header stayed put, but the orange filter bar (search / sort / veg chips)
was gone: its `getBoundingClientRect().top` was -1388 instead of 0, i.e. it had scrolled off
the screen. It only happened on narrow screens (<=480px); on wider screens the bar stuck fine.

**Cause:** Inside the `@media (max-width: 480px)` block of `frontend/style.css` there was
`.view { overflow-x: hidden; }`. A `hidden` overflow makes that element a scroll container,
and `position: sticky` on `.filters` then sticks relative to that container's scrollport
instead of the page. Since the page scrolls (not the `.view` box), the filter bar never stayed
at the top. The site header was unaffected because it is not inside `.view`.

**Fix:** Changed the one line to `.view { overflow-x: clip; }`. `clip` also stops any
horizontal overflow (same visual result) but does not create a scroll container, so the
sticky filter bar works again. One value changed, no markup or JS touched.

**Checked:** At 360 and 390px, after scrolling 400, 1200 and 1500px, the filter bar now stays
at `top: 0` (was -1388) and `scrollWidth` still equals the window width (`overflow-x` still
clips, nothing new scrolls sideways). Header and everything else unchanged.

**Time:** ~25 minutes.



### CC-08 — "An old coupon still works"

**Reproduced:** Started the server and quoted an order with the long-expired coupon:
`POST /api/cart/quote { items: [{ dishId: 18, qty: 5 }], couponCode: "FRESHERS24" }`. It came back
with `couponDiscount: 30` and `couponError: null` — the discount was applied instead of refused.
`FRESHERS24` in `backend/data/coupons.json` expired on 2024-09-30, so it should never have counted.

**Cause:** `applyCoupon()` in `backend/logic/pricing.js` checked `usesLeft`, `minOrder`, veg-only
and the slot, but never looked at the coupon's `expiresAt` field at all. The `/api/coupons`
listing route (`routes/coupons.routes.js:24`) does filter out expired coupons with
`new Date(c.expiresAt) > now`, which is why the offer looked gone from the UI — but the pricing
path that actually decides the bill never applied that rule, so an expired code passed by the
browser still gave its discount. The expiry was only enforced in one of the two places.

**Fix:** Added an expiry check in `applyCoupon()` beside the existing `usesLeft` check:

```js
if (coupon.expiresAt && new Date(coupon.expiresAt) <= now) {
  return { valid: false, discount: 0, reason: 'This coupon has expired' };
}
```

This is the right place because every quote and every order is priced through `applyCoupon`
(`calculateBill` is the single entry point), so one check covers both the cart preview and the
real order. It also reuses the `now` the function already receives rather than reading the clock
again, so a quote and the order it produces agree on the time.

**Checked:** Re-ran the same quote: `couponDiscount: 0`, `couponError: "This coupon has expired"`,
and `couponCode: null`. A still-valid code (`BYTE10`) still returns its Rs. 33 discount, so live
coupons are unaffected. In `backend/smoke.js` the pre-existing check `expired coupon refused`
now passes (it failed before), along with `unknown coupon refused`, `percent coupon is capped`
and `only usable coupons listed`; no other check regressed.

**Time:** ~20 minutes, most of it tracing why the coupon disappeared from the UI but still
worked when sent directly.



### CC-06 — "I ordered more than they had"

**Reproduced:** Sent `PATCH /api/menu/10/stock { stock: 2 }` to leave only two plates, then
`POST /api/orders { items: [{ dishId: 10, qty: 5 }] }`. The order came back `201` with a token and
the stored stock went to **-3**. So the counter said two were left and it still sold five.
Note the smoke test's `qty over stock rejected` check did *not* catch this: it sends `qty: 99`,
which trips the separate "max 10 of one dish" rule first.

**Cause:** `validateLine()` in `backend/logic/validation.js` checked `dish.stock <= 0` (sold out),
`dish.disabled` and whether the dish was served now, but it never compared the **requested
quantity** against the stock that was left. `reserveStock()` then just subtracted whatever was
asked (`dish.stock -= item.qty`), so an over-order pushed stock negative instead of being refused.

**Fix:** Added one check to the same if/else chain in `validateLine`:
`else if (qty > Number(dish.stock)) errors.push(...only N left)`. This is the right place because
every order passes through `validateOrder()` before stock is touched, so the bad order is stopped
before `reserveStock` runs. I left the existing max-quantity rule alone; the two rules catch
different cases (a huge order vs. an order bigger than today's stock).

**Checked:** Stock 2 + qty 5 now returns `400` and stock stays 2. Ordering qty 2 with stock 2
still returns `201`. `git checkout -- backend/data/menu.json` after testing.

**Time:** ~25 minutes; the confusing bit was that `qty: 99` was already refused by a *different*
rule, so the bug looked fixed when it wasn't.



### CC-07 — "Cancelling makes it worse"

**Reproduced:** Set a dish to stock 10, placed an order of 1 (stock became 9), then
`PATCH /api/orders/:token/status { status: "cancelled" }`. Stock went **down** to 8 instead of
back up. Repeat a few times and the counter runs out while the kitchen is full, exactly as staff
described. The smoke test's `cancel puts stock back` check shows it too (`2 vs 4`).

**Cause:** `releaseStock()` in `backend/logic/validation.js` put stock back with
`stock: dish.stock - line.qty` — a minus sign. It was subtracting a second time instead of
returning the plates. Reservations use `reserveStock` (`stock -= qty`), so the release must be the
opposite (`stock += qty`).

**Fix:** Changed that one operator from `-` to `+`. This is the single function the cancel route
calls (`routes/orders.routes.js`), so the fix covers every cancellation path.

**Checked:** 10 -> order -> 9 -> cancel -> 10. The smoke test's `cancel puts stock back` now
passes. `git checkout -- backend/data/menu.json` after testing.

**Time:** ~15 minutes.



### CC-09 — "The menu shows more dishes than it should"

**Reproduced:** Loaded the menu and watched the network tab. The page asks for
`/api/menu?...&page=1&limit=5`, but the reply's `dishes` array held all 37 dishes, so the grid
dumped the lot at once instead of five with a "Load more" button.

**Cause:** `paginate()` in `backend/logic/search.js` did the paging maths correctly and even
computed the slice (`list.slice(start, start + perPage)`), but then returned `items: list` — the
**whole** array — instead of the slice. `menu.routes.js` sends `paged.items` to the browser, so
every "page" was really the full list.

**Fix:** Returned the computed slice (`items`) instead of `list`. The route, the maths and the
frontend already expected the correct field, so nothing else needed changing.

**Checked:** `page=1&limit=5` returns 5 dishes, `page=2` returns the next 5 (different first id),
`total=37`, `pages=8`, `hasMore=true`. The smoke checks `pages differ` and `page count is right`
pass.

**Time:** ~20 minutes.

### CC-10 — "Sorting by price is backwards"

**Reproduced:** `GET /api/menu?sort=price-asc&limit=48` returned the most expensive dish first
(first five: 138.5, 135, 133, 119.5, 114.5). `sort=price-desc` returned the cheapest first
(16.5, 21, ...). Both were reversed.

**Cause:** The `SORTERS` map in `backend/logic/search.js` had its two price comparators swapped:
`'price-asc': (a, b) => b.price - a.price` (descending) and
`'price-desc': (a, b) => a.price - b.price` (ascending).

**Fix:** Traded the two bodies so `price-asc` is `a.price - b.price` and `price-desc` is
`b.price - a.price`. Only those two were wrong; name/rating/prep/popular were fine, so I left them.

**Checked:** `price-asc` now starts 16.5, 21, 23, ...; `price-desc` starts 138.5, 135, 133, ....
The smoke check `sort by price works` passes.

**Time:** ~10 minutes.

### CC-04 — "The buttons don't work on my tablet"

**Reproduced:** By reading `frontend/style.css`. A media block at lines 1031-1046 only applies
between 761px and 900px (tablet widths). It adds two invisible `::after` overlays with
`content: ""` and a transparent background:
  - `.dish-card::after` covers the whole bottom 58px of each card at `z-index: 3` — exactly where
    the "Add to Cart" / quantity stepper sits.
  - `.img-wrap::after` covers the top-right 52px at `z-index: 1` — exactly where the favourite
    star sits.
They are transparent, so the buttons still look normal, but they are positioned boxes and are
hit-tested, so they swallow the taps. That is why Add to Cart and the star "do nothing" yet look
fine, and only on a tablet: below 761px and above 900px the block does not apply, which is why the
laptop and the phone work.

I could not click a real tablet in this environment, so this is a code-level reproduction rather
than a device one.

**Cause:** A transparent pseudo-element is still an event target. These overlays are decorative
(the blocked-card shading and its `transition: background`), but without `pointer-events: none`
they intercept the clicks meant for the buttons underneath.

**Fix:** Added `pointer-events: none;` to the shared `.dish-card::after, .img-wrap::after` rule so
the overlays still paint but no longer capture taps. That keeps the intended visual (blocked
shading) and lets clicks fall through to the buttons. Nothing else in the block depends on those
elements receiving events.

**Checked:** Reasoning only: with `pointer-events: none` the hit-test passes through to
`.add-btn` / `.fav-btn`, which are handled by the delegated click listener on the grid, so they
fire as they do at other widths. The block is unchanged otherwise, so wider and narrower viewports
are untouched. I would confirm on a 768px-wide device if one were available.

**Time:** ~30 minutes.

## Example — delete this

### CC-99 — "The cart total is wrong"

**Reproduced:** Added 2 dosas at Rs. 60 each. The cart showed
Rs. 119.99999 instead of Rs. 130. Happened every time, on any dish with
a price ending in .50.

**Cause:** The total was being added up with plain floating point and
never rounded, so 0.1 + 0.2 style errors showed up on screen. The
rounding helper existed but this one place was not using it.

**Fix:** Ran the total through the existing rounding helper instead of
adding a new one, so every price on screen goes through the same path.

**Checked:** Cart, checkout and the order screen all show Rs. 130 now.
Prices without decimals still show without a trailing .00.

**Time:** about 40 minutes, most of it working out that the cart and the
order screen round in different places.



## CC-0X — "<the complaint, in short>"

**Reproduced:**

**Cause:**

**Fix:**

**Checked:**

**Time:**



## Could not fix

For anything you investigated but did not solve. Say what you tried and
where you got to. This is worth marks — leaving it blank when you got
stuck is not.

### CC-0X — "<the complaint>"

**What I tried:**

**Where I got to:**

**What I would try next:**



## Extra credit

Anything not on the bug log: a problem you found yourself, a test you
wrote, or a fix you are unsure about. Same format, plus one line on how
you noticed it.
