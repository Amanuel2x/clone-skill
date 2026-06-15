# clone — website cloning skill

Claude Code skill that clones any website into a production-ready Next.js + Tailwind client project. Extracts design tokens, animations, layout, typography, and all sections, then swaps in client assets.

## Usage

```
/clone https://example.com for client-name
```
or just:
```
/clone https://example.com
```

## Install

Drop `SKILL.md` into a `clone/` folder inside your Claude Code skills directory:

```
~/.claude/skills/clone/SKILL.md
```

See `SKILL.md` for the full pipeline (map → screenshot → extract tokens → scrape → rebuild → componentize → config → register).
