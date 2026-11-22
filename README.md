# agda-wasm-build

Automated [WebAssembly](https://webassembly.org/) builds of [Agda](https://github.com/agda/agda).

Each release (`agda-wasm-vX.Y.Z`) contains:

- `agda.wasm` — the Agda compiler compiled to `wasm32-wasi`, built with GHC's
  wasm backend (`-O2 --enable-split-sections -foptimise-heavily`) and shrunk
  with Binaryen (`wasm-opt -Oz --strip-debug`)
- `SHA256SUMS.txt`

## How it's built

The workflow checks out the corresponding tag of `agda/agda`, enters the Nix
dev shell defined by that tag's own `flake.nix` — so the GHC wasm toolchain,
`wasm32-wasi-cabal` and `wasmtime` versions are exactly the ones pinned by
upstream's `flake.lock` (same as the upstream `[wasm]` CI) — then builds
`wasm32-wasi-cabal build -O2 --enable-split-sections -foptimise-heavily`
and runs `wasm-opt -Oz --strip-debug` on the result.

## Triggers

- **Weekly schedule** (Mondays 04:23 UTC): checks every upstream stable
  release with version >= v2.9.0 (the first version with WASM support) and
  builds all the ones that don't have a matching release here yet.
- **Manual**: `gh workflow run build.yml -f ref=vX.Y.Z` — `ref` can be any
  `agda/agda` tag, branch or commit; empty means "every missing upstream
  release >= v2.9.0". Existing releases are never rebuilt.
