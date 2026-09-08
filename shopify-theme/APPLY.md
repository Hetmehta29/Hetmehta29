# Bluprint — product card size availability + quick-add fix

Store: **bluprint.in** · Theme: **Copy of Stretch** (Stretch 2.0.1 by Maestrooo), theme id `160137576674`

**Nothing here has been applied to the live theme.** Duplicate the live theme first
(Online Store → Themes → ⋯ → Duplicate), apply to the copy, preview it, then publish.

---

## Part 1 — Size availability on the product card

Small read-only size row at the bottom of the product image. In-stock sizes read normally,
sold-out sizes are greyed out with a diagonal line through them. Nothing is clickable, so the
quick-add button and the card link keep their current behaviour and sit beside it.

### 1a. Create `snippets/product-card-size-availability.liquid`

New file — copy `snippets/product-card-size-availability.liquid` from this folder.

### 1b. Edit `snippets/product-card.liquid`

Find the end of the product media link and the start of the quick-buy block (~line 177):

```liquid
    </a>

    {%- if show_quick_buy and product.available -%}
```

Insert the render between them:

```liquid
    </a>

    {%- if settings.product_card_show_size_availability -%}
      {%- render 'product-card-size-availability', product: product -%}
    {%- endif -%}

    {%- if show_quick_buy and product.available -%}
```

That is the only change to this file. The theme's own `show_product_size_selector` block further
down is left untouched — it is switched off in your settings and stays off.

### 1c. Append the stylesheet

Paste the whole of `assets/product-card-size-availability.css` from this folder at the **end** of
`assets/theme.css`.

### 1d. Add the theme setting

In `config/settings_schema.json`, find the last entry of the **product card** panel:

```json
      {
        "type": "checkbox",
        "id": "product_card_show_product_size_selector",
        "label": "t:theme_settings.product_card.show_product_size_selector",
        "info": "t:theme_settings.product_card.show_product_size_selector_info",
        "default": true
      }
    ]
  },
```

Add a comma after it and append the new checkbox:

```json
      {
        "type": "checkbox",
        "id": "product_card_show_product_size_selector",
        "label": "t:theme_settings.product_card.show_product_size_selector",
        "info": "t:theme_settings.product_card.show_product_size_selector_info",
        "default": true
      },
      {
        "type": "checkbox",
        "id": "product_card_show_size_availability",
        "label": "Show size availability on card",
        "info": "Shows a small read-only row of sizes on the product image. Sold-out sizes are greyed out and crossed through. It does not add to cart.",
        "default": true
      }
    ]
  },
```

It then appears in **Theme settings → Product card → Quick buy** as a checkbox you can switch
off without touching code — useful for measuring the effect on drop-off.

### Why the sizes are read correctly

Availability is derived from `variant.available`, not from the option list, and each size value is
wrapped in pipes (`|M|2XL|`) before matching. Without the pipes, `XL` would match inside `2XL` and a
sold-out size would show as in stock. Checked against live data:

| Product | Sizes | Rendered |
|---|---|---|
| Pine Green Pure Cotton Shirt | S·M·L·XL·2XL | S, L, XL struck through — M, 2XL normal |
| Black Majenta Checkered Shirt | S·M·L·XL·2XL | S, M struck through — L, XL, 2XL normal |
| Cobalt blue washed vintage denim | 28–36 | all normal |

Every product in the catalogue has exactly one option named `Size`, inventory is tracked, and every
variant is `inventoryPolicy: DENY`, so `variant.available` is reliable.

---

## Part 2 — Quick-add opens with no size selector

### Cause

`snippets/product-quick-buy.liquid` passes an allowlist of blocks to render in the modal:

```liquid
allow_blocks: '@app,vendor,title,sku,badges,price,payment_terms,rating,separator,variant_picker,product_variations,line_item_property,inventory,buy_buttons,volume_pricing'
```

`snippets/product-info.liquid` filters against it with a plain substring test:

```liquid
if allow_blocks contains block.type
```

Your `templates/product.json` has **no `variant_picker` block**. Its size selection is a custom
section-local block, `size-picker` ("BP · Size picker"), which is not in the allowlist and is
therefore dropped. `buy_buttons` **is** in the allowlist, so it renders — which is exactly the
symptom: Add to cart and Buy now, no size selector.

### Why whitelisting `size-picker` alone is not enough

`QuickBuyModal.show()` in `assets/theme.js` fetches the product page and lifts only the template:

```js
const responseContent = await (await cachedFetch(this.getAttribute("product-url"))).text();
const tempDoc = new DOMParser().parseFromString(responseContent, "text/html");
const quickBuyContent = tempDoc.getElementById("quick-buy-content");
this.replaceChildren(quickBuyContent.content.cloneNode(true).firstElementChild);
```

Only markup inside `<template id="quick-buy-content">` travels, and `shouldAppendToBody` puts it on
`document.body`. The BP block's styles live in `{% style %}` blocks in `sections/main-product.liquid`
(some scoped to `#shopify-section-{{ section.id }}`), and its click handlers and the
"variant availability map + purchase guard" script are section scripts that never run on a
collection page. So the block would render unstyled and inert, and the guard that blocks sold-out
sizes would be missing.

### Option A — recommended, no code, ~5 minutes

Add the theme's native **Variant picker** block to the product template
(Customize → Product page → Add block). It is already in the allowlist, and its CSS and JS live in
`theme.css` / `theme.js`, which load on every page — so the modal gets a working, inventory-aware
picker immediately.

If you want the BP picker to stay the visible one on the product page, hide the native one there
only. The BP block's own description anticipates this: *"Keep or hide the theme's native variant
picker."*

```css
/* product page only — the quick-add modal is appended to <body>, outside this section */
#shopify-section-template--main .product-info__block-item[data-block-type="variant-picker"] {
  display: none;
}
```

Confirm the section id from the rendered page before using that selector.

### Option B — keep the BP picker in the modal

More work, and it is the only route if you want the modal to look identical to the product page:

1. Add `size-picker` to `allow_blocks` in `snippets/product-quick-buy.liquid`:
   ```liquid
   allow_blocks: '@app,vendor,title,sku,badges,price,payment_terms,rating,separator,variant_picker,size-picker,product_variations,line_item_property,inventory,buy_buttons,volume_pricing'
   ```
2. Move the `.bp-size*` rules out of `sections/main-product.liquid` into `assets/theme.css`, dropping
   the `#shopify-section-…` prefixes so they apply on collection pages.
3. Re-bind the size-picker click handlers after the modal injects its content, and port the purchase
   guard so sold-out sizes stay blocked inside the modal.

Do not ship step 1 on its own — it renders the block unstyled and without the sold-out guard, which
is worse than the current state.

### Note on GoKwik

Your theme settings have GoKwik with `goBuynowEnable` and `goEnable` on, intercepting Buy Now and
Checkout. Re-test the quick-add modal end to end after this change, since the modal's buy buttons go
through that interception.

---

## Also worth fixing

`assets/theme.css` has a malformed declaration in the theme's own (currently unused) size selector:

```css
.floating-size-selector__button.is-disabled {
  --button-text-color: var(--button-background-primary) / .5;   /* invalid — stray "/ .5" */
```

The `/ .5` makes the declaration invalid, so the sold-out colour silently falls back. It only bites
if you ever switch on `product_card_show_product_size_selector`. Correct form:

```css
  --button-text-color: var(--button-background-primary);
  opacity: 0.5;
```
