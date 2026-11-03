# Stack specification for static websites

Standing instructions for LLM coding agents that build the author's new static websites. The
author's opinion, not a general recommendation. Decided 2026-10-10; amended 2026-10-11 (§3.4)
and 2026-10-29 (§3.8). Versions as of October 2026: check current releases and official docs
before relying on them. A site records its own decisions in its own repository, numbered from
§5 on. Section and source numbers are stable; cite them.

## 1. Fixed requirements

1. A Nix flake builds everything: the site, the dev shell, every check, CI's inputs. Layout:
   flake-parts, import-tree, one module per concern.
2. TypeScript with `strict`, never plain JavaScript.
3. Room for interactive WebAssembly components written in Rust.
4. A strict Content-Security-Policy as an HTTP header: no `style` attributes; inline scripts
   only by hash.
5. Cloudflare's free plan only.
6. As few npm packages as practical, installed by Nix from the lock file, never by
   `npm install`.
7. Jujutsu, signed commits, a linear `main`.

## 2. The stack, at a glance

| Concern | Choice | Version line | Chosen over |
|---|---|---|---|
| Site framework | **Astro**, `output: 'static'` | 7.x | Hakyll, Hugo, Eleventy, Lume, Zola |
| Markdown | Sätteri (Astro's default) without maths; **unified + remark-math + rehype-katex** with maths | — | — |
| Maths | **KaTeX at build time**, HTML+MathML output, CSS and fonts self-hosted | 0.18 | MathML only (Temml, KaTeX `output: 'mathml'`), MathJax, client-side rendering |
| Code highlighting | **Shiki**, light and dark themes, inline styles moved into classes (§3.3) | Astro's | Prism |
| Interactive islands | **Solid**, latest stable, through `@astrojs/solid-js`; 2.0 once it and Astro's support are stable | 1.9, then 2.0 | Preact, React, Svelte 5, Vue, Lit |
| Small scripts | Plain TS modules or custom elements | — | A framework |
| Animation | **Web Animations API** | Baseline | Motion |
| Computation | **Rust → `wasm32-unknown-unknown` + wasm-bindgen**, built by crane with rust-overlay | — | wasm-pack, Trunk, workers-rs, Leptos, Dioxus |
| Styling | **Tailwind v4 through `@tailwindcss/vite`**, design tokens in CSS | 4.3 | Standalone CLI, plain CSS |
| Images | `astro:assets` with sharp, **at build time** | — | Cloudflare's image binding |
| Fonts | Self-hosted, subset `woff2` | — | Google Fonts |
| Type checking | **TypeScript 6** + `astro check` | 6.x | TypeScript 7, for now |
| Lint and format | **Biome 2**, nixfmt, rustfmt, clippy, all through treefmt-nix | 2.x | ESLint + Prettier, Oxlint + Oxfmt |
| Tests | **Vitest** + fast-check, `cargo test`, **Playwright** with browsers from nixpkgs | Vitest 5 | `node:test`, Cypress |
| Package manager | **npm**, installed by Nix's `importNpmLock` | npm 11 | pnpm, Yarn, Bun |
| Node.js | **24 LTS**; 26 once it is LTS | — | Odd releases (Astro does not support them) |
| Hosting | **Cloudflare Workers**, static assets; a script beside them only where needed | — | Pages |
| Server code (optional) | **A plain Worker script beside the static assets**, for the few routes that need one (§3.10) | — | Astro routes via `@astrojs/cloudflare`, D1, workers-rs |
| CI | GitHub Actions: `nix flake check`, `nix build`, deploy with the flake's Wrangler | — | Workers Builds |

## 3. Why each choice

### 3.1 Site framework: Astro
- HTML by default; JavaScript ships only for islands; server code runs only on routes that opt
  in [1][2].
- Highest satisfaction among meta-frameworks in State of JS 2025 (13,002 responses), 39 points
  above Next.js [3].
- The Astro team joined Cloudflare in January 2026; Astro stays MIT-licensed with open
  governance [4]. Its Cloudflare adapter targets Workers only [5].
- Astro 7 (June 2026): Rust compiler, Vite 8 with Rolldown, Sätteri (Rust Markdown). Builds
  15–61% faster in Astro's benchmarks [6][7].
- Cost: about 300 npm packages for `astro` alone, the floor (§3.9).
- Not chosen:
  - Hakyll 4.17 (maintained [8]): no npm, but islands, TS bundling and images by hand.
  - Hugo: esbuild and KaTeX→MathML built in [9][10], but Tailwind needs the npm CLI since
    v0.161 [11]; no islands; Go templates.
  - Eleventy: lean (105–134 packages [12]), but v4 is alpha; no islands.
  - Lume: Deno, a second JavaScript runtime.
  - Zola: no islands.
- One stack for every site outweighs Hakyll's zero packages: components, design tokens and
  agent guidance carry over.

### 3.2 Markdown and maths
- Without maths: Sätteri, Astro 7's default: smart punctuation, GFM, heading IDs, directives
  [6][13].
- With maths: unified (`@astrojs/markdown-remark`) + remark-math + rehype-katex with
  `strict`, so errors fail the build. Sätteri only parses maths; the one renderer plugin is
  third-party [14][15]. Astro keeps unified supported [7].
- KaTeX's default output is HTML plus MathML [16]: identical in every browser, readable by
  screen readers. Self-host its CSS and fonts.
- MathML only (Temml, or KaTeX `output: 'mathml'`) needs no CSS, only a maths font; MathML Core
  is in every engine since January 2023 [17][18]. Leaner, but rendering differs by engine. Not
  yet; see *Revisit when*.

### 3.3 Maths and code without inline styles
- Shiki [19] and KaTeX's HTML output write `style` attributes; `style-src 'self'` makes
  browsers ignore them. Local previews apply them, so maths and code colours break only once
  deployed. Measured 2026-10-10 on a 27-page build with maths and code: 476 distinct `style`
  values, 363 KaTeX's, the rest Shiki's.
- Rule: one rehype plugin, run after rehype-katex and Shiki, replaces each `style` attribute in
  Markdown output with a class named after a hash of its value, and writes the rules into one
  generated stylesheet. It covers Shiki, so `transformerStyleToClass` [20] is not needed.
- Rule: a flake check fails on any `style` attribute in the built site, whatever produced it.
- Fallback, if the plugin proves impractical: `style-src-attr 'unsafe-inline'` (attributes
  only, not `<style>` or stylesheets; Baseline since December 2022 [47]). Weaker; only on a
  site without user content.
- MathML-only output would remove KaTeX's styles; see *Revisit when*.

### 3.4 Interactivity: Solid islands, plain TS for the rest
Decision (2026-10-10): Solid. Amended 2026-10-11: the latest stable Solid now (1.9); Solid 2.0
once both it and Astro's support for it are stable. Compared as a fresh start:

| | Preact | Solid | Svelte 5 |
|---|---|---|---|
| Download | ~3 kB plus hooks [21] | ~7 kB (estimate) | Small (estimate) |
| Toolchain fit (TS, Biome, one type checker) | TSX, full | TSX, full | `.svelte`: Biome experimental [24], plus `svelte-check` |
| Stable now | Yes (10.x; 11 just out) | 1.9 yes; 2.0 is a release candidate with breaking changes [23] | Yes |
| Animated, interactive widgets | Good with `@preact/signals`; plain hooks re-render the component | Best: fine-grained updates, components run once | Very good: fine-grained runes |
| Agents write correct code | Best (React model) | Weakest: React habits such as destructuring props silently break reactivity; 2.0 postdates most training data | Medium |
| Satisfaction, momentum [22] | Gaining interest | Highest satisfaction five years running, gaining | — |

- Why Solid: widgets that update a few values many times a second (scrubbable animations,
  live playgrounds) update exactly the dependent nodes, and components run once.
  Same TSX toolchain as Preact; small runtime.
- Not React: 65 KB gzipped (209 KB uncompressed, measured). Not Svelte: partial Biome support
  and a second type checker. Not Vue: heavier, nothing extra. Not Lit: no official Astro
  integration [1].
- Accepted risks:
  - Two majors, two migrations. Solid 2.0 is a release candidate (2.0.0-rc.14, 8 October 2026)
    whose API changes reach the code's structure [43][44][50]. Write islands on 1.9 now and
    port once (decided 2026-10-11). Pin exact versions; the age gate (§3.8) applies.
  - `@astrojs/solid-js` 7.0.2 (August 2026) requires `solid-js ^1.9.13`; no release supports
    2.0 [43][45], which moves to `@solidjs/web` and `@solidjs/vite-plugin` [43]. Use the
    official integration. For 2.0, wait for an official release that accepts it: no local
    renderer, no community renderer such as `astro-solid-next` [46].
  - Agents bring habits from other frameworks (destructured props, dependency arrays,
    re-render thinking). Give island files a path-scoped agent rule, written from the
    installed major's docs (docs.solidjs.com for 1.x [49]):
    - read props, never destructure them (`splitProps`, `mergeProps`);
    - components run once: no early returns, nothing that expects a re-render;
    - a derived value is a function or `createMemo`, never an effect copying one signal into
      another;
    - `<Show>`, `<For>`, `<Index>`, `<Switch>` instead of conditionals and `.map` in JSX;
    - HTML's DOM names: `class`; `onInput` for each keystroke (`onChange` fires only on
      commit).
  - Every island gets an e2e test that exercises its interaction; these bugs show only at
    runtime.
