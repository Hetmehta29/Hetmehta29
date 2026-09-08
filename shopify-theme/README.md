# Bluprint — size availability on product cards + quick-add fix

Applied to the **unpublished** theme `Size availability test (safe copy)`
(id `162640330978`) on bluprint.in. The live theme, `Copy of Stretch`, was not modified.
Every file here was uploaded and then re-downloaded and checksum-verified against these copies.

## What changed (5 files)

| File | Change |
|---|---|
| `snippets/product-card-size-availability.liquid` | **New.** Read-only size row for the product card. |
| `snippets/product-card.liquid` | Renders the above inside `product-card__figure`. |
| `snippets/product-quick-buy.liquid` | Renders the native variant picker in the quick-add popup. |
| `assets/bp-size-availability.css` | **New.** Styles for the size row. |
| `layout/theme.liquid` | Loads the new stylesheet. |

`assets/theme.css`, `config/settings_schema.json` and `templates/product.json` were left untouched.

## Size row

In-stock sizes read normally; sold-out sizes are greyed and struck with a diagonal line. The row is
inert (`pointer-events: none`) and sits at the bottom of the image with `padding-inline-end: 3rem`
so it never collides with the quick-add button pinned bottom-right — the two read side by side.

Availability comes from `variant.available`, matched against a pipe-delimited map (`|M|2XL|`). The
pipes matter: without them `XL` matches inside `2XL` and a sold-out size renders as in stock.
Verified against the live catalogue plus an adversarial "only 2XL in stock" case.

It renders unless `settings.product_card_show_size_availability` is explicitly `false`, so it works
without a schema change and still honours the toggle if one is added later.

## Quick-add popup

`product-info.liquid` filters section blocks with `if allow_blocks contains block.type`. The product
template has no `variant_picker` block — sizes come from a custom `size-picker` block, which is not
in the allowlist and gets dropped. Only `buy_buttons` survives (even the Kiwi app block is filtered),
which is exactly the reported symptom: Add to cart and Buy now, no size selector.

Allowing `size-picker` through would not work. `QuickBuyModal.show()` lifts only the
`#quick-buy-content` template and appends it to `document.body`, so that block's section-scoped CSS
and its scripts — including the sold-out purchase guard — never travel with it.

Instead the popup renders `snippets/variant-picker` directly, above `product-info` so it sits over
the buy buttons. Its CSS and JS live in `theme.css` / `theme.js`, which load on every page. This
needs no change to the product template, so the product page keeps the BP Size picker as its only
selector and no hiding CSS is required.

## Still to check on preview

- Sold-out sizes on a real mixed product (Pine Green Cotton Shirt: S, L, XL out; M, 2XL in)
- Mobile, where the size row and quick-add sit side by side
- Quick-add → pick a size → correct variant reaches the cart; sold-out sizes blocked
- **Buy Now**, which GoKwik intercepts (`goBuynowEnable` and `goEnable` are on)
