---
name: clone
description: "Clone any website into a production-ready Next.js + Tailwind client project. Extracts design tokens, animations, layout, typography, and all sections. Then swaps in client assets. Usage: '/clone https://example.com for client-name' or just '/clone https://example.com'"
---

# /clone — Full Website Clone + Rebuild

Clone a website pixel-perfectly, rebuild it as Next.js + Tailwind, then optionally swap in client assets.

**Fidelity is the prime directive.** The rebuilt site must match the original to the T — every section, every color, every font, every animation (scroll, hover, entrance, looping), every breakpoint. When in doubt, capture MORE than you think you need. A clone that "looks about right" is a failed clone. Do not stop until a screenshot-diff pass (Step 11) confirms parity.

## Trigger Examples
- `/clone https://example.com`
- `/clone https://example.com for acme-corp`
- `clone this site: https://...`

---

## Prerequisites — MCP setup (do this once)

This skill REQUIRES two MCP servers: **Playwright** (renders the live site, runs JS, captures animations + screenshots) and **Firecrawl** (maps the sitemap and scrapes rendered HTML/markdown). Before cloning, verify both are connected:

```bash
claude mcp list
```

You want to see `playwright` and `firecrawl` both marked **✓ Connected**. If either is missing, set it up below, then re-run `claude mcp list` to confirm.

> Note: this pipeline uses **Firecrawl** (web scraping/mapping), not Firebase. Firebase is a database/hosting product and has no role in cloning. If you were told "Firebase MCP," it almost certainly means Firecrawl.

### Playwright MCP

Browser automation — navigation, full-page screenshots, `browser_run_code_unsafe` (in-page JS), network capture, hover/scroll simulation.

```bash
# Recommended: the official Microsoft Playwright MCP via npx (auto-updates)
claude mcp add playwright -- npx -y @playwright/mcp@latest

# First run downloads the browser binaries; if screenshots fail with a
# "browser not installed" error, install them explicitly:
npx -y playwright install chromium
```

Verify:
```bash
claude mcp list | grep playwright   # → playwright: ... ✓ Connected
```

Tools this skill uses: `browser_navigate`, `browser_take_screenshot` (or `browser_screenshot`), `browser_run_code_unsafe`, `browser_evaluate`, `browser_resize`, `browser_hover`, `browser_network_requests`, `browser_wait_for`.

### Firecrawl MCP

