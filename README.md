# clone — a Claude Code skill for pixel-perfect website cloning

A [Claude Code](https://claude.com/claude-code) skill that clones any website into a production-ready **Next.js 14 + Tailwind CSS** project — extracting the design system, layout, typography, assets, and **every animation** (scroll, hover, entrance, looping), then rebuilding it section by section as clean React components.

The goal is fidelity to the **T**: a clone that "looks about right" is a failed clone. The pipeline render-primes the page, captures CSS *and* JS-driven motion, self-hosts the exact fonts and assets, and finishes with a screenshot-diff verification loop before it calls itself done.

> **Note:** the rebuilt site is a faithful reconstruction intended as a starting point (e.g. rebranding for a client). Only clone sites you have the right to copy. Respect copyright, trademarks, and licensing.

---

## What it does

```
/clone https://example.com for acme-corp
```

Running the skill executes a 12-step pipeline:

1. **Parse** the command → source URL + client slug + target directory.
2. **Map** the site with Firecrawl to discover all pages.
3. **Extract the design system** — every CSS variable, `@keyframes` (verbatim), `@font-face` rule, per-role font metrics, color palette by usage, gradients, shadows, and the spacing/radius/z-index scales.
4. **Capture dynamic animations** — detect GSAP / Framer Motion / AOS / Lenis / Swiper / Locomotive and build a motion manifest of what triggers each animation, its transform deltas, duration, easing, and stagger.
5. **Scrape** the full rendered HTML (Firecrawl + a live-browser DOM cross-check).
6. **Screenshot** every section at desktop, tablet, and mobile widths.
7. **Download** all assets and self-host the exact fonts — no hotlinking, no "close enough" substitutions.
8. **Scaffold** a Next.js 14 App Router project (TypeScript + Tailwind + Framer Motion).
9. **Generate** `tailwind.config.ts`, `globals.css`, `lib/animations.ts`, and `config/client.ts` from the extracted tokens.
10. **Build each section** as a React component, matching the original's box model and animations.
11. **Register** the client and run it locally.
12. **Verify fidelity** — screenshot-diff the rebuild against the original at every breakpoint and iterate until parity.

---

## Output structure

Each clone produces a self-contained Next.js project:

```
<client-slug>/
├── app/
│   ├── layout.tsx          # Root layout — fonts (next/font) + metadata
│   ├── page.tsx            # Homepage assembles all sections in order
│   └── globals.css         # CSS variables + @font-face + @keyframes (verbatim)
├── components/
│   ├── sections/           # One file per section (Hero, Features, …)
│   ├── ui/                 # Reusable UI: Button, Card, Badge, …
│   └── layout/             # Header, Footer, Nav
├── config/
│   └── client.ts           # ALL client-specific content lives here
├── lib/
│   └── animations.ts       # Framer Motion variants from keyframes + motion manifest
├── public/
│   └── assets/             # Downloaded logo, images, fonts, videos
└── tailwind.config.ts      # Extended with extracted colors/fonts/spacing/radius/shadows
```

All swappable content is isolated in `config/client.ts`, so the same clone can be re-skinned for a new brand without touching components.

---

## Requirements

| Requirement | Why |
|---|---|
| **Claude Code** | The runtime that executes the skill. |
| **Playwright MCP** | Renders the live site, runs in-page JS, captures animations and screenshots. **Required.** |
| **Firecrawl MCP** | Maps the sitemap and scrapes JS-rendered HTML. Recommended (degrades to homepage-only without it). |
| **Node.js 18+** | For `create-next-app` and running the generated project. |

> This pipeline uses **Firecrawl** for scraping — not Firebase. Firebase is a database/hosting product and plays no role in cloning.

### Install the MCP servers

**Playwright:**
```bash
claude mcp add playwright -- npx -y @playwright/mcp@latest
npx -y playwright install chromium      # if screenshots report "browser not installed"
```

**Firecrawl** (get a free key at https://www.firecrawl.dev):
```bash
claude mcp add firecrawl --env FIRECRAWL_API_KEY=fc-YOUR_KEY_HERE -- npx -y firecrawl-mcp
```

Verify both are connected:
```bash
claude mcp list      # → playwright and firecrawl should show ✓ Connected
```

---

## Installing the skill

Drop `SKILL.md` into a `clone/` folder inside your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/clone
curl -sL https://raw.githubusercontent.com/Amanuel2x/clone-skill/master/SKILL.md \
  -o ~/.claude/skills/clone/SKILL.md
```

Or clone this repo and copy the file:
```bash
git clone https://github.com/Amanuel2x/clone-skill.git
cp clone-skill/SKILL.md ~/.claude/skills/clone/SKILL.md
```

Restart Claude Code (or start a new session) and the `/clone` skill will be available.

---

## Usage

```
/clone https://example.com                 # clones, slug derived from the domain
/clone https://example.com for acme-corp   # clones into the "acme-corp" client slug
clone this site: https://example.com       # natural-language trigger
```

After a clone completes:

1. Drop the client's real assets into `public/assets/` (`logo.svg`, `hero.jpg`, `brand-colors.json`, `copy.json`).
2. Re-skin with a companion `swap-assets` step (if you use one).
3. `cd <client-slug> && npm run dev`.

---

## How fidelity is enforced

Most "website cloners" screenshot a half-loaded page and approximate the styling. This skill is built to avoid that:

- **Render-priming** — waits for `document.fonts.ready` and scrolls the full page height so lazy-loaded images and scroll-triggered animations fire *before* anything is captured.
- **Verbatim keyframes** — `@keyframes` and `@font-face` rules are copied in full, never truncated.
- **Motion manifest** — JS-driven animations (which static CSS extraction misses) are detected and documented with their triggers, deltas, durations, and easings, then recreated as Framer Motion variants.
- **Exact fonts** — the original typefaces are self-hosted (via `next/font` or copied `woff2` files); typeface substitution is the single biggest fidelity killer and is explicitly avoided.
- **Multi-viewport capture** — every section is screenshotted at desktop, tablet, and mobile.
- **Verification loop** — the rebuild is screenshot-diffed against the original top-to-bottom and iterated until it matches.

---

## License

MIT — see [LICENSE](LICENSE).
