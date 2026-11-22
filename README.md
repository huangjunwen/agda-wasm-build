# agda-wasm-build

Automated [WebAssembly](https://webassembly.org/) builds of the [Agda compiler](https://github.com/agda/agda).

Each release (`agda-wasm-vX.Y.Z`) contains:

- `agda.wasm` — the Agda compiler compiled to `wasm32-wasi`, built with GHC's
  wasm backend (`-O2 --enable-split-sections -foptimise-heavily`) and shrunk
  with Binaryen (`wasm-opt -Oz --strip-debug`)
- `SHA256SUMS.txt`

## Running

Any WASI runtime works, e.g. [wasmtime](https://wasmtime.dev/). The first
launch compiles the ~34 MB module with Cranelift, which can take a couple of
minutes; enable wasmtime's compilation cache so later launches start in
about a second:

```sh
wasmtime config new        # one-time, writes ~/.config/wasmtime/config.toml
```

Verified commands (always pass `-C cache` after the one-time setup above;
`--dir /::/` maps the host filesystem to the same paths inside the guest,
which also stops Agda's search for a project root at `/`):

```sh
# version
wasmtime run -C cache --dir /::/ --env HOME="$HOME" agda.wasm --version

# extract the built-in library (Agda_datadir must be an existing directory)
mkdir -p agda-data
wasmtime run -C cache --dir /::/ --env HOME="$HOME" \
    --env Agda_datadir="$PWD/agda-data" agda.wasm --setup

# type-check a module (include dirs use lowercase -i)
wasmtime run -C cache --dir /::/ --env HOME="$HOME" --env PWD="$PWD" \
    --env Agda_datadir="$PWD/agda-data" agda.wasm -i . Main.agda
```

Notes:

- `HOME` is mandatory: without it Agda falls back to probing the effective
  UID, which fails under WASI.
- `--interaction` additionally needs a runtime that never blocks on stdin
  plus the RTS option `+RTS -V1 -RTS` (the default RTS tick value thrashes
  on WASM).
- On WASM, Agda cannot spawn external processes (no LaTeX compilation, no
  external programs from `--interaction`).

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
