# Source build on Apple Silicon through Docker

## Inputs and scope

This experiment builds the `lilypond` target from the WASI port's release
`v0.1.0-alpha.1`, rather than downloading its prebuilt engine.
The LilyPond source is pinned by that release's `flake.lock`; it is not the
current C++ checkout at the root of this fork.

- Port revision: `8cca1c156c4364371285af27fa22459cd92a8fde`
- LilyPond revision: `9c1e4dc760f6de0e4fdb762f6d5b124baa0f3a17` (2.27.2)
- nixpkgs revision: `d3498f786f97ac0bded21b34bae0bf3809b45aa3`
- Host: macOS arm64; build container: Linux aarch64
- Nix image: `nixos/nix@sha256:7a007c766426c1877758ddc5cb87a965ac131fc78c582ce0083d922d51ae945c`

The complete source archive was downloaded and its SHA-256 matched
`ec09aec6382e3db1355d02b3354ae81705d3100fad9fd1cf111ea8d8ed4650f7`.
The release checkout's `flake.nix`, `flake.lock`, and LilyPond derivation were
compared with those in the complete source bundle and matched byte for byte.

## Commands

Run from this fork's root. Docker Desktop must be running. Generated inputs,
logs, and binaries belong in the ignored `wasm/build/` directory.

```sh
mkdir -p wasm/build/source-build/project
curl -fsSL https://api.github.com/repos/hlolli/lilypond-wasm/tarball/v0.1.0-alpha.1 -o wasm/build/source-build/project.tar.gz
tar -xzf wasm/build/source-build/project.tar.gz --strip-components=1 -C wasm/build/source-build/project

docker pull nixos/nix@sha256:7a007c766426c1877758ddc5cb87a965ac131fc78c582ce0083d922d51ae945c
docker run -d --name tetorica-lilypond-source-build \
  --mount "type=bind,source=$PWD/wasm/build/source-build,target=/work" \
  --workdir /work \
  nixos/nix@sha256:7a007c766426c1877758ddc5cb87a965ac131fc78c582ce0083d922d51ae945c \
  sleep infinity

docker exec tetorica-lilypond-source-build \
  nix --extra-experimental-features 'nix-command flakes' build \
  path:/work/project#lilypond --no-update-lock-file \
  --option sandbox false --max-jobs 2 --cores 4 \
  --print-build-logs --out-link /work/result-lilypond
```

Nix's nested sandbox is disabled inside this dedicated container. The container
shares only the build directory; no Docker socket is mounted inside it.
Compilers and some dependencies come from the Nix binary cache. This is a source
build of the engine, not a bootstrap of every compiler and tool.
The initial evaluation selected 27 derivations to build and about 669 MiB of
cache downloads (2.6 GiB unpacked).

`result-lilypond` points into the container's `/nix/store`; copy the engine out
from inside the container rather than following that symlink on macOS:

```sh
if [ -f wasm/build/source-build/lilypond-source-built.wasm ]; then
  chmod u+w wasm/build/source-build/lilypond-source-built.wasm
fi
docker exec tetorica-lilypond-source-build \
  cp /work/result-lilypond/bin/lilypond.wasm /work/lilypond-source-built.wasm
chmod u+w wasm/build/source-build/lilypond-source-built.wasm
LILYPOND_WASM_PATH=wasm/build/source-build/lilypond-source-built.wasm \
  node wasm/scripts/check_lilypond_wasm.mjs
```

Nix store files are read-only. The initial copy can inherit that mode, causing
later copies through Docker Desktop's bind mount to fail with `Operation not
permitted`. The host-side `chmod` above makes only the exported copy writable;
it does not modify the Nix store.

This check reuses the preserved alpha.1 runtime pack and browser worker. It does
not rebuild fonts/runtime data or validate a complete rebuilt npm distribution.
The distinct `lilypond-npm` target also assembles runtime data; `lilypond-npm-tarball`
packages it with corresponding-source information.

## Call-stack optimization

Apply the same Binaryen 132 postprocessing used by the preserved Safari demo:

```sh
/path/to/binaryen-version_132/bin/wasm-opt \
  wasm/build/source-build/lilypond-source-built.wasm \
  --coalesce-locals --vacuum \
  -o wasm/build/source-build/lilypond-source-built-coalesced.wasm
LILYPOND_WASM_PATH=wasm/build/source-build/lilypond-source-built-coalesced.wasm \
  node --liftoff-only wasm/scripts/check_lilypond_wasm.mjs /path/to/score.ly
```

Do not use the fixed-input optimizer script for this engine: it deliberately
requires the original npm binary's hash. Do not add `--all-features`.

## Result

The source build completed successfully on 2026-09-16 (Nix 2.35.2, exit 0).
The build log records compilation of the LilyPond derivation, not substitution
of a cached engine. The unmodified build output rendered the four-note smoke
score successfully in Node (exit 0, one SVG page).
The coalesced source-built engine also rendered the user's full nine-staff
Jungle score under Node `--liftoff-only` (exit 0, eight SVG pages). The six
preview/worker lifecycle tests passed. The user's score was not copied into
this repository.

| Engine | Bytes | SHA-256 |
|---|---:|---|
| Source-built | 61,233,871 | `a3df5ef22b1d2ce5ae0b089d90fc4883cbabc3ea02dbe4ca7dba5bc921747785` |
| Source-built, coalesced | 60,148,735 | `59b346682e4e96c75458992883522cc0427751756280f0e6d36917dc0907d986` |

The coalesced engine is now copied to `wasm/web/vendor/lilypond/dist/lilypond.wasm`
for the clone-and-run demo, alongside the preserved runtime pack. Working
binaries and the build log remain under ignored `wasm/build/source-build/`.
Browser testing of this engine is separate from the earlier user-confirmed
Safari result.
