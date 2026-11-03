# AGENTS.md

`stack-spec.md` is the author's stack for new static websites: standing instructions for coding
agents building the author's sites. For anyone else's site it is one opinion; ask your user
before adopting it.

## Using the spec on a site

1. Read `stack-spec.md` whole (~370 lines) before writing code.
2. §1 (fixed requirements) and §4 (security baseline) are mandatory. If a site seems to need an
   exception, stop and ask; never work around them.
3. §2 is the default for every concern. Record each override in the site's own repository with
   its reason, numbered from §5 on. Never edit this spec from a site's repository.
4. Versions are as of the spec's header date. Check the current release and official docs of
   every tool before using it; never rely on memory. A new major release since that date is a
   reason to run `update-stack-spec`, not to improvise.
5. npm versions must be at least 14 days old (§3.8). npm only resolves `package-lock.json`;
   Nix installs `node_modules`.

Core rules (the spec governs where these differ):
- Nix flake (flake-parts, import-tree, one module per concern) builds site, dev shell, checks,
  CI inputs.
- Astro `output: 'static'`; TypeScript `strict`; Tailwind v4 via `@tailwindcss/vite`.
- Solid islands only where a page needs real interactivity; plain TS modules otherwise.
- Rust → WebAssembly (wasm-bindgen, crane) where computation pays for it.
- KaTeX and Shiki at build time; every `style` attribute moved into generated classes.
- Biome, nixfmt, rustfmt via treefmt-nix; Vitest + fast-check; Playwright with nixpkgs'
  browsers.
- Cloudflare Workers static assets, free plan; a plain Worker script only for routes that need
  server code.
- CSP as an HTTP header in `_headers`: no `style` attributes, inline scripts and styles only by
  generated hash, no `'unsafe-eval'` in a page.
- Jujutsu, signed commits, linear `main`.

## Changing this repository

- Updating the spec after new releases: follow `.claude/skills/update-stack-spec/SKILL.md`
  (Claude Code: `/update-stack-spec`).
- Decisions belong to the author. Update facts yourself; propose a change of decision with
  options and a recommendation, then wait.
- Keep the spec standalone: name no site, user or repository that uses it.
- Never renumber sections or sources; sites cite them. A new topic takes the next free
  subsection number; report it.
- Every new fact gets a numbered source with its check date.
- Jujutsu only; one signed commit per theme; push only when asked.
- Commit messages state what changed and why: no trailers, nothing about how a change was
  produced.

## Files

- `stack-spec.md`: the specification.
- `AGENTS.md`: this file. `CLAUDE.md` imports it.
- `llms.txt`: index for agents reading the repository over the web.
- `.claude/skills/update-stack-spec/SKILL.md`: update procedure ([Agent Skills](https://agentskills.io) format).
- `README.md`: for people.