- Fallback: Preact (10, or 11 once aged) [21][25][26][27], only if Solid stalls: no more 1.x
  fixes, and neither 2.0 nor Astro's support for it arrives. Same toolchain; nothing else
  changes.
- Small behaviour (a reveal button, a toggle, URL sync): a plain TS module or a custom
  element. A page without real widgets ships no framework.
- Animation: the Web Animations API, Baseline since March 2020 [28]. Scripts may set
  `element.style` or call `animate()`; the CSP restricts only markup.

#### Moving to Solid 2.0
Check at every dependency update of a site with Solid islands. Both must hold:
1. `npm view solid-js dist-tags`: `latest` is 2.x, not a release candidate, and
   `npm view solid-js time` shows it is more than 14 days old.
2. `npm view @astrojs/solid-js peerDependencies`: its `latest` accepts that 2.x and is more
   than 14 days old.

Until then, update as usual. Once both hold, stop the routine update; the port is a session of
its own:
1. Read first; training data holds little 2.0 code, and 1.x and React habits look plausible:
   - the migration guide and its topic documents in full [50]; the 2.0 docs' concept pages
     [48] (their `llms.txt` lists every page; search `llms-full.txt`, 1.4 MB, rather than
     read it);
   - the release notes [44]; Astro's integration docs and changelog [45];
   - reports from people who have migrated, the Solid team's first [51].

   Check every import and signature against the installed type definitions, never memory.
