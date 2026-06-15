---
name: clone
description: "Clone any website into a production-ready Next.js + Tailwind client project. Extracts design tokens, animations, layout, typography, and all sections. Then swaps in client assets. Usage: '/clone https://example.com for client-name' or just '/clone https://example.com'"
---

# /clone — Full Website Clone + Rebuild

Clone a website pixel-perfectly, rebuild it as Next.js + Tailwind, then optionally swap in client assets.

## Trigger Examples
- `/clone https://example.com`
- `/clone https://example.com for acme-corp`
- `clone this site: https://...`

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

### Step 1: Scout the site structure
Use `firecrawl_map` to discover all URLs on the site:
```
tool: firecrawl_map
url: <SOURCE_URL>
```
Report back: "Found X pages: [list top 10]"

Then use `browser_navigate` (Playwright) to load the homepage and take a full screenshot:
```
tool: browser_navigate → url: <SOURCE_URL>
tool: browser_screenshot → name: "original-homepage", fullPage: true
```

### Step 2: Extract the design system
Use `browser_run_code_unsafe` to extract ALL design tokens in one pass:
```js
const extract = () => {
  // CSS Variables
  const vars = {};
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules) {
        if (rule.selectorText === ':root') {
          for (const prop of rule.style) {
            if (prop.startsWith('--')) vars[prop] = rule.style.getPropertyValue(prop).trim();
          }
        }
      }
    } catch(e) {}
  }

  // Typography
  const fonts = [...new Set([...document.querySelectorAll('h1,h2,h3,p,a,button,span')]
    .map(el => getComputedStyle(el).fontFamily))].slice(0,8);
  
  // Color palette (most used)
  const colorMap = {};
  [...document.querySelectorAll('*')].forEach(el => {
    const s = getComputedStyle(el);
    [s.color, s.backgroundColor].forEach(c => {
      if (c && c !== 'rgba(0, 0, 0, 0)' && c !== 'transparent' && c !== 'rgb(0, 0, 0)' && c !== 'rgb(255, 255, 255)') {
        colorMap[c] = (colorMap[c] || 0) + 1;
      }
    });
  });
  const colors = Object.entries(colorMap).sort((a,b)=>b[1]-a[1]).slice(0,15).map(([c])=>c);

  // Spacing/sizing scale
  const spacings = [...new Set([...document.querySelectorAll('*')].flatMap(el => {
    const s = getComputedStyle(el);
    return [s.marginTop, s.paddingTop, s.gap].filter(v => v && v !== '0px');
  }))].slice(0,20);

  // Animations & transitions
  const keyframes = [];
  const transitions = new Set();
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules) {
        if (rule instanceof CSSKeyframesRule) keyframes.push({name: rule.name, text: rule.cssText.slice(0,500)});
        if (rule.style?.transition) transitions.add(rule.style.transition);
      }
    } catch(e) {}
  }

  // Layout structure (sections)
  const sections = [...document.querySelectorAll('section, [class*="section"], [class*="hero"], [class*="feature"], [class*="pricing"], [class*="testimonial"], [class*="footer"], [class*="nav"], header, footer, nav, main > div')]
    .slice(0,20)
    .map(el => ({
      tag: el.tagName,
      id: el.id,
      classes: el.className.toString().split(' ').slice(0,5).join(' '),
      children: el.children.length,
      text: el.innerText?.slice(0,100)
    }));

  // Font sources
  const fontLinks = [...document.querySelectorAll('link[href*="font"], link[href*="Font"]')].map(l=>l.href);
  const fontFaces = [];
  for (const sheet of document.styleSheets) {
    try {
      for (const rule of sheet.cssRules) {
        if (rule instanceof CSSFontFaceRule) fontFaces.push(rule.cssText.slice(0,200));
      }
    } catch(e) {}
  }

  // Border radius scale
  const radii = [...new Set([...document.querySelectorAll('button, a, [class*="card"], [class*="btn"]')]
    .map(el => getComputedStyle(el).borderRadius)
    .filter(v => v && v !== '0px'))].slice(0,8);

  return { vars, fonts, colors, spacings, keyframes, transitions: [...transitions].slice(0,10), sections, fontLinks, fontFaces, radii };
};
console.log(JSON.stringify(extract(), null, 2));
```

Save the full output to `CLIENT_DIR/design-tokens.json`.

### Step 3: Scrape full HTML source
Use `firecrawl_scrape` on the homepage (and any key pages like /pricing, /about if they exist):
```
formats: ["html", "rawHtml", "markdown"]
```
Save to `CLIENT_DIR/source/index.html`.

