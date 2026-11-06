# REBUILD COMMAND: "Lighting Desires" hero (dark mode) with infinite staircase carousel

> Paste this whole document into the agent (Cursor / Claude Code / Lovable / Bolt / v0). It describes, with exact numbers, a full-screen hero: giant display typography on the left and, along the bottom, an **infinite staircase carousel** (cards grow from left to right, anchored to the bottom-left corner) driven by the **mouse wheel and by dragging**, with inertia. Do not interpret, do not "improve", do not swap libraries, fonts, colors, images or timings. Every value below was measured in a browser at 1440×900 unless stated otherwise.

**Brand name:** Lume (the name of the page: logo, `<title>`, logo `aria-label` and the h1 sentence use it, see sections 2 and 9)
**Project slug:** `lume`
**Asset base URL (Cloudflare R2):** `https://assets.layerweb.com.br/lume`

---

## 0. GLOBAL RULES (non-negotiable)

1. Deliver **one single `index.html`** (HTML + CSS + JS inline). No build step, no framework, no TypeScript, no Tailwind.
2. Allowed libraries (exactly this version, exactly this URL):
   - GSAP core: `https://cdnjs.cloudflare.com/ajax/libs/gsap/3.13.0/gsap.min.js`
   - **Nothing else.** DO NOT use SplitText, ScrollTrigger, Lenis, Swiper, Three.js or React. Splitting text into characters is done by hand (section 5.2). The carousel is a custom `requestAnimationFrame` engine (section 6), **not** a library slider.
3. Fonts:
   - **Mango Grotesque Bold (700)**, self-hosted: `https://assets.layerweb.com.br/lume/fonts/MangoGrotesque-Bold.woff2` (**status: hosted, answers 200, CORS rule active**, see section 7). Used for the display title, the h1 and the `<sub>`.
   - **Barlow 400** (Google Fonts): nav, card tags, MENU button.
   - Display fallback (only if Mango fails): **Antonio 700** (Google Fonts), with the mandatory `no-mango` rules of section 3.2.
   - `<link href="https://fonts.googleapis.com/css2?family=Barlow:wght@400&family=Antonio:wght@700&display=swap" rel="stylesheet">`
4. Assets: **Path A, hosted on Cloudflare R2.** Every image and the font are fetched from the asset base URL above. The exact URLs are listed in section 7. **DO NOT** replace, resize (change the aspect ratio), re-host or substitute any image, and **DO NOT** hotlink any third-party file or use stock-photo placeholders (picsum, unsplash...).
5. Language of the page: **English, everywhere** (visible text, `alt`, `aria-label`, `<title>`, `<html lang="en">` and code comments), exactly as in section 9. This document is in English too.
6. If a value is not specified here, use the value of the closest item mentioned and mark it with a `/* ASSUMED */` comment. Never eyeball a value.

### 0.1 Tokens

```css
:root{
  --display:"Mango Grotesque","Antonio","Arial Narrow",sans-serif;
  --fat-size:14.375rem;   /* display title size; becomes 7.75rem at <=991px (section 10.2) */
  --fb:1;                 /* fallback multiplier: 1 with Mango, .72 with Antonio (section 3.2) */
  --ease-out:cubic-bezier(.33,1,.68,1);   /* image hover, card tag */
}
/* DARK THEME. Colors: background #000000, ink #FFFFFF, tag dot #FFFFFF. Nothing else. */
```

### 0.2 Base unit (the hero is fluid, it is NOT a scaled canvas)

```css
html{font-size:1vw}                                   /* 992–1749 px: 1rem = 1% of the width */
@media (min-width:1750px){html{font-size:17.5px}}
@media (max-width:991px){html{font-size:16px}}
```

Every `rem` value below derives from this. At 1440 px wide, **1rem = 14.4px**. DO NOT use `transform: scale(k)` on a fixed canvas. DO NOT replace rem with px.

### 0.3 Reset / body

```css
@font-face{font-family:"Mango Grotesque";src:url("https://assets.layerweb.com.br/lume/fonts/MangoGrotesque-Bold.woff2") format("woff2");font-weight:700;font-style:normal;font-display:swap}
*,*::before,*::after{margin:0;padding:0;box-sizing:border-box}
html{scrollbar-width:none}
html,body{width:100%;height:100%;-webkit-font-smoothing:antialiased}
body{background:#000;color:#fff;font-family:"Barlow",sans-serif;font-size:1rem;font-weight:400;line-height:1.5;overflow:hidden}
body::-webkit-scrollbar{display:none}
a{color:inherit;text-decoration:none}
```

The page is **the hero only**: `body` has no vertical scroll. The mouse wheel moves the carousel, not the page.

---

## 1. LAYER ARCHITECTURE

### 1.1 DOM order (exactly this)

```
<body>
  .loader                     position:fixed; inset:0; z-index:999; background:#000; flex-centered
  header.nav                  position:fixed; inset:0 0 auto 0; z-index:99; mix-blend-mode:difference; pointer-events:none
    nav.nav-c
      .nav-r   → a.logo-nav, .link-w(Talents), .link-w(Projects)
      .nav-l   → .link-w(Services), .link-w(Content), .link-w(Contact), a.menu-open
  main
    section.hero[data-horizontal-scroll]     position:relative; width:100%; height:100vh; display:flex; overflow:hidden
      .c                                      position:relative; z-index:1; flex column; padding:4rem 1.5rem 1.5rem
        .copy                                 position:relative; z-index:90  (text ALWAYS above the cards)
          .fat            "Lighting"
          .fat.snd        "Desires"
          .space-1        (padding-top:1rem, empty)
          h1.h1-home      sentence + <br> + <sub>
        .track-w                              position:absolute; inset:auto 0 0 0; width:100%; height:100%; flex; align-items:flex-end
          .track[role=list]                   position:relative; height:100%; display:flex; pointer-events:none
            .card × 16 (generated by JS)      position:absolute; left:0; bottom:0
```