2. Port all Solid code, one island at a time, its e2e test passing each time. Checklist
   [50]:
   - `solid-js/web` → `@solidjs/web`; `jsxImportSource` → `@solidjs/web`;
   - `Suspense`, `ErrorBoundary` → `Loading`, `Errored`; `onMount` → `onSettled`;
   - `mergeProps`, `splitProps` → `merge`, `omit`; `Index` → `<For keyed={false}>`;
   - `classList` → `class`; `use:` directives → `ref` factories;
   - `createResource`, `batch`, `produce`, `createMutable`, `on`: removed.

   The community migration assistant [52] may analyse code; it does not write it.
3. Add a flake check that fails on any import or name that exists only in 1.x.
4. Re-evaluate the architecture, not just the syntax [50]. Decide each, and record the
   decisions in this section:
   - Batching: a write is visible after the microtask batch or `flush()`. Affects code and
     tests that read right after writing.
   - Effects: split into a tracked compute and an untracked apply. Check every
     `createEffect`, and whether it should be one.
   - Async: built in (async memos, `Loading`, `isPending`, `action`, optimistic stores).
     Affects saved state and anything that waits.
   - Stores: setters take a draft; `createProjection` replaces `createSelector`. Affects
     state shared between islands.
   - Refs: unowned; `ref` factories replace directives. Affects cleanup and direct DOM code.
   - Compiler: `@solidjs/vite-plugin` uses Oxc instead of Babel. Affects the build,
     `tsconfig.json`, Biome, Vitest, and the CSP hashes of what Astro inlines.
5. Rewrite the island rule from the 2.0 docs, adding the 1.x habits now wrong: reading a
   signal right after setting it, one-function effects, `onMount`, `batch`.
6. Remove the check from the dependency update, and its row in *Revisit when*.

### 3.5 Computation: Rust compiled to WebAssembly
- One crate per engine (evaluator, type checker, solver), compiled to
  `wasm32-unknown-unknown`, JS glue by wasm-bindgen. An island imports it as an ES module and
  only draws the result.
- Built by crane with rust-overlay. crane pins `wasm-bindgen-cli_0_2_<n>`, which "must match
  the one from Cargo.lock" [29]; a flake check compares the two.