Sitemap discovery + clean rendered scraping (handles JS-rendered SPAs that raw `curl` can't).

1. Get a free API key at https://www.firecrawl.dev (dashboard → API Keys).
2. Add the server with the key in its environment:

```bash
# npx form (env var inline)
claude mcp add firecrawl --env FIRECRAWL_API_KEY=fc-YOUR_KEY_HERE -- npx -y firecrawl-mcp

# If your gh/Claude config prefers a persisted env, also export it in your shell profile:
#   echo 'export FIRECRAWL_API_KEY=fc-YOUR_KEY_HERE' >> ~/.zshrc && source ~/.zshrc
```

Verify:
```bash
claude mcp list | grep firecrawl    # → firecrawl: ... ✓ Connected
```

Tools this skill uses: `firecrawl_map`, `firecrawl_scrape`, `firecrawl_crawl` (optional, for multi-page sites), `firecrawl_extract`.

### If an MCP is unavailable mid-clone
- **No Playwright:** you cannot capture animations, computed styles, or screenshots — these are non-negotiable for fidelity. Stop and tell the user to set up Playwright. Do NOT fall back to scraping static HTML and calling it a clone.
- **No Firecrawl:** you can degrade gracefully — use `browser_navigate` + `browser_run_code_unsafe` to read `document.documentElement.outerHTML` for the HTML, and skip multi-page sitemap discovery (clone the homepage only, and ask the user for additional page URLs). Tell the user Firecrawl would improve coverage.

---

## Instructions

### Step 0: Parse the command
Extract:
- `SOURCE_URL` — the URL to clone
- `CLIENT_SLUG` — if provided after "for", use it; otherwise derive from domain (e.g. `example.com` → `example`)
- `CLIENT_DIR` = `/Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG>`

Check if client already exists:
```bash
ls /Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG> 2>/dev/null
```
If it does, ask: "Client '<CLIENT_SLUG>' already exists. Overwrite or create '<CLIENT_SLUG>-2'?"

Confirm MCP prereqs are connected (`claude mcp list`) before proceeding.

### Step 1: Scout the site structure
Use `firecrawl_map` to discover all URLs on the site:
```
tool: firecrawl_map
url: <SOURCE_URL>
```
Report back: "Found X pages: [list top 10]". Decide with the user (or default to homepage + obvious key pages: /, /about, /pricing, /services, /contact) which pages to clone for full fidelity.

Then use Playwright to load the homepage. **Critically: render it like a real browser before capturing anything** — lazy-loaded images, scroll-triggered animations, and web fonts only appear after interaction:

```
tool: browser_navigate → url: <SOURCE_URL>
tool: browser_resize   → width: 1440, height: 900   (canonical desktop viewport)
```

Then run the "full render" priming routine via `browser_run_code_unsafe` so EVERYTHING loads before extraction:
```js
// Prime the page: wait for fonts, scroll the full height to trigger lazy-load
// + IntersectionObserver entrance animations, then return to top.
await document.fonts.ready;
const sleep = ms => new Promise(r => setTimeout(r, ms));
const h = document.body.scrollHeight;
for (let y = 0; y <= h; y += Math.round(window.innerHeight * 0.6)) {
  window.scrollTo(0, y); await sleep(350);
}
window.scrollTo(0, 0); await sleep(600);
console.log(JSON.stringify({ scrollHeight: h, fontsReady: true }));
```

Now take the full-page screenshot (this is the master reference for the Step 11 diff):
```
tool: browser_take_screenshot → filename: "original-homepage-desktop.png", fullPage: true
```

### Step 2: Extract the design system (deep)
Use `browser_run_code_unsafe` to extract ALL design tokens in one pass. This is the hardened extraction — it pulls **every** `@keyframes` verbatim, every CSS variable from every rule (not just `:root`), the full font stack with weights, transition/animation declarations per element, and the spacing/radius/shadow scales:

```js
const extract = () => {
  const out = { vars:{}, keyframes:[], fontFaces:[], fontLinks:[], fonts:{}, colors:[],
                spacings:[], radii:[], shadows:[], transitions:[], animations:[], gradients:[],
                zIndexes:[], breakpointHints:[], sections:[] };

  // --- All CSS variables (from :root AND any scoped rule) ---
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules) {
        if (rule.style) {
          for (const prop of rule.style) {
            if (prop.startsWith('--')) out.vars[prop] = rule.style.getPropertyValue(prop).trim();
          }
        }
        // --- @keyframes captured VERBATIM and in full (no truncation) ---
        if (rule instanceof CSSKeyframesRule) out.keyframes.push({ name: rule.name, cssText: rule.cssText });
        // --- @font-face full text ---
        if (rule instanceof CSSFontFaceRule) out.fontFaces.push(rule.cssText);
        // --- @media breakpoint hints ---
        if (rule instanceof CSSMediaRule && rule.conditionText) out.breakpointHints.push(rule.conditionText);
      }
    } catch(e) {}
  }
  out.keyframes = out.keyframes.filter((k,i,a) => a.findIndex(x=>x.name===k.name)===i);
  out.breakpointHints = [...new Set(out.breakpointHints)].slice(0, 30);

  // --- Font links + a font-by-role map (weight + size + line-height + letter-spacing) ---
  out.fontLinks = [...document.querySelectorAll('link[href*="font" i], link[href*="googleapis"]')].map(l => l.href);
  const roleSel = { h1:'h1', h2:'h2', h3:'h3', body:'p', link:'a', button:'button, .btn, [class*="button" i]', nav:'nav a' };
  for (const [role, sel] of Object.entries(roleSel)) {
    const el = document.querySelector(sel); if (!el) continue;
    const s = getComputedStyle(el);
    out.fonts[role] = { fontFamily:s.fontFamily, fontWeight:s.fontWeight, fontSize:s.fontSize,
                        lineHeight:s.lineHeight, letterSpacing:s.letterSpacing, textTransform:s.textTransform };
  }

  // --- Color palette by usage frequency (keep rgba/hsla as-is for exactness) ---
  const colorMap = {}, gradSet = new Set(), shadowSet = new Set();
  [...document.querySelectorAll('*')].forEach(el => {
    const s = getComputedStyle(el);
    [s.color, s.backgroundColor, s.borderColor].forEach(c => {
      if (c && c !== 'rgba(0, 0, 0, 0)' && c !== 'transparent') colorMap[c] = (colorMap[c]||0)+1;
    });
    if (s.backgroundImage && s.backgroundImage.includes('gradient')) gradSet.add(s.backgroundImage);
    if (s.boxShadow && s.boxShadow !== 'none') shadowSet.add(s.boxShadow);
  });
  out.colors = Object.entries(colorMap).sort((a,b)=>b[1]-a[1]).slice(0,24).map(([c,n])=>({color:c,count:n}));
  out.gradients = [...gradSet].slice(0,12);
  out.shadows = [...shadowSet].slice(0,12);

  // --- Spacing, radius, z-index, transition/animation scales ---
  const spaceSet=new Set(), radSet=new Set(), zSet=new Set(), transSet=new Set(), animSet=new Set();
  [...document.querySelectorAll('*')].forEach(el => {
    const s = getComputedStyle(el);
    [s.marginTop,s.marginBottom,s.paddingTop,s.paddingBottom,s.gap].forEach(v=>{ if(v&&v!=='0px') spaceSet.add(v); });
    if (s.borderRadius && s.borderRadius!=='0px') radSet.add(s.borderRadius);
    if (s.zIndex && s.zIndex!=='auto') zSet.add(s.zIndex);
    if (s.transition && s.transition!=='all 0s ease 0s') transSet.add(s.transition);
    if (s.animationName && s.animationName!=='none')
      animSet.add(JSON.stringify({name:s.animationName,dur:s.animationDuration,timing:s.animationTimingFunction,
                                  delay:s.animationDelay,iter:s.animationIterationCount,dir:s.animationDirection}));
  });
  out.spacings=[...spaceSet].slice(0,30); out.radii=[...radSet].slice(0,12);
  out.zIndexes=[...zSet].slice(0,12); out.transitions=[...transSet].slice(0,24);
  out.animations=[...animSet].map(s=>JSON.parse(s)).slice(0,30);

  // --- Section inventory: ordered, with bounding box + computed layout per section ---
  const top = [...document.querySelectorAll('body > *, main > *, section, header, footer, nav')]
    .filter(el => el.offsetHeight > 40);
  out.sections = top.slice(0,40).map((el,i) => {
    const s = getComputedStyle(el), r = el.getBoundingClientRect();
    return { order:i, tag:el.tagName, id:el.id, classes:el.className.toString().slice(0,160),
             display:s.display, flexDir:s.flexDirection, gridCols:s.gridTemplateColumns,
             justify:s.justifyContent, align:s.alignItems, gap:s.gap,
             paddingY:`${s.paddingTop}/${s.paddingBottom}`, bg:s.backgroundColor,
             width:Math.round(r.width), height:Math.round(r.height),
             text:(el.innerText||'').slice(0,140) };
  });

  return out;
};
console.log(JSON.stringify(extract(), null, 2));
```

Save the full output to `CLIENT_DIR/design-tokens.json`. **Do not truncate keyframes or font-faces** — they are the source of animation fidelity.

### Step 2b: Capture DYNAMIC animations (the part static extraction misses)
CSS extraction catches declared animations, but many sites animate via JS (GSAP, Framer Motion, Lenis, AOS, Intersection-triggered classes). Detect and document these so they can be faithfully recreated:

```js
const detectMotion = () => {
  const libs = {
    gsap: !!window.gsap || !!window.TweenMax,
    framerMotion: !!document.querySelector('[style*="transform"][data-projection-id], [data-framer-name]'),
    aos: !!document.querySelector('[data-aos]'),
    lenis: !!document.querySelector('.lenis') || !!window.Lenis,
    swiper: !!document.querySelector('.swiper'),
    locomotive: !!document.querySelector('[data-scroll]'),
    splide: !!document.querySelector('.splide'),
  };
  // Elements that carry scroll/entrance hooks
  const triggered = [...document.querySelectorAll('[data-aos],[data-scroll],[class*="animate" i],[class*="reveal" i],[class*="fade" i],[class*="slide" i]')]
    .slice(0,60).map(el => ({ tag:el.tagName, classes:el.className.toString().slice(0,120),
                              dataAos:el.getAttribute('data-aos'), dataScroll:el.getAttribute('data-scroll') }));
  // Currently-running CSS animations on screen
  const running = (typeof document.getAnimations==='function' ? document.getAnimations() : [])
    .slice(0,40).map(a => ({ type:a.constructor.name,
                             name:a.animationName||(a.effect&&a.effect.getKeyframes&&'css-anim'),
                             playState:a.playState }));
  return { libs, triggered, running };
};
console.log(JSON.stringify(detectMotion(), null, 2));
```

Then **observe entrance animations in action**: scroll slowly and screenshot at intervals so you can SEE what fades/slides/scales in. Also probe **hover states** on interactive elements (buttons, cards, nav links) with `browser_hover`, screenshotting before/after to capture hover transitions. Save findings to `CLIENT_DIR/source/motion-manifest.json` with, for each animated element: what triggers it (scroll-into-view / hover / load / loop), the transform/opacity deltas, duration, easing, and stagger. This manifest drives `lib/animations.ts`.

### Step 3: Scrape full HTML source
Use `firecrawl_scrape` on the homepage (and each key page from Step 1):
```
formats: ["html", "rawHtml", "markdown"]
onlyMainContent: false        # keep nav/footer/everything — we want the WHOLE page
waitFor: 3000                 # let JS render
```
Save to `CLIENT_DIR/source/<page>.html` (homepage → `index.html`). Also save the Playwright-rendered DOM as a cross-check (`document.documentElement.outerHTML`) to `CLIENT_DIR/source/index.rendered.html`, since Firecrawl and a live browser can differ on JS-heavy sites.

### Step 4: Capture all sections visually (every section, every breakpoint)
Fidelity requires per-section reference shots at multiple widths. For EACH viewport, resize then capture each section:

```
Viewports: desktop 1440×900, tablet 768×1024, mobile 390×844
```
For each viewport: `browser_resize` → run the Step-1 priming scroll → screenshot full page (`section-overview-<vp>.png`), then scroll each section into view and screenshot it individually:
```js
const els = [...document.querySelectorAll('body > *, main > *, section, header, footer')].filter(e=>e.offsetHeight>40);
els.forEach((el,i)=> el.setAttribute('data-clone-idx', i));   // tag for stable reference
console.log(JSON.stringify(els.map((el,i)=>({i, h:el.offsetHeight, cls:el.className.toString().slice(0,80)})), null, 2));
```
Save as `CLIENT_DIR/screenshots/<vp>/section-<i>.png`. These are checked against the rebuild in Step 11.

### Step 5: Network asset capture (download everything, don't hotlink)
Get the complete asset inventory, then DOWNLOAD assets locally so the clone is self-contained:
```js
const assets = {
  images: [...document.querySelectorAll('img')].map(i => ({src:i.currentSrc||i.src, srcset:i.srcset, alt:i.alt, w:i.naturalWidth, h:i.naturalHeight, loading:i.loading})),
  bgImages: [...document.querySelectorAll('*')].map(el=>getComputedStyle(el).backgroundImage).filter(b=>b&&b!=='none'&&b.includes('url(')),
  svgsInline: [...document.querySelectorAll('svg')].length,
  svgFiles: [...document.querySelectorAll('img[src$=".svg" i], use')].map(s=>s.src||s.getAttribute('href')||s.getAttribute('xlink:href')),
  fonts: performance.getEntriesByType('resource').filter(r=>/\.(woff2?|ttf|otf|eot)(\?|$)/i.test(r.name)).map(r=>r.name),
  videos: [...document.querySelectorAll('video source, video[src]')].map(v=>v.src),
  posters: [...document.querySelectorAll('video[poster]')].map(v=>v.poster),
};
console.log(JSON.stringify(assets, null, 2));
```
Save to `CLIENT_DIR/source/assets-manifest.json`. Then download the real bytes (images, fonts, videos, SVGs) into `CLIENT_DIR/public/assets/` so nothing hotlinks back to the source domain:
```bash
mkdir -p "$CLIENT_DIR/public/assets/fonts" "$CLIENT_DIR/public/assets/img"
# For each URL in assets-manifest.json:
curl -sL "<asset-url>" -o "$CLIENT_DIR/public/assets/img/<filename>"
# fonts → public/assets/fonts/   (then reference via @font-face in globals.css)
```
**Self-host the exact fonts.** Matching the typeface is the single biggest fidelity factor — never substitute a "close enough" font. If the original uses Google Fonts, pull the exact families/weights via `next/font/google`; if custom, copy the woff2 files and recreate the `@font-face` rules verbatim from the extracted `fontFaces`.

### Step 6: Build the Next.js project
Create `CLIENT_DIR/` as a full Next.js 14 App Router project:
```bash
cd /Users/amanuel2x/clients/LBN/Agency/LBN
npx create-next-app@latest <CLIENT_SLUG> --typescript --tailwind --app --no-src-dir --import-alias "@/*" --yes
cd <CLIENT_SLUG>
npm install framer-motion
```
> Note: `@next/font` is deprecated — fonts come from the built-in `next/font` (no install needed). Add `lenis` only if the original used smooth-scroll; add `swiper`/`embla-carousel-react` only if the original had a carousel.

Structure to create:
```
<CLIENT_SLUG>/
├── app/
│   ├── layout.tsx          # Root layout with fonts (next/font) + metadata
│   ├── page.tsx            # Homepage assembles all sections IN ORDER
│   └── globals.css         # CSS variables + @font-face + @keyframes (verbatim)
├── components/
│   ├── sections/           # One file per section (Hero, Features, etc.)
│   ├── ui/                 # Reusable UI: Button, Card, Badge, etc.
│   └── layout/             # Header, Footer, Nav
├── config/
│   └── client.ts           # ALL client-specific content here
├── lib/
│   └── animations.ts       # Framer Motion variants from keyframes + motion-manifest
├── public/
│   └── assets/             # Downloaded logo, images, fonts, videos
└── tailwind.config.ts      # Extended with extracted colors/fonts/spacing/radius/shadows
```

### Step 7: Generate all files from extracted tokens (exact, not approximate)

**tailwind.config.ts** — map the EXTRACTED scales as named tokens (don't round to Tailwind defaults):
```ts
// theme.extend.colors  ← every entry from design-tokens.colors (name them: primary, accent, ink, bg, muted…)
// theme.extend.fontFamily ← exact families from design-tokens.fonts
// theme.extend.spacing, borderRadius, boxShadow, zIndex ← the extracted scales
// theme.extend.keyframes + theme.extend.animation ← converted from design-tokens.keyframes
//   so CSS-driven animations also work via Tailwind utility classes
```

**app/globals.css** — recreate exactly:
```css
:root {
  /* Paste EVERY extracted --variable from design-tokens.json verbatim */
}
/* Paste EVERY @font-face from design-tokens.fontFaces (or use next/font) */
/* Paste EVERY @keyframes from design-tokens.keyframes verbatim */
```

**lib/animations.ts** — recreate all animations from BOTH sources:
```ts
// For each CSS @keyframes → an equivalent Framer Motion variant (match duration/easing/delay from
// design-tokens.animations and source/motion-manifest.json).
// For each scroll-triggered element in motion-manifest → a whileInView variant with the right
// transform/opacity deltas, duration, easing, and stagger.
// For each hover transition → a whileHover variant.
// Preserve looping animations (animationIterationCount: infinite) as `repeat: Infinity`.
export const fadeInUp = { hidden:{opacity:0,y:24}, visible:{opacity:1,y:0, transition:{duration:0.6, ease:[0.22,1,0.36,1]}} };
// …one per detected animation. Do NOT invent generic animations — mirror what was measured.
```

**config/client.ts**:
```ts
export const client = {
  name: "<DERIVED_FROM_SITE>",
  logo: "/assets/img/logo.svg",
  tagline: "",
  primaryColor: "<TOP_COLOR_1>",
  accentColor: "<TOP_COLOR_2>",
  phone: "", email: "", address: "",
  socialLinks: { instagram:"", facebook:"", twitter:"", linkedin:"" },
  heroImage: "/assets/img/hero.jpg",
  heroHeadline: "<EXTRACTED_H1_TEXT>",
  heroSubline: "<EXTRACTED_HERO_SUBTEXT>",
  ctaText: "<EXTRACTED_CTA_TEXT>",
  ctaLink: "/contact",
  nav: { links: [/* exact nav labels + hrefs from the original */] },
  sections: { /* ordered list of section components to render */ }
};
```

### Step 8: Build each section as a React component (match layout to the box model)
For each section from the Step-2 inventory (in order), create `components/sections/<SectionName>.tsx`:
- Recreate the EXACT layout: same flex/grid direction, same `gridTemplateColumns`, same gap, same vertical padding, same max-width container — use the per-section computed values captured in Step 2.
- Wire up the animations from `lib/animations.ts`: entrance on `whileInView`, hover on `whileHover`, loops with `repeat: Infinity`. Match stagger order to the original.
- Pull all text from `client` config, never hardcoded.
- Use the named Tailwind tokens (extracted colors/spacing/radius/shadows), not arbitrary values.
- Reproduce every breakpoint behavior seen in Step 4 (mobile/tablet/desktop) using `md:`/`lg:` — compare against the per-viewport screenshots.
- If the original had a carousel/marquee/accordion/parallax, rebuild that interaction, don't flatten it to a static block.

### Step 9: Register the client
Update `/Users/amanuel2x/clients/LBN/Agency/registry.json`:
```json
{
  "clients": [
    {
      "slug": "<CLIENT_SLUG>",
      "sourceUrl": "<SOURCE_URL>",
      "clonedAt": "<ISO_DATE>",
      "path": "/Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG>",
      "status": "cloned",
      "pages": [],
      "deployUrl": null
    }
  ]
}
```

### Step 10: Run it locally
```bash
cd /Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG>
npm run dev
```
Confirm it builds with no errors and the homepage renders.

### Step 11: Fidelity verification loop (DO NOT SKIP — this is what makes it "to the T")
Compare the rebuild against the original, side by side, and fix every drift:

1. With `npm run dev` running, use Playwright to navigate to `http://localhost:3000`, run the SAME priming scroll from Step 1, and take a full-page screenshot at each viewport (`clone-<vp>.png`).
2. Put `clone-<vp>.png` next to `original-homepage-<vp>.png` (and the per-section shots) and **visually diff them**. Walk top to bottom and check:
   - Section order, presence, and proportions (nothing missing, nothing reordered)
   - Colors and gradients (exact, not "similar")
   - Fonts: family, weight, size, line-height, letter-spacing
   - Spacing/padding rhythm and container widths
   - **Animations**: trigger an entrance scroll and hovers on the clone — do they fade/slide/scale/loop like the original? Same duration and easing?
   - Responsive behavior at tablet + mobile
3. For each discrepancy, edit the relevant component / tokens / animation variant and re-screenshot. **Iterate until parity.** List any residual differences you couldn't resolve and why.

Only after Step 11 passes is the clone "done."

### Step 12: Final report
```
✅ Clone complete: <CLIENT_SLUG>
📁 Location: /Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG>
🎨 Design tokens: X colors, X fonts, X CSS @keyframes, X JS/scroll animations recreated
🎬 Motion libs detected: [gsap / framer / aos / …]  → recreated via [framer-motion / CSS]
📐 Sections built (in order): [list section components]
📸 Screenshots: original vs clone at desktop/tablet/mobile in screenshots/
🔍 Fidelity check: [PASS / list of residual diffs]

Next steps:
  1. Drop assets into:  <CLIENT_DIR>/public/assets/
     → logo.svg, hero.jpg, brand-colors.json, copy.json
  2. Run: /swap-assets <CLIENT_SLUG>
  3. Run: cd <CLIENT_DIR> && npm run dev
```