### 1.2 Stacking

| Layer | z-index | Note |
|---|---|---|
| `.loader` | 999 | disappears after the entrance (section 8) |
| `.nav` | 99 | `mix-blend-mode:difference`, **white** text/icons → white on the black background, inverted (dark) over bright cards |
| `.copy` | 90 | inside the stacking context of `.c` (z 1) |
| `.card` | `round(scale × 10)` | dynamic: 0…16 at 1440 px (max 21 above 1440 px). Always < 90 |
| `.media` (inside the card) | 20 | covers the `.tag`, which is born behind the image |

---

## 1.5 ABSOLUTE COORDINATE TABLE (viewport 1440×900, final state of the entrance, carousel at `current = 0`)

Measured in the browser and confirmed on the rebuild (difference ≤ 1 px).

| Element | x | y | w | h | z |
|---|---|---|---|---|---|
| `.logo-nav` | 22 | 14 | 45 | 38 | 99 |
| link "TALENTS" (text) | 170 | 28 | ~44 | 11 | 99 |
| link "CONTACT" (right edge of the text) | — | 28 | — | 11 | 99 (ends at x = 1418) |
| `.fat` "LIGHTING" (line box) | 22 | 58 | 445 | 185 | 90 |
| 1st `.cp` character ("L", mask incl. its `.3em` bottom padding) | 22 | 70 | 48 | 228 | — |
| `.fat.snd` "DESIRES" | 22 | 214 | 445 | 185 | 90 |
| `h1.h1-home` | 22 | 413 | 346 | ~72 | 90 |
| `sub` | 22 | 471 | 171 | 21 | — |
| card 0 (scale 0.28675) | 0 | 786 | 124 | 114 | 3 |
| card 1 (0.5185) | 112 | 695 | 224 | 205 | 5 |
| card 2 (0.75025) | 324 | 603 | 324 | 297 | 8 |
| card 3 (0.982) | 636 | 511 | 424 | 389 | 10 |
| card 4 (1.21375) | 1049 | 419 | 524 | 481 | 12 |
| card 5 (1.4455) | 1561 | 328 | 624 | 572 | 14 (off screen) |
| card 6+ (1.6) | 1944+ | 266 | 691 | 634 | 16 (off screen) |

The base of **every** card: `bottom = 900` (flush with the bottom edge of the viewport). `translateX` of card *i* = `i × 216 − current`.

---

## 2. NAV (PIXEL CONTRACT)

```css
.nav{position:fixed;inset:0 0 auto 0;z-index:99;min-height:3.5rem;display:flex;align-items:center;justify-content:space-around;mix-blend-mode:difference;pointer-events:none}
.nav-c{width:100%;display:flex;justify-content:space-between;align-items:center;padding:1rem 1.5rem 0;color:#fff;text-transform:uppercase}
.nav-r{display:flex;align-items:center;justify-content:flex-start;gap:6vw}
.nav-l{display:flex;align-items:center;justify-content:flex-end;gap:6vw}
.logo-nav{pointer-events:auto;width:3.125rem;display:flex;align-items:center;justify-content:center}
.logo-nav svg{width:100%;height:auto;display:block}
.link-w{display:flex;align-items:center;gap:.5rem;pointer-events:auto}
.dot-link{width:10px;height:10px;border-radius:50%;background:#fff;flex-shrink:0;opacity:0;transform:scale(.8);
  transition:opacity .6s cubic-bezier(.34,1.56,.64,1),transform .6s cubic-bezier(.34,1.56,.64,1)}
.link-w:hover .dot-link{opacity:1;transform:scale(1)}
.l{font-size:.75rem;line-height:1;text-transform:uppercase}
.menu-open{display:none;pointer-events:auto;font-size:1rem;line-height:1.5}
@media (max-width:991px){.link-w{display:none}.menu-open{display:block}}
@media (max-width:479px){.nav-c{padding-left:3vw;padding-right:3vw}}
```

Structure of each link: `<div class="link-w"><span class="dot-link"></span><a class="l">Text</a></div>`. The dot sits **to the left** of the text and takes up space even when invisible (10px + 0.5rem gap), which is why the "TALENTS" text starts at x = 170, not 153.