- Not wasm-pack: it downloads its own wasm-bindgen at build time, which the Nix sandbox
  forbids. The rustwasm organisation was wound down in 2025; wasm-bindgen has its own
  organisation, wasm-pack an individual maintainer [30].
- Server-side Rust only if logic must be shared: the TS Worker calls a WASM module.
  `workers-rs` is officially supported [31], but TS is shorter for handlers that mostly read
  and write a database.
- A site that ships WebAssembly needs `'wasm-unsafe-eval'` in `script-src` [32].
- Rust and TS only approximate code that needs higher-kinded types. Where widgets depend on
  such encodings, PureScript or Haskell on GHC's WebAssembly backend shows them exactly, at
  the cost of another toolchain.

### 3.6 Styling: Tailwind v4 through the Vite plugin
- The Vite plugin is the official integration for Astro [33]: 11 packages more than no
  Tailwind (measured); the CSS goes through Astro's pipeline, so file names are hashed and
  caches bust automatically.
- Turn off the built-in colours. Each theme defines its colour roles as CSS custom
  properties.

### 3.7 Types, lint, format, tests
- TypeScript 6 for now. TypeScript 7 (the native Go compiler, stable since summer 2026) has no
  programmatic API until 7.1, so Astro's, Vue's and Svelte's checkers and typescript-eslint
  cannot use it [34].
- `tsconfig`: `strict`, `verbatimModuleSyntax`, `erasableSyntaxOnly`. Node runs such files
  directly; type stripping is stable from Node 24.12 [35].
- `astro check` is the one heavy dev-only dependency: 81 packages (measured) for its language
  server. It ships nothing to readers.
- Biome 2 formats and lints TS, JSON and CSS; turn on its experimental `.astro` support [24].
  Not Oxlint + Oxfmt: Oxfmt is 0.x and does not format `.astro` [36]. Not ESLint + Prettier:
  two more dependency trees.
- Tests:
  - Vitest with fast-check for logic; Astro's experimental Container API for components [37];
  - `cargo test` on the native target for crates;
  - Playwright against the built site, browsers from nixpkgs' `playwright-driver`; a check
    keeps npm's Playwright version equal to it.

### 3.8 npm under Nix
- `importNpmLock` builds `node_modules` from the integrity hashes in `package-lock.json`; no
  extra hash to keep in step [38].
- The flake module that builds `node_modules` filters out other platforms' packages before
  fetching.
- `.npmrc`:
  - `package-lock-only=true`, `save-exact=true`;
  - `min-release-age=14` (days). Exception (decided 2026-10-29): in an occasional manual
    update, a version at least 12 days old may be taken if its changelog has been read and it
    is not a major with migration work. Take it by per-command override
    (`npm install --min-release-age=12 <pkg>@<version>`), name each such package in its
    commit, and keep 14 in `.npmrc`;
  - `ignore-scripts=true`, `allow-git=none` [39];
  - `engine-strict=true`.
- pnpm has stronger defaults [40] (install scripts blocked, a minimum release age,
  `trustPolicy: no-downgrade`, `blockExoticSubdeps`), but `fetchPnpmDeps` needs a hash that
  changes with every dependency change [38]. npm's settings above cover most of it; reconsider
  after a supply-chain incident that gets past them.

### 3.9 Measured costs (a measured lock file, non-optional packages)

| Set | Packages |
|---|---|
| `astro` alone | 301 |
| All direct dependencies of that build | 476 |
| Added on its own by `@astrojs/check` | 81 |
| … by Vitest | 15 |
| … by the Tailwind Vite plugin + `tailwindcss` | 11 |
| … by the unified pipeline + remark-math + rehype-katex | 7 |
| … by React (`@astrojs/react`, `react`, `react-dom`, types) | 5 |
| … by Motion | 4 |

Package count is no reason to drop React or the Tailwind plugin: most of their packages come
with Astro. The reason to drop React is the 65 KB runtime every reader downloads.

### 3.10 Hosting and server code
- Workers with static assets; `workers_dev: false`, `preview_urls: false`. `_headers` and
  `_redirects` work as on Pages, except:
  - a slashless directory address gets a 307, not a 308, unless `_redirects` says otherwise;
  - Email Address Obfuscation does not apply.
- No adapter while a site is fully static: Wrangler uploads the `astro build` output.
- Server routes: a few routes, such as an API under `/api/*`, are a plain Worker script beside
  the assets, not Astro routes. The pages stay static assets, code that checks a sign-in stays
  small enough to read whole, and no Astro runtime runs on the server. For many routes, use
  `@astrojs/cloudflare` (Workers only since v13 [5]) with:
  - `imageService: 'compile'`: images made at build time, not by Cloudflare's image binding at
    run time;
  - `session: false` unless sessions are needed; otherwise the adapter creates a KV namespace
    on deploy.