### Step 4: Capture all sections visually
Use Playwright to scroll through the page and screenshot each major section:
```js
// Scroll to each section and screenshot
document.querySelectorAll('section, header, footer, [class*="hero"], [class*="feature"]').forEach((el, i) => {
  el.scrollIntoView();
});
```
Take screenshots at key scroll positions. Save as `CLIENT_DIR/screenshots/section-N.png`.

### Step 5: Network asset capture
Use `browser_run_code_unsafe` to get all external assets:
```js
const assets = {
  images: [...document.querySelectorAll('img')].map(i => ({src: i.src, alt: i.alt, width: i.naturalWidth})),
  svgs: [...document.querySelectorAll('svg use, img[src*=".svg"]')].map(s => s.src || s.getAttribute('href')),
  fonts: performance.getEntriesByType('resource').filter(r => r.initiatorType === 'css' || r.name.match(/\.(woff2?|ttf|otf)/)).map(r => r.name),
  videos: [...document.querySelectorAll('video source, video[src]')].map(v => v.src),
};
console.log(JSON.stringify(assets, null, 2));
```
Save to `CLIENT_DIR/source/assets-manifest.json`.

### Step 6: Build the Next.js project
Create `CLIENT_DIR/` as a full Next.js 14 App Router project:

```bash
cd /Users/amanuel2x/clients/LBN/Agency/clients
npx create-next-app@latest <CLIENT_SLUG> --typescript --tailwind --app --no-src-dir --import-alias "@/*" --yes
cd <CLIENT_SLUG>
npm install framer-motion
npm install @next/font
```

Structure to create:
```
<CLIENT_SLUG>/
├── app/
│   ├── layout.tsx          # Root layout with fonts + metadata
│   ├── page.tsx            # Homepage assembles all sections
│   └── globals.css         # CSS variables + base styles
├── components/
│   ├── sections/           # One file per section (Hero, Features, etc.)
│   ├── ui/                 # Reusable UI: Button, Card, Badge, etc.
│   └── layout/             # Header, Footer, Nav
├── config/
│   └── client.ts           # ALL client-specific content here
├── lib/
│   └── animations.ts       # Framer Motion variants from extracted keyframes
├── public/
│   └── assets/             # Client logo, images go here
└── tailwind.config.ts      # Extended with extracted colors/fonts
```

### Step 7: Generate all files from extracted tokens

**tailwind.config.ts** — map extracted colors and fonts:
```ts
// Use extracted colors as named tokens: primary, secondary, accent, etc.
// Map top 5 colors from design-tokens.json
```

**app/globals.css** — recreate CSS variables:
```css
:root {
  /* Paste all extracted --variables from design-tokens.json */
}
```

**lib/animations.ts** — recreate all animations:
```ts
// Convert each extracted keyframe → Framer Motion variant
export const fadeInUp = { hidden: {opacity:0,y:20}, visible: {opacity:1,y:0} };
// etc for each extracted animation
```

**config/client.ts**:
```ts
export const client = {
  name: "<DERIVED_FROM_SITE>",
  logo: "/assets/logo.svg",
  tagline: "",
  primaryColor: "<TOP_COLOR_1>",
  accentColor: "<TOP_COLOR_2>",
  phone: "",
  email: "",
  address: "",
  socialLinks: { instagram: "", facebook: "", twitter: "", linkedin: "" },
  heroImage: "/assets/hero.jpg",
  heroHeadline: "<EXTRACTED_H1_TEXT>",
  heroSubline: "<EXTRACTED_HERO_SUBTEXT>",
  ctaText: "<EXTRACTED_CTA_TEXT>",
  ctaLink: "/contact",
  nav: { links: [] },
  sections: { /* which sections to show/hide */ }
};
```

### Step 8: Build each section as a React component

For each section identified in Step 2, create `components/sections/<SectionName>.tsx`:
- Preserve exact layout proportions (grid columns, padding ratios)
- Use Framer Motion for any animations that were detected
- Pull all text from `client` config, not hardcoded
- Use Tailwind classes that map to extracted colors/spacing
- Mobile responsive (md: breakpoints)

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

### Step 10: Final report

Report to user:
```
✅ Clone complete: <CLIENT_SLUG>
📁 Location: /Users/amanuel2x/clients/LBN/Agency/LBN/<CLIENT_SLUG>
🎨 Design tokens extracted: X colors, X fonts, X animations
📐 Sections built: [list section components]
📸 Screenshots saved: clients/<CLIENT_SLUG>/screenshots/

Next steps:
  1. Drop assets into:  Maarnet/clients/<CLIENT_SLUG>/public/assets/
     → logo.svg, hero.jpg, brand-colors.json, copy.json
  2. Run: /swap-assets <CLIENT_SLUG>
  3. Run: cd Maarnet/clients/<CLIENT_SLUG> && npm run dev
```
