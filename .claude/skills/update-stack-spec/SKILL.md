---
name: update-stack-spec
description: Updates stack-spec.md to the current releases of the tools it names. Checks every version line and stability claim against primary sources, reads the changelog or migration guide of each new major release, tests every "Revisit when" trigger, updates facts with dated sources, and proposes (never makes) changes of decision. Use when asked to update or refresh the stack spec, after a major release of a tool in it (Astro, TypeScript, Tailwind CSS, Solid, Node.js, Vite, Biome, Vitest, Playwright, KaTeX, Shiki, wasm-bindgen, Wrangler, npm), or when its version lines are old. Optional arguments name the tools to check; default is all.
---

# Update the stack spec

Facts (versions, dates, measurements, sources) you update. Decisions you propose to the author
and wait. Arguments, if given, name the tools to check; otherwise check all.

## 1. Inventory

From `stack-spec.md`, list:
- every row of §2 with its version line;
- every version, release date or stability claim in §3 and §4 ("stable", "release
  candidate", "alpha", "experimental", a version number next to a tool);
- every row of *Revisit when*, as a condition to test;
- the header's version date.

## 2. Check each against primary sources

Registry, release notes, official docs. Never memory.
- npm: `npm view <pkg> dist-tags time --json`. Current = newest release at least 14 days old;
  list younger releases as upcoming.
- nixpkgs tools (Node.js, Biome, nixfmt, Wrangler, Playwright browsers): `nix eval --raw
  nixpkgs#<pkg>.version` on `nixos-unstable`, plus the project's release page.
- Platform features (Cloudflare, browser support): the official docs page and its date.

Record per tool: spec's version line, current version, and whether a major release (or a
release the spec waits for) lies between.

## 3. Classify each change

Read every new major's changelog or migration guide in full, and the docs the spec relies on.
Then classify:
- **Fact** (version line, date, package count): update it.
- **Reason** (a stated drawback fixed, a new one introduced): update the reason's text; list it
  in the report.
- **Decision** (the choice itself, an alternative now better, a *Revisit when* trigger fired):
  do not edit. Report: what changed, source, options with costs, recommendation. Wait.

## 4. Edit

- Keep the spec's style: terse, rule first, reason after, ISO dates; mark changed decision text
  "Amended <date>: …".
- New fact → new source at the end of the numbered list, with "checked <date>". Never renumber
  sources; delete one only when nothing cites it.
- Never renumber sections. A new topic becomes the next subsection of its section; ask the
  author whether a site already uses that number, and report the number chosen.
- Update the header's version date.
- Name no site, user or repository that uses the spec.

## 5. Verify and commit

- Every new or changed URL resolves: `curl -sIL <url>`.
- No site, user or repository that uses the spec is named.
- Jujutsu, signed, one commit per theme (e.g. "Record Astro 8", "Propose TypeScript 7"), the
  message stating what changed, why, and the sources checked. Do not push unless asked.

## 6. Report

Per tool: spec's version → current. Then: facts updated, reasons changed, decisions proposed
with recommendations, upcoming releases not yet 14 days old.