## 4. Security baseline

1. The CSP is an HTTP header in `_headers`. Astro's `security.csp` stays off: it writes a
   `<meta>` tag, which cannot carry `frame-ancestors` [41], and it does not support Shiki
   [42].
2. `build.inlineStylesheets: 'never'` [42]: all CSS comes from files, so `style-src 'self'`
   holds without exceptions.
3. Inline scripts and `<style>` elements are allowed by hash, and the hashes are generated.
   Astro inlines code itself; a measured build had 11 distinct inline scripts (the island
   runtime, small component scripts it chose to inline, the pre-paint theme script) and an
   `astro-island { display: contents }` style on pages with islands. They change with Astro
   updates. A build step hashes every executable inline script and every inline `<style>` in
   the built HTML and writes the hashes into the CSP in `_headers`; a flake check fails on any
   inline script or style the CSP does not cover. JSON data blocks are not executed and need no
   hash.
4. No `style` attributes in the built site, checked by the flake (§3.3). Markdown output goes
   through the styles-to-classes plugin. Components never write `style` in markup: placement
   goes into stylesheets, or into the CSSOM at run time (`element.style`, `animate()`), which
   the CSP does not restrict.
5. `'wasm-unsafe-eval'` only on sites that ship WebAssembly. `'unsafe-eval'` never for a page:
   code that needs it runs in a sandboxed frame under that frame's own, minimal policy.
6. Other headers: HSTS, `X-Content-Type-Options`, `X-Frame-Options`, COOP, CORP, and a
   `Permissions-Policy` that allows only what the site's pages use (a Copy button needs
   `clipboard-write=(self)`; `clipboard-write=()` rejects it, tested in Chromium 155).

## Revisit when

| Trigger | Then |
|---|---|
| A stable `solid-js` 2.x and a stable `@astrojs/solid-js` that accepts it are both past the age gate (checked at every dependency update) | Port all Solid code to 2.0 (§3.4, *Moving to Solid 2.0*), then remove this row and the check |
| Solid stalls: no more 1.x fixes, and no 2.0 or no Astro support for it | Fall back to Preact (§3.4) |
| TypeScript 7.1 ships its API and `astro check` uses it | Move to TypeScript 7 |
| Sätteri gets a first-party maths renderer | Drop the unified pipeline |
| KaTeX's CSS and fonts become a burden | Test MathML-only output with a maths font in Chromium, Firefox and WebKit |
| Oxfmt reaches 1.0 with `.astro` support | Compare it with Biome |
| A supply-chain incident gets past npm's settings | Move to pnpm |

## Sources