**Logo (PIXEL CONTRACT, brand "Lume").** Icon-only logo: a minimalist flame ("lume" means light/flame). Inline SVG, 50×42 viewBox, rendered **45×38px** at 1440 px (`.logo-nav` width 3.125rem, height auto) at x 22, y 14. Two shapes only, one color (#FFFFFF): (1) a flame outline, a teardrop with a pointed top, stroke 2.5, round line join, no fill; (2) a filled circle of radius 4.5 inside it, the core of the light. The brand name is carried by the `aria-label`, not by visible text.

```html
<svg viewBox="0 0 50 42" fill="none"><path d="M25 2C25 2 39 14 39 25A14 14 0 0 1 11 25C11 14 25 2 25 2Z" stroke="#fff" stroke-width="2.5" stroke-linejoin="round"/><circle cx="25" cy="27" r="4.5" fill="#fff"/></svg>
```

**DO NOT (logo):** redraw or restyle the icon; add a gradient, fill, glow or shadow to the flame; change the stroke width or the circle radius; add visible text next to it; use any color other than #FFFFFF; use a raster image.

**DO NOT:** give the nav a background; add a shadow; use `position:sticky`; use black text (it must be white + `difference`); underline the links; center the groups (they are `space-between`); show "MENU" on desktop.

---

## 3. DISPLAY TYPOGRAPHY "LIGHTING / DESIRES" (PIXEL CONTRACT, fragile point)

```css
.copy{position:relative;z-index:90;align-self:flex-start;user-select:none;-webkit-user-select:none}
.fat{font-family:var(--display);font-weight:700;font-size:calc(var(--fat-size)*var(--fb));line-height:.8;text-transform:uppercase;white-space:nowrap;
     display:flex;flex-wrap:nowrap;align-items:flex-start;height:.8937em;padding-top:.058em}
.fat.snd{margin-top:-2rem}
.cp{display:block;flex:0 0 auto;overflow:hidden;line-height:.8;padding-bottom:.3em}   /* per-character mask; the padding-bottom extends the mask downwards WITHOUT moving the letter, so the glyph base is never clipped, whatever the font */
.cc{display:block;position:relative}                               /* the character that rises */
.space-1{padding-top:1rem}
.sr-only{position:absolute;width:1px;height:1px;margin:-1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap;border:0}
```

Resulting values at 1440 px: font-size **207px**, line-height **165.6px**, line box **185px** (`.8937em`), each letter's mask starts **12px** below the top (`.058em`), overlap between lines **−28.8px**. Measured ink width: "LIGHTING" ≈ 445px, "DESIRES" ≈ 397px. Color: #FFFFFF (inherited from body).

**Why explicit height/padding:** the masks would otherwise be `inline-block` and the line height (185px) would depend on the font metrics. Here it is locked in `em` so the result is identical in every browser and scales together with the font. DO NOT go back to `inline-block` and DO NOT remove `height`/`padding-top`.

**DO NOT:** add letter-spacing to the display title `.fat` (it is `normal`; the .04em letter-spacing belongs only to the h1/sub, section 3.1); use `text-wrap`; break the word; use a weight other than 700; use another condensed font instead of Mango when it loads; center it; put both words in one block with a `<br>`; use accented words in the display title (the `.cp` mask clips accents above the 0.8em line height, which is why "LIGHTING / DESIRES" has no accents).

### 3.1 H1 and sub

```css
.h1-home{font-family:var(--display);font-weight:700;font-size:calc(2rem*var(--fb));line-height:.85;letter-spacing:.04em;text-transform:uppercase;max-width:24rem;text-align:left;opacity:0}
@media (max-width:767px){.h1-home{letter-spacing:.025em}}   /* on phones the column is 352px wide: .025em keeps the sentence on 2 lines */
.h1-home sub{font-size:75%;line-height:0;vertical-align:baseline;position:relative;bottom:-.25em}
```

At 1440 px: h1 28.8px / line-height 24.48px / **letter-spacing .04em (1.15px)** / max width 345.6px (24rem; wide enough to absorb the letter-spacing and keep the 2-line break: "LUME - AI STUDIO IN SÃO PAULO," / "PRODUCING CULTURE FOR LUXURY BRANDS."). The sentence length is tuned so the break falls right after "PAULO," at 1440, 1093 and 375 px (a longer sentence such as "production studio" or "shaping" wraps to 3 lines): DO NOT reword it. Sub 21.6px. There is a `<br>` between the h1 text and the sub. The sub spaces are `&nbsp;` exactly as in section 9.

### 3.2 Fallback when Mango Grotesque does not load (mandatory)

If the `.woff2` fails (404, offline, file not uploaded yet) the browser uses **Antonio**, which is **22% taller** (cap height 178px vs 145px at 207px) and **51% wider** ("LIGHTING" 673px vs 445px). Untreated, the title becomes huge and the base of the letters is clipped by the mask. Rule: after the fonts settle, if Mango is not `loaded`, add the class `no-mango` to `<html>` **before** the loader leaves.

```js
const mangoOk = [...document.fonts].some(f => f.family.replace(/"/g, "") === "Mango Grotesque" && f.status === "loaded");
if (!mangoOk) document.documentElement.classList.add("no-mango");
```

```css
html.no-mango{--fb:.72}                                   /* title and h1 at 72% of the size */
html.no-mango .fat{height:.95em;padding-top:0;line-height:.95}
html.no-mango .cp{line-height:.95;padding-bottom:.15em}
html.no-mango .cc{top:-.1em}                               /* lifts the letter so its base fits inside the mask */
html.no-mango .fat.snd{margin-top:-.08em}
html.no-mango .h1-home{line-height:1.05}
```

With this, "LIGHTING" is ≈ 485px wide at 1440 px (vs 445px with Mango), with no clipping and no overlap with the h1. With Mango loaded, `no-mango` is never applied.

---

## 4. CAROUSEL CARD (PIXEL CONTRACT, fragile point)

### 4.1 Markup (generated by JS)

```html
<div class="card" role="listitem">
  <div class="tag"><i></i><div>Project 01</div></div>
  <a href="#" class="media" draggable="false">
    <img src="https://assets.layerweb.com.br/lume/slides/slide-01.jpg" alt="..." draggable="false" style="object-position:50% 35%" onerror="this.classList.add('broken')">
  </a>
</div>
```

### 4.2 CSS

```css
.track-w{position:absolute;inset:auto 0 0 0;width:100%;height:100%;display:flex;justify-content:flex-start;align-items:flex-end;user-select:none;-webkit-user-select:none}
.track{position:relative;height:100%;display:flex;pointer-events:none}
.card{position:absolute;left:0;bottom:0;width:30rem;height:27.5rem;pointer-events:auto;transform-origin:bottom left;
      will-change:transform,scale,opacity;scale:0;opacity:0;--px:0%}
.tag{position:absolute;top:-5%;left:0;display:inline-flex;align-items:center;gap:.5rem;font-size:.75rem;line-height:1.5;text-transform:uppercase;white-space:nowrap;
     transform-origin:bottom left;translate:0 100%;opacity:0;transition:translate .4s var(--ease-out),opacity .4s var(--ease-out)}
.tag i{width:10px;height:10px;border-radius:50%;background:#fff;display:block;flex-shrink:0;transform:scale(.8)}
.card:hover .tag{translate:0 0;opacity:1}
.media{position:absolute;inset:0;z-index:20;overflow:hidden;display:block;user-select:none;-webkit-user-drag:none}
.media .shape,.media img{position:absolute;top:0;left:0;width:110%;height:100%;max-width:none;object-fit:cover;display:block;pointer-events:none;
     will-change:transform;transform:translate(var(--px));transition:transform .6s var(--ease-out)}
.card:hover .media .shape,.card:hover .media img{transform:translate(var(--px)) scale(1.03)}
.media img.broken{visibility:hidden}
```

Base card size (scale 1): **30rem × 27.5rem = 432 × 396px** at 1440 px (12:11 ratio). The inner media is **110%** of the card width to allow the horizontal parallax.

Points models usually get wrong:
- The card scale is applied with the **individual `scale` property** (inline, via JS) and the position with `transform: translateX()`. DO NOT merge both into a single `transform`; DO NOT use `width/height` to scale.
- `transform-origin: bottom left` on card and tag. DO NOT use `center`.
- The tag is born **behind** the image (`.media` has z-index 20, the tag has none): on hover it rises from `translate: 0 100%` to `0 0` and appears above the card. The tag receives an inline `scale = 1/cardScale` so its size stays constant (12px text at 1440).
- DO NOT add `border-radius`, shadow, border, visual gap or a fixed caption to the cards. Corners are square.
- DO NOT put `overflow:hidden` on `.card` (the tag sits outside it, at `top:-5%`).

---

## 5. TEXT: SPLITTING INTO CHARACTERS

### 5.1 Starting HTML

```html
<div class="fat" data-split>Lighting</div>
<div class="fat snd" data-split>Desires</div>
```

### 5.2 Splitting (reference code; use exactly this logic)

```js
document.querySelectorAll("[data-split]").forEach(line => {
  const txt = line.textContent;
  line.textContent = "";
  const sr = document.createElement("span"); sr.className = "sr-only"; sr.textContent = txt; line.appendChild(sr);
  [...txt].forEach(ch => {
    const p = document.createElement("span"); p.className = "cp";
    const c = document.createElement("span"); c.className = "cc"; c.setAttribute("aria-hidden", "true"); c.textContent = ch;
    p.appendChild(c); line.appendChild(p);
  });
});
const chars = [...document.querySelectorAll(".cc")];
gsap.set(chars, {yPercent: reduce ? 10 : 150});
```

The final state of each `.cc` is **`yPercent: 10`** (not 0): the letters stop 10% lower and the mask clips the overflow. Keep it.

---

## 6. INFINITE CAROUSEL ENGINE

### 6.1 Parameters per viewport width

| Width | minScale | maxScale | lerp | Mode |
|---|---|---|---|---|
| ≥ 1441 px | 0.055 | **2.1** | 0.06 | JS engine |
| 992–1440 px | 0.055 | **1.6** | 0.06 | JS engine |
| 768–991 px | 0.35 | 1.4 | 0.06 | JS engine |
| ≤ 767 px | — | — | — | native horizontal scroll strip (section 6.6) |

Constants: `gapRatio = 0.5` → step between cards = `base × 0.5` (216px at 1440). `total = step × N` (N = 16 → 3456px). Lerp while dragging: **0.1**.

### 6.2 Formulas (per frame, for each card *i*)

```
V      = width of section.hero
B      = i × step − current
cand   = [B];  if B > V: cand += B − total;  if B < −base: cand += B + total
X      = candidate closest to V/2
t      = clamp((X + base/2) / V, 0, 1)
s      = minScale + t × (maxScale − minScale)
px     = −15 × (1 − (s − minScale)/(maxScale − minScale))     → --px in %  (parallax: −15% at the smallest, 0% at the largest)
during the intro: s ×= q ; opacity = q      (q = easeOutCubic of the intro progress)
card.style.transform = translateX(X px); card.style.scale = s; card.style.opacity = q;
card.style.zIndex = round(s × 10); tag.style.scale = 1/s; card.style.setProperty("--px", px + "%")
after the loop: if current > total → current −= total, target −= total (and dragStart −= total while dragging); if current < 0 → += total
```

### 6.3 Entrance (automatic intro)

`current` and `target` go from **600 → 0** in **1500 ms**, `easeOutCubic` (`1 − (1 − p)^3`); cards are born with scale × q and opacity q. During the intro the wheel is ignored (the loop rewrites `target`). A `mousedown` ends the intro immediately (`introDone = true; current = target`).

### 6.4 User input

- **Wheel:** listener on `section.hero`, `{passive:false}`, `preventDefault()`. `delta = |deltaX| > |deltaY| ? deltaX : deltaY`; `target −= delta × 0.5`. Wheel down → cards move **right** (they grow). Do not normalize `deltaMode`.
- **Drag (mouse):** `mousedown` on the hero → `dragging = true; startX = pageX; dragStart = current`. `mousemove` on `document`: `dx = startX − pageX`; if `|dx| > 5` → `dragged = true` and `preventDefault()`; `target = dragStart + dx`. `mouseup` on `document` → `dragging = false`. On every `<a>` inside the hero: `mousedown` → `preventDefault()`; `click` → `preventDefault()` and, if it was dragged, `stopPropagation()`; `dragstart` → `preventDefault()`.
- **DO NOT** add autoplay, arrows, pagination dots, snap, Lenis or page scroll. The carousel stays still until there is wheel/drag input.

### 6.5 Reference code (use this, only adapting names)

```js
const params = () => {
  const w = innerWidth;
  return w <= 767 ? null : w <= 991 ? {min:.35, max:1.4, lerp:.06} : w <= 1440 ? {min:.055, max:1.6, lerp:.06} : {min:.055, max:2.1, lerp:.06};
};
let P = params();
const st = {target:600, current:600, dragging:false, dragged:false, startX:0, dragStart:0, base:0, gap:.5, total:0};
const INTRO_FROM = 600, INTRO_TO = 0, INTRO_MS = 1500;
let introDone = reduce, t0 = 0, raf = 0;
const easeOutCubic = x => 1 - Math.pow(1 - x, 3);

const measure = () => {
  const prev = items[0].style.scale; items[0].style.scale = "1";
  st.base = items[0].getBoundingClientRect().width;
  items[0].style.scale = prev || "1";
  st.total = st.base * st.gap * items.length;
};

const render = () => {
  const V = wrapper.getBoundingClientRect().width;
  const step = st.base * st.gap;
  let q = 1;
  if (!introDone) q = easeOutCubic(Math.min((performance.now() - t0) / INTRO_MS, 1));
  items.forEach((el, i) => {
    const B = i * step - st.current;
    const cand = [B];
    if (B > V) cand.push(B - st.total);
    if (B < -st.base) cand.push(B + st.total);
    let X = cand[0], best = Math.abs(X - V / 2);
    cand.forEach(n => { const d = Math.abs(n - V / 2); if (d < best) { X = n; best = d; } });
    let t = (X + st.base / 2) / V; t = Math.max(0, Math.min(1, t));
    let s = P.min + t * (P.max - P.min);
    const px = -15 * (1 - (s - P.min) / (P.max - P.min));
    s *= q;
    const tag = el.querySelector(".tag");
    if (s > 0) tag.style.scale = 1 / s;
    el.style.transform = `translateX(${X}px)`;
    el.style.scale = s;
    el.style.opacity = q;
    el.style.zIndex = Math.round(s * 10);
    el.style.setProperty("--px", px + "%");
  });
  if (st.current > st.total) { st.current -= st.total; st.target -= st.total; if (st.dragging) st.dragStart -= st.total; }
  else if (st.current < 0) { st.current += st.total; st.target += st.total; if (st.dragging) st.dragStart += st.total; }
};

const tick = () => {
  if (innerWidth !== lastW) syncSize();
  if (!P) { raf = requestAnimationFrame(tick); return; }
  if (introDone) {
    const k = reduce ? 1 : (st.dragging ? .1 : P.lerp);
    st.current += (st.target - st.current) * k;
  } else {
    const p = Math.min((performance.now() - t0) / INTRO_MS, 1);
    const v = INTRO_FROM + (INTRO_TO - INTRO_FROM) * easeOutCubic(p);
    st.current = st.target = v;
    if (p >= 1) { introDone = true; st.current = st.target = INTRO_TO; }
  }
  render();
  raf = requestAnimationFrame(tick);
};
```

`wrapper` = `section.hero`; `items` = the 16 original `.card` elements (no clones); `reduce` = `matchMedia("(prefers-reduced-motion: reduce)").matches`.

### 6.6 Phone (≤ 767 px): native strip with clone loop

No JS engine. `.track` becomes a strip with native horizontal scroll. On init: clone the 16 cards twice (one set before, reversed by `insertBefore` from the last to the first, and one set after), compute `W = Σ (width + margin-left + gap)` over the 16 originals, set `scrollLeft = W`. On the `scroll` event (passive, with a 50 ms lock): if `scrollLeft ≥ 2W − 50` → `scrollLeft = W + (scrollLeft − 2W)`; if `scrollLeft ≤ 50` → `scrollLeft = W + scrollLeft`.

```css
@media (max-width:767px){
  .c{padding-left:3vw;padding-right:3vw}
  .track{position:relative;display:flex;gap:.5rem;height:auto;pointer-events:auto;overflow-x:scroll;overflow-y:hidden;scrollbar-width:none;
         -webkit-overflow-scrolling:touch;width:100%;align-items:flex-end}
  .track::-webkit-scrollbar{display:none}
  .card{position:relative!important;left:auto!important;bottom:auto!important;flex:0 0 18rem;width:18rem;height:55vw;margin-left:.5rem;
        scale:1!important;opacity:1!important;transform:none!important;z-index:1!important;display:flex;flex-direction:column}
  .tag{position:relative;top:0;translate:0 0!important;opacity:1!important;scale:1!important;flex:0 0 auto}
  .media{position:relative;inset:auto;flex:1 1 auto}
  .media .shape,.media img{width:100%;transform:none!important;transition:none}
  .card:hover .media .shape,.card:hover .media img{transform:none}
}
@media (max-width:479px){
  .fat.snd{margin-top:-4vw}
  .card{height:80vw;margin-left:.25rem}
}
```

On phones the tag is **always visible**, above the image, with no hover and no parallax. The `!important` flags are mandatory: they stop inline styles left by the desktop engine (after resizing the window) from breaking the strip.

### 6.7 Resizing

When the window width changes, the carousel **must recompute** `P` (parameters) and `st.base`/`st.total` (through `measure()`). After resizing, positions and scales must be **identical to a fresh load at that width**. `addEventListener("resize")` alone is not enough: in some browsers the event fires before `rem` updates and the carousel keeps the measurement of the previous width (the "distorted carousel" bug). Use **three redundant triggers** calling the same function:

```js
let started = false, lastW = innerWidth;
const syncSize = () => {
  lastW = innerWidth;
  P = params();
  if (!started) return;
  if (P) { startDesktop(); measure(); } else { setupPhone(); }   // setupPhone runs only once (flag)
};
addEventListener("resize", syncSize);
if ("ResizeObserver" in window) new ResizeObserver(syncSize).observe(wrapper);
// and at the start of every tick():  if (innerWidth !== lastW) syncSize();
```

`startDesktop` is idempotent (`ticking` flag). The `tick` keeps being scheduled with `P = null` without rendering (avoids a `TypeError`).

---

## 7. ASSETS AND SLIDES (Path A, Cloudflare R2)

Asset base: `https://assets.layerweb.com.br/lume`

| File | Exact URL | Type | Size | Status |
|---|---|---|---|---|
| slide-01.jpg | `https://assets.layerweb.com.br/lume/slides/slide-01.jpg` | image/jpeg, 736×920 | 46,012 B | hosted, 200 |
| slide-02.jpg | `https://assets.layerweb.com.br/lume/slides/slide-02.jpg` | image/jpeg, 720×1280 | 92,439 B | hosted, 200 |
| slide-03.webp | `https://assets.layerweb.com.br/lume/slides/slide-03.webp` | image/webp, 1125×2000 | 94,842 B | hosted, 200 |
| slide-04.jpg | `https://assets.layerweb.com.br/lume/slides/slide-04.jpg` | image/jpeg, 736×1103 | 57,011 B | hosted, 200 |
| MangoGrotesque-Bold.woff2 | `https://assets.layerweb.com.br/lume/fonts/MangoGrotesque-Bold.woff2` | font/woff2 | 20,568 B | hosted, 200, answers `Access-Control-Allow-Origin: *` (bucket CORS rule active). If the font is ever unreachable, the page runs on the `no-mango` fallback, section 3.2 |

Notes:
- The font is loaded cross-origin by `@font-face`, so the bucket must answer with `Access-Control-Allow-Origin`. Required bucket CORS policy: `[{"AllowedOrigins":["*"],"AllowedMethods":["GET","HEAD"],"AllowedHeaders":["*"],"MaxAgeSeconds":3600}]`. Test: `curl -sI -H "Origin: https://example.com" <font URL>` must show `Access-Control-Allow-Origin`. Images do not need CORS.
- The 4 images are distributed over the 16 cards in sequence: card *i* uses image `i mod 4` (1, 2, 3, 4, 1, 2, ...). The 4 files never change their aspect ratio; the cards crop them with `object-fit:cover`.

```js
const SLIDES = [   // 16 cards; "Project NN" is the tag text; c1/c2 are used only when USE_IMAGES = false
  {tag:"Project 01", c1:"#8A8A8A", c2:"#3A3A3A", src:""},
  {tag:"Project 02", c1:"#5E5E5E", c2:"#262626", src:""},
  {tag:"Project 03", c1:"#A3A3A3", c2:"#4D4D4D", src:""},
  {tag:"Project 04", c1:"#6B6B6B", c2:"#2E2E2E", src:""},
  {tag:"Project 05", c1:"#B5B5B5", c2:"#5A5A5A", src:""},
  {tag:"Project 06", c1:"#7A7A7A", c2:"#333333", src:""},
  {tag:"Project 07", c1:"#9A9A9A", c2:"#444444", src:""},
  {tag:"Project 08", c1:"#575757", c2:"#222222", src:""},
  {tag:"Project 09", c1:"#ABABAB", c2:"#525252", src:""},
  {tag:"Project 10", c1:"#707070", c2:"#303030", src:""},
  {tag:"Project 11", c1:"#929292", c2:"#404040", src:""},
  {tag:"Project 12", c1:"#636363", c2:"#292929", src:""},
  {tag:"Project 13", c1:"#B0B0B0", c2:"#575757", src:""},
  {tag:"Project 14", c1:"#7F7F7F", c2:"#363636", src:""},
  {tag:"Project 15", c1:"#9F9F9F", c2:"#484848", src:""},
  {tag:"Project 16", c1:"#686868", c2:"#2B2B2B", src:""}
];
const USE_IMAGES = true;
const IMGS = [   // pos = vertical object-position, keeps the face / subject visible in the almost-square card
  {src:"https://assets.layerweb.com.br/lume/slides/slide-01.jpg",  pos:"50% 35%", alt:"Man with round glasses and a bun, in profile and silhouette against an orange sky"},
  {src:"https://assets.layerweb.com.br/lume/slides/slide-02.jpg",  pos:"50% 45%", alt:"Face half human, half liquid gold metal, floating among clouds"},
  {src:"https://assets.layerweb.com.br/lume/slides/slide-03.webp", pos:"50% 38%", alt:"Angel statue with hands together looking up at the light, in dark mist"},
  {src:"https://assets.layerweb.com.br/lume/slides/slide-04.jpg",  pos:"50% 35%", alt:"Man with platinum hair and futuristic metal sunglasses, wearing a black jacket"}
];
if (USE_IMAGES) SLIDES.forEach((s, i) => { const m = IMGS[i % IMGS.length]; s.src = m.src; s.pos = m.pos; s.alt = m.alt; });
const SHUFFLE = false;   // the order is fixed so the result is deterministic
```

Card media rule: when `src` is set, render `<img src alt draggable="false" style="object-position:${pos}" onerror="this.classList.add('broken')">` inside `.media`; otherwise render `<div class="shape" style="background:linear-gradient(160deg,c1 0%,c2 100%)" role="img" aria-label="tag">`. Same box, same `object-fit:cover`, no layout shift.

Rules:
- Minimum number of cards: **12** (below ~9 there are gaps in the loop at 1440 px). DO NOT go below it.
- DO NOT replace the 4 images with shapes, solid colors or numbered frames while the files answer 200.

---

## 8. ENTRANCE TIMELINE

t = 0 is when the fonts have settled, with a **3000 ms** ceiling (`Promise.race`). **Each** `document.fonts.load` needs a `.catch`: if a font fails (404, offline, file not uploaded yet) the promise rejects and, without the `.catch`, the loader **never leaves**.

```js
const safeLoad = f => document.fonts.load(f).catch(() => null);
const fontsReady = Promise.race([
  Promise.all([safeLoad('700 1em "Mango Grotesque"'), safeLoad('400 1em "Barlow"')]).then(() => document.fonts.ready).catch(() => null),
  new Promise(r => setTimeout(r, 3000))
]);
```

| t (s) | Element | From → To | Duration | Easing |
|---|---|---|---|---|
| 0 | `.loader` (5rem white logo, centered, black background) | holds | 0.5 | — |
| 0.5 | `.loader` opacity | 1 → 0, then `display:none` | 0.6 | `power2.inOut` |
| 1.1 | `.cc` of "LIGHTING" and, **at the same time**, `.cc` of "DESIRES" (one tween per line) | yPercent 150 → 10 | 1.2 each, stagger 0.05 within the line | `expo.out` |
| 1.1 | carousel `current` (section 6.3) | 600 → 0, scale × q, opacity q | 1.5 | easeOutCubic |
| 1.4 | `h1.h1-home` opacity | 0 → 1 | 1.0 | `power3.inOut` |

Implementation: `gsap.to(loader,{opacity:0,duration:.6,delay:.5,ease:"power2.inOut",onComplete:()=>{loader.style.display="none";start();}})`, where `start()` fires the following animations at the same instant (h1 with `delay:.3`). Each word has **its own tween** with stagger: the "L" of LIGHTING and the "D" of DESIRES rise together, then "I"/"E", and so on.

```js
document.querySelectorAll("[data-split]").forEach(line => {
  gsap.to(line.querySelectorAll(".cc"), {yPercent:10, duration:1.2, ease:"expo.out", stagger:.05});
});
gsap.to(".h1-home", {opacity:1, duration:1, ease:"power3.inOut", delay:.3});
```

Loader logo: the same SVG as section 2 (`stroke="#fff"`, `fill="#fff"`), 5rem wide.

**DO NOT:** fade the letters (it is only a masked displacement); run a single stagger across both words; animate the nav; animate the `sub` separately from the h1; delay the carousel relative to the letters.

---

## 9. LITERAL TEXTS (English)

| Where | Text (exactly like this) |
|---|---|
| `<title>` | `Lume \| AI Production Studio` |
| Display line 1 | `Lighting` (renders LIGHTING) |
| Display line 2 | `Desires` (renders DESIRES) |
| h1 | `Lume - AI studio in São Paulo, producing culture for luxury brands.` |
| sub | `PHOTO &amp; FILM.&nbsp;AI.&nbsp;3D. CGI.&nbsp;` |
| Nav | `Talents` · `Projects` · `Services` · `Content` · `Contact` |
| Mobile button | `MENU` |
| Card tags | `Project 01` … `Project 16` |
| Logo aria-label | `Lume, home` |
| Nav aria-label | `Main` |

`<html lang="en">`. Mango Grotesque has the glyphs Ã, Ç, É, Ê, Õ, Ú (verified); the h1 uses "Ã" in "SÃO". Do not replace accents.

---

## 10. RESPONSIVE (explicit architectures)

### 10.1 Desktop ≥ 992 px
Everything from sections 2–8. `1rem = 1vw` (up to 1749 px) / 17.5px (≥ 1750). Hero with `height:100vh`. From 1441 px up, `maxScale` jumps to 2.1 (keep it).

**Proportional scaling (core rule):** because `1rem = 1vw`, **everything** (title, h1, nav, card base and staircase) shrinks/grows in the same proportion as the window width, without wrapping or distortion. Measured example: the "LIGHTING" title takes ~30.9% of the window width at any width between 992 and 1749 px (445px at 1440, 338px at 1093, 307px at 992). The card staircase keeps the same shape: the scale step between neighboring cards is constant (0.2318 at 992–1440 px), so the width difference between cards is always `0.2318 × 30rem` (76px at 1093, 100px at 1440).

### 10.2 Tablet 768–991 px
`1rem = 16px`. Hero `height:100svh; padding-bottom:.5rem`. `.fat` 7.75rem (124px), `.fat.snd{margin-top:-1vw}`. Nav links hidden, "MENU" visible (16px, Barlow 400, uppercase). JS engine with min 0.35 / max 1.4; card base 480 × 440px. Everything else as on desktop.

```css
@media (max-width:991px){
  .hero{height:100svh;padding-bottom:.5rem}
  :root{--fat-size:7.75rem}      /* the size goes in the variable; DO NOT write .fat{font-size:7.75rem}, it would break the fallback (--fb) */
  .fat.snd{margin-top:-1vw}
}
```

### 10.3 Phone ≤ 767 px
Section 6.6. Reference values at 375 × 812: logo 50 × 42 at (11, 16); "MENU" at (321, 25); `.fat` 124px starting at (11, 64); cards **288 × 300**, first at x = 4, second at x = 304, top at y = 504, bottom at y = 804 (8px above the edge). Between 480 and 767 px: card height 55vw, `margin-left:.5rem`, `.fat.snd{margin-top:-1vw}`.

---

## 11. ACCESSIBILITY

- `prefers-reduced-motion: reduce`: no loader (it disappears at once), letters already at `yPercent:10`, h1 at opacity 1 with no animation, carousel intro skipped (`introDone = true`, `current = 0`), `lerp = 1` (no inertia), no transitions on tag/image/nav dot, no hover zoom (`transform: translate(var(--px))`).
  ```css
  @media (prefers-reduced-motion:reduce){
    .tag,.media .shape,.media img,.dot-link{transition:none}
    .card:hover .media .shape,.card:hover .media img{transform:translate(var(--px))}
  }
  ```
- Split words: `.sr-only` with the whole word + `aria-hidden="true"` on every letter.
- Images: descriptive English `alt` (section 7). Shapes (only when `USE_IMAGES = false`): `role="img"` + `aria-label` with the tag name.
- Loader with `aria-hidden="true"`. Nav with `aria-label="Main"`.
- Broken image: class `.broken` (`visibility:hidden`), without changing dimensions.

---

## 12. GLOBAL "DO NOT" LIST

1. DO NOT use a slider/carousel library (Swiper, Splide, Embla, Slick, Keen).
2. DO NOT put `transform: scale()` inside the card `transform`; the scale goes in the `scale` property.
3. DO NOT vertically center the cards; all of them sit on the bottom (`bottom:0`, origin `bottom left`).
4. DO NOT add autoplay, arrows, pagination, snap or a progress bar.
5. DO NOT add rounding, shadow or border to cards, nav or text.
6. DO NOT scroll the page; the wheel moves only the carousel (`preventDefault`).
7. DO NOT replace Mango Grotesque with Bebas, Oswald, Anton or any other font (Antonio is only the load-failure fallback).
8. DO NOT add any element that is not in section 1.1 (no pop-ups, banners, cookie bars, footer or extra sections).
9. DO NOT hotlink any third-party file; every asset comes from the R2 base of this document.
10. DO NOT leave the display text behind the cards (z 90 > max 21).
11. DO NOT use a white background or black text anywhere: the theme is dark (#000 / #fff). DO NOT use "almost black" dark grays (#111, #121212) for the background; it is pure #000.
12. DO NOT rely only on `addEventListener("resize")` to recompute the carousel (section 6.7).
13. DO NOT write a fixed `font-size` on `.fat` inside media queries: use the `--fat-size` variable (section 10.2).

---

## 13. SELF-CHECK (run before answering)

At **1440 × 900** viewport, 3 s after load, without touching the wheel:

- [ ] `getComputedStyle(document.documentElement).fontSize === "14.4px"`
- [ ] `.fat` → x 22, y 58, h 185 (±1) once Mango is loaded; `.fat.snd` → y 214 (±1); `h1.h1-home` → y 413 (±1). (If the font is blocked or unreachable, the `no-mango` fallback applies and these three values do not hold; that is expected, see sections 3.2 and 7.)
- [ ] `document.fonts.check('700 1em "Mango Grotesque"') === true` and `<html>` does NOT have the class `no-mango`; if the font is unreachable it has that class instead
- [ ] 16 `.card` elements; card 0 with `style.scale === "0.28675"`, `transform === "translateX(0px)"`, `zIndex === "3"`
- [ ] card 3: `scale 0.982`, `translateX(648px)`; card 6+: `scale 1.6`, `zIndex 16`
- [ ] the base of every visible card at y = 900
- [ ] **16 `<img>` loaded** (`naturalWidth > 0`), none with the `.broken` class, using exactly the 4 URLs of section 7
- [ ] `wheel` with `deltaY:400` on the hero → after ~2 s every `translateX` increased by 200px (or wrapped around the loop)
- [ ] hovering a card → tag "● PROJECT NN" appears above the card in 0.4 s; image zooms to 1.03 in 0.6 s
- [ ] dragging 300px to the left and releasing → cards move left; the link `click` does not fire
- [ ] `body` background `rgb(0, 0, 0)` and text `rgb(255, 255, 255)`; black loader with white logo
- [ ] logo: an inline SVG with exactly 2 shapes (1 `path` flame outline + 1 `circle`), rendered 45 × 38px at x 22, y 14; `document.title === "Lume | AI Production Studio"`; logo `aria-label === "Lume, home"`; `document.documentElement.lang === "en"`
- [ ] nav: white text with `mix-blend-mode:difference`, 5 links + logo; "MENU" hidden
- [ ] at 375 × 812: cards 288 × 300 at x = 4 / 304, y = 504; infinite horizontal scroll; no error when resizing from 1440 to 375 and back
- [ ] the loader leaves within ~2 s **even with the Mango font unavailable or blocked**; the title shows in the fallback font, with no loader stuck
- [ ] **proportional scaling** (fresh load at 1093 × 948, rem = 10.93px, Mango loaded): `.fat` x 16, y 44, h 140; cards with widths 94 / 170 / 246 / 322 / 398 / 474 and x 0 / 85 / 246 / 483 / 796 / 1185
- [ ] **resize without reloading**: open at 1920 × 1080 and resize to 1093 × 948, then 992 × 800, then 1440 × 900 → at 1093 the widths above; at 1440 cards 124 / 224 / 324 / 424 / 524 / 624 and x 0 / 112 / 324 / 636 / 1049 / 1561 (identical to a fresh load)
- [ ] at 1920 × 1080 (rem = 17.5px, Mango loaded): `.fat` x 26, y 70, h 225; `.fat.snd` y 260; h1 y 502; card 0 is 176px wide
- [ ] h1 on 2 lines ("…SÃO PAULO," / "…LUXURY BRANDS.") at 1440, 1093 and 375 px (3 lines only with the Antonio fallback)
- [ ] console without errors; no CORS or 404 error on the font request; no request to domains other than cdnjs.cloudflare.com, fonts.googleapis.com, fonts.gstatic.com and `assets.layerweb.com.br`
