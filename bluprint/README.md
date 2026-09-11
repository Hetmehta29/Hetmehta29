# Bluprint PDP — hero gallery & zoom rebuild

Three files, dropped into the Shopify theme at these paths:

| File here | Paste into |
|---|---|
| `sections/main-product.liquid` | `sections/main-product.liquid` |
| `snippets/product-info.liquid` | `snippets/product-info.liquid` |
| `templates/product.json` | `templates/product.json` |

Everything outside the **BP · Hero gallery** block is untouched.

## What changed

**The main image is a true 4:5 window.** The frame used to carry an
`aspect-ratio` *and* a `max-height: 460px`. The height got clamped while the
width kept filling its column, so on a 412px phone a "4:5" frame actually
rendered 392×460 — near square. The width is now capped off the same ratio,
so the shape is exact at every screen size (measured 0.8000 from 320px to
1920px). Images `cover` the frame by default, so there are no grey bars
inside it.

**It slides sideways.** Native scroll-snap for touch, plus click-and-drag for
a mouse, one image per gesture. Progress lines and the thumbnail rail track
the position; a resize re-lands on the image you were on.

**The viewer is a full-screen phone gallery.** An opaque backdrop, the photo
edge to edge with a soft corner, a counter above it and one dock of round
controls — prev / close / next — in thumb reach at the bottom.

The overlay is re-homed onto `<body>` so a fixed layer is not trapped inside
the block's `zoom:`/overflow-clip ancestor. That move also carried it out of
`.bp-blk`, where the palette tokens are declared, so `var(--pg)` resolved to
nothing, the background fell back to transparent and the page showed straight
through the viewer — the header, the thumb rail and the sticky bar all visible
behind the photo, the close control drawn as a bare outline. The script now
copies the resolved tokens onto the element before it moves, and every `var()`
in the overlay's CSS carries a literal fallback.

**It opens the photo whole, then zooms anywhere:**

- pinch to zoom, or tap the picture to jump to 2.5× at the point you touched
- drag to pan — the image is moved by `transform`, so panning reaches every
  corner at any zoom, clamped so it never drifts off into empty space
- wheel / trackpad-pinch on desktop; `+` `-` `0` `←` `→` `Esc` on a keyboard
- at 1×: swipe sideways for the next photo, pull down or tap the surround to
  close

The gallery is also operable from a keyboard now — it previously could not be
opened without a pointer. The image on screen is the single tab stop (roving
tabindex, so twelve photos still cost one stop), `Enter` or `Space` opens it,
focus moves into the dialog and returns to the image on close.

**The thumbnail rail is small and shows full photos.** Fixed 56px cells in the
same shape as the main image, `contain` so nothing is cropped — a landscape
shot letterboxes inside its cell rather than being cut down. The rail scrolls
itself when there are more than fit, and never widens the page.

## New block settings

| Setting | Default | Note |
|---|---|---|
| Thumbnail width | 56px | 36–110 |
| Images to show | 12 | was hard-capped at 6 |
| Deepest zoom | 5× | 2–8 |
| Widest the gallery gets | 460px | was a *height* cap; height now follows the ratio |
| How the main image fills the frame | Fill the frame | thumbnails and the viewer always show the whole photo |
