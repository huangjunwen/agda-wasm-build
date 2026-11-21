# agda-wasm-build

Automated [WebAssembly](https://webassembly.org/) builds of the [Agda compiler](https://github.com/agda/agda).

Each release (`agda-wasm-vX.Y.Z`) contains:

- `agda.wasm` — the Agda compiler compiled to `wasm32-wasi`, built with GHC's
  wasm backend (`-foptimise-heavily`) and shrunk with Binaryen
  (`wasm-opt -Os --strip-debug`)
- `SHA256SUMS.txt`

## Running

Any WASI runtime works, e.g. [wasmtime](https://wasmtime.dev/):

```sh
wasmtime run --env "PATH=$PATH" --env "PWD=$PWD" --env "HOME=$HOME" \
    --dir / --argv0 agda agda.wasm --version
```

Note: on WASM, Agda cannot spawn external processes (no LaTeX compilation,
no external programs from `--interaction`).

## How it's built

The workflow checks out the corresponding tag of `agda/agda`, enters the Nix
dev shell defined by that tag's own `flake.nix` — so the GHC wasm toolchain,
`wasm32-wasi-cabal` and `wasmtime` versions are exactly the ones pinned by
upstream's `flake.lock` (same as the upstream `[wasm]` CI) — then builds
`wasm32-wasi-cabal build -foptimise-heavily` and runs `wasm-opt` on the result.

## Triggers

- **Weekly schedule** (Mondays 04:23 UTC): checks every upstream stable
  release with version >= v2.9.0 (the first version with WASM support) and
  builds all the ones that don't have a matching release here yet.
- **Manual**: `gh workflow run build.yml -f ref=vX.Y.Z` — `ref` can be any
  `agda/agda` tag, branch or commit; empty means "every missing upstream
  release >= v2.9.0". Existing releases are never rebuilt.
