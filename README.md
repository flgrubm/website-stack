# Stack specification for static websites

The tech stack I use for my static websites, written down for the LLM coding agents that build
them. It's my opinion, not advice for anyone else.

In short: Astro and TypeScript, Tailwind CSS v4, Solid islands where a page needs interactivity,
Rust compiled to WebAssembly for heavy computation, npm dependencies installed by Nix flakes,
hosting on Cloudflare Workers, a strict Content-Security-Policy, and Jujutsu for version control.

- [`stack-spec.md`](stack-spec.md) is the spec: each choice, why, what it was chosen over, and
  sources.
- [`AGENTS.md`](AGENTS.md) and [`llms.txt`](llms.txt) tell coding agents how to use it.
- The `update-stack-spec` skill (`/update-stack-spec` in Claude Code) brings it up to date when
  a tool in it gets a major release.