1. Astro, *Framework components* (islands, client directives): https://docs.astro.build/en/guides/framework-components/
2. Astro, *Configuration reference* (`output: 'static'` default): https://docs.astro.build/en/reference/configuration-reference/
3. State of JS 2025, *Meta-frameworks*: https://2025.stateofjs.com/en-US/libraries/meta-frameworks/
4. Business Wire, *Cloudflare Acquires Astro* (2026-01-16): https://secure.businesswire.com/news/home/20260116386991/en/Cloudflare-Acquires-Astro-to-Accelerate-the-Future-of-High-Performance-Web-Development
5. Astro, *@astrojs/cloudflare*: https://docs.astro.build/en/guides/integrations-guide/cloudflare/
6. Astro blog, *Astro 7*: https://astro.build/blog/astro-7/
7. InfoQ, *Astro's Sätteri* (2026-08): https://infoq.com/news/2026/08/astro-satteri-rust
8. Hackage, *hakyll*: https://hackage.haskell.org/package/hakyll
9. Hugo, *js.Build*: https://gohugo.io/functions/js/build/
10. Hugo, *transform.ToMath*: https://gohugo.io/functions/transform/tomath/
11. Hugo, *css.TailwindCSS*: https://gohugo.io/functions/css/tailwindcss/
12. Eleventy, *Core Dependency Watch (2025 Edition)*: https://www.11ty.dev/blog/dependency-watch
13. Astro, *Markdown in Astro*: https://docs.astro.build/en/guides/markdown-content/
14. Astro, *Sätteri API reference*: https://docs.astro.build/en/reference/markdown-processors/satteri/
15. Third-party report on Sätteri maths: https://dev.to/ashish-codejourney/astro-7-renders-your-display-maths-as-a-code-block-and-wont-tell-you-cp0
16. KaTeX, *Options* (`output`): https://katex.org/docs/options
17. MDN, *MathML*: https://developer.mozilla.org/en-US/docs/Web/MathML
18. Temml: https://temml.org/
19. Astro, *Syntax highlighting*: https://docs.astro.build/en/guides/syntax-highlighting/
20. Shiki, *Transformers*: https://shiki.style/packages/transformers
21. Preact: https://preactjs.com/
22. State of JS 2025, *Front-end frameworks*: https://2025.stateofjs.com/en-US/libraries/front-end-frameworks/
23. InfoQ, *SolidJS 2.0* (2026-05): https://www.infoq.com/news/2026/05/solidjs-2-async/
24. Biome, *Language support*: https://biomejs.dev/internals/language-support/
25. Astro, *@astrojs/preact*: https://docs.astro.build/en/guides/integrations-guide/preact/
26. Preact releases: https://github.com/preactjs/preact/releases
27. Preact, *Upgrading to v11*: https://preactjs.com/guide/v11/upgrade-guide
28. MDN, *Element.animate()*: https://developer.mozilla.org/en-US/docs/Web/API/Element/animate
29. crane, *Trunk example* (wasm-bindgen pinning): https://crane.dev/examples/trunk.html
30. Rust blog, *Sunsetting the rustwasm GitHub org* (2025-07-21): https://blog.rust-lang.org/inside-rust/2025/07/21/sunsetting-the-rustwasm-github-org
31. Cloudflare, *Rust on Workers*: https://developers.cloudflare.com/workers/languages/rust/
32. MDN, *CSP script-src* (`'wasm-unsafe-eval'`): https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/script-src
33. Tailwind, *Install with Astro*: https://tailwindcss.com/docs/installation/framework-guides/astro
34. InfoQ, *TypeScript 7 released* (2026-08): https://infoq.com/news/2026/08/typescript-7-released/
35. Node.js, *Modules: TypeScript*: https://nodejs.org/api/typescript.html
36. Oxc, *Compatibility*: https://oxc.rs/compatibility, and *Oxlint v1.0*: https://oxc.rs/blog/2025-06-10-oxlint-stable
37. Astro, *Testing*: https://docs.astro.build/en/guides/testing/
38. nixpkgs manual, *JavaScript*: https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/javascript.section.md
39. npm, *config* (v11): https://docs.npmjs.com/cli/v11/using-npm/config
40. pnpm, *Supply chain security*: https://pnpm.io/supply-chain-security
41. MDN, *CSP frame-ancestors*: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/frame-ancestors
42. Astro, *Configuration reference* (`security.csp`, `build.inlineStylesheets`), as [2]
43. npm registry, checked 2026-10-11: `solid-js` (latest 1.9.17 of 7 October, so under the age gate npm resolves 1.9.15 until 1.9.16 ages on 20 October; next 2.0.0-rc.14), `@solidjs/web` (2.0 release candidates only), `@astrojs/solid-js` 7.0.2 (peer `solid-js ^1.9.13`; no tag supports 2.0), `vite-plugin-solid` 3.0.0-next (depends on `@solidjs/vite-plugin`): https://registry.npmjs.org/solid-js
44. Solid releases: https://github.com/solidjs/solid/releases
45. Astro, *@astrojs/solid-js*: https://docs.astro.build/en/guides/integrations-guide/solid-js/
46. astro-solid-next, a community Astro renderer for Solid 2.0: https://github.com/mikepage/astro-solid-next
47. MDN, *CSP style-src-attr*: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/style-src-attr
48. Solid 2.0 docs (release candidate), with `llms.txt` and `llms-full.txt`: https://v2.solidjs.com/
49. Solid docs (1.x), with `llms.txt`: https://docs.solidjs.com/
50. Solid, *Solid 2.0 — quick migration guide* (`MIGRATION.md`) and the twelve topic documents beside it: https://github.com/solidjs/solid/tree/next/documentation/solid-2.0
51. Solid blog, *Solid 2.0 RC: The Big \<Reveal\>*: https://www.solidjs.com/blog/solid-2-0-rc-the-big-reveal
52. solid-migration-assistant (community, experimental; analysis and a few import moves): https://github.com/solidjs-community/solid-migration-assistant
