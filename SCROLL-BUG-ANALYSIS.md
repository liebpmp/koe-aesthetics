# Scroll Bug Analysis — KÖ-Aesthetics Mobile

## Problem
First click on CTA → scrolls to #formular but then jumps back up (lands ABOVE the form).
Second click works correctly. Only on mobile (iOS Safari).

## Root Cause (confirmed by Villa Bella comparison)

### 1. CSS `scroll-behavior: smooth` + `scroll-padding-top` is buggy on iOS Safari
iOS Safari doesn't reliably honor `scroll-padding-top: 84px` when using CSS-native `scroll-behavior: smooth`. The browser overshoots or miscalculates the final position, especially when DOM changes occur during the scroll animation.

### 2. IntersectionObserver animations cause position instability during scroll
When smooth-scrolling from hero → form, dozens of elements along the path get their `visible` class added by IntersectionObserver. While `transform` and `opacity` don't cause reflow, iOS Safari's smooth scroll implementation can recalculate the target position when compositing layers change (new opacity transitions = new compositor layers).

### 3. `contact-info.fade-left` is directly above `#formular` on mobile
On mobile (single-column layout), the contact-info div (with `fade-left` = `opacity: 0; transform: translateX(-40px)`) is stacked directly above the form card containing `#formular`. When this element transitions to visible during the scroll, it can cause iOS Safari to adjust the scroll anchor point.

### 4. Why second click works
After the first scroll, all animations along the path are already triggered (elements have `visible` class). No more DOM/compositing changes occur during the second scroll → position stays stable.

## Villa Bella Reference (working)
URL: https://liebpmp.github.io/villa-bella-bruststraffung/gynaekomastie.html

Villa Bella uses:
- **JS `scrollTo` with explicit -80px offset** (not CSS scroll-behavior)
- **No scroll animations on mobile (≤768px)** — IntersectionObserver is wrapped in `if(window.innerWidth<=768) return;`
- **No `scroll-padding-top`** in CSS

## Fix Applied

1. **Removed `scroll-behavior: smooth` from CSS** — replaced with JS scrollTo
2. **Added JS smooth scroll handler** for all `a[href^="#"]` links:
   - `e.preventDefault()`
   - Close mobile nav if open
   - **Pre-reveal all animated elements in the target section** (add `visible` without transition)
   - Calculate `target.getBoundingClientRect().top + window.pageYOffset - 84`
   - `window.scrollTo({top, behavior: 'smooth'})`
3. **Removed redundant nav link toggle handlers** (scroll handler closes nav)

### Why pre-reveal works
By instantly revealing animated elements in the target section BEFORE starting the scroll:
- No compositing layer changes during scroll
- No opacity/transform transitions interfering with scroll position
- The layout is fully settled before scrollTo calculates the target
- First click behaves identically to second click

## Actual Root Cause (confirmed via Playwright test)

**The doctor image `hosseini_4.jpeg` (2117×1992px, lazy-loaded, no dimensions) caused a 308px layout shift during scroll.**

Before fix: Image starts at 0px height → loads during scroll → jumps to 308px → pushes `#formular` down by 308px → scroll lands 308px too high.

After fix: CSS `aspect-ratio: 2117/1992` + HTML `width="2117" height="1992"` → image reserves 308px height before loading → zero layout shift → scroll lands perfectly at 84px offset.

## Test Results (Playwright, 375×812 mobile viewport)

| Test | Before Fix | After Fix |
|------|-----------|-----------|
| Click 1 → formular top | 392px ❌ | 84px ✅ |
| Click 2 → formular top | 84px | 84px ✅ |
| Position difference | 308px | 0px ✅ |
| Page height change during scroll | +308px | 0px ✅ |
| Sticky CTA scroll | untested | 84px ✅ |
| Mobile nav CTA scroll | untested | 84px ✅ |

## Files Changed
- `lidstraffung.html` — CSS aspect-ratio on doctor image + JS scrollTo + pre-reveal
- `brustvergroesserung.html` — same fix
- `index.html` — copy of brustvergroesserung.html
- `SCROLL-BUG-ANALYSIS.md` — this file
