# Debug log

Your notes. One entry per bug you fixed, using the template below.

This file is read as carefully as your code. A correct fix you cannot
explain counts for little; a bug you could not fix but investigated
honestly still counts for something.

Delete the example before you submit.



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
