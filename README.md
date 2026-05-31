<p align="center">
  <img src="https://raw.githubusercontent.com/nasty-project/nasty/main/webui/src/lib/assets/nasty-white.svg" width="300" alt="NASty" />
</p>

<p align="center">
  <strong>Reference documentation for the <a href="https://github.com/nasty-project/nasty">NASty</a> NAS appliance.</strong>
</p>

---

This repository holds the long-form docs for [nasty-project/nasty](https://github.com/nasty-project/nasty). For the project itself — features, installation, screenshots, contributing — start there.

## What's here

[`api.md`](api.md) — the **JSON-RPC method reference**. Every RPC the engine accepts, with role, params, return type, and any related object schemas. Regenerated nightly from the engine's `--dump-docs` output (the engine is the source of truth).

## Interactive API browser

For a searchable, browsable, *try-it-out* version of `api.md`, the same OpenAPI spec is served as a static Swagger UI at:

**[nasty-api.pages.dev](https://nasty-api.pages.dev)**

Updated on the same nightly cadence as `api.md` from this repo. On a running NASty box the engine serves the identical page at `/api/docs` (authenticated against your session, so "Try it out" actually works against your own RPC surface) and the raw spec at `/api/openapi.json`.

## How the docs stay in sync

`api.md` is *generated*, not hand-written. The regen workflow:

1. Checks out [`nasty-project/nasty`](https://github.com/nasty-project/nasty) `main`
2. Runs `nasty-engine --dump-docs` against the workspace
3. Copies the resulting `api.md` into this repo
4. Commits if anything changed (commit message includes the source-repo SHA)
5. Bundles the matching `openapi.json` + vendored Swagger UI into a static site and deploys it to `nasty-api.pages.dev`

Runs once a day at 04:00 UTC and on manual workflow_dispatch. So docs are at most ~24h behind `main`. See [`.github/workflows/regen-api-md.yml`](.github/workflows/regen-api-md.yml) for the implementation.

Spotted drift? File an issue or PR against [`nasty-project/nasty`](https://github.com/nasty-project/nasty) — the registry lives in the engine, this repo just publishes.

## License

GPLv3
