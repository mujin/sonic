# Mujin fork of sonic

Projects use this fork through a `replace` directive in their `go.mod`.
The Mujin commits on top of upstream change what goes over the wire:

- Floats always carry a decimal point (`1` encodes as `1.0`, zero as `0.0`).
- With `UseInt64`, integers decoded into `interface{}` become `int64`, or
  `uint64` above the `int64` range, and `float64` only when neither fits.
- `Config.ReplaceNulls` replaces NUL with U+FFFD when encoding and U+FFFD
  with NUL when decoding.

## Branches

- `mujin`: the fork on upstream v1.13.2 (Go 1.25 and older).
- `mujin-v1.15.5`: the same commits on upstream v1.15.5 (Go 1.17 to 1.27).

## Rebasing onto a new upstream release

1. Branch from the upstream tag and cherry-pick the Mujin commits with `-x`.
   Upstream runs `go fmt` now and then, so conflicts are often whitespace
   only. If you re-apply a hunk with `patch -l` or fuzz, check where it
   landed. A misplaced flag setup in `internal/decoder/jitdec` breaks every
   escaped string. Compare each file's Mujin hunks with context against the
   previous branch:

   ```bash
   git diff -w -U5 <old-base> <old-branch> -- <file>
   git diff -w -U5 <new-base> <new-branch> -- <file>
   ```

2. Take upstream's copy of the generated files under `internal/native/sse`
   and `internal/native/avx2` while cherry-picking, then regenerate them once
   at the end. Upstream hand-edits some template outputs
   (`native_export.go`, `recover_test.go`), so keep upstream's copies of
   those unless a Mujin commit changes their template.

3. Regenerate the assembly with clang 13. This toolchain reproduces the
   `mujin` branch's checked-in assembly byte for byte:

   ```dockerfile
   FROM ubuntu:22.04
   RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends clang-13 libc6-dev python3 python3-pip git ca-certificates && rm -rf /var/lib/apt/lists/*
   RUN pip3 install --no-cache-dir git+https://github.com/Maratyszcza/PeachPy
   ```

   ```bash
   git submodule update --init tools/asm2asm
   docker run --rm -u $(id -u):$(id -g) -v $PWD:/src -w /src sonic-clang13 \
       bash scripts/build-x86.sh clang-13 "-I/usr/include/x86_64-linux-gnu -U__nonnull"
   ```

   The script compiles for an Apple target, where clang predefines
   `__nonnull` and breaks glibc headers, hence `-U__nonnull`.

4. Run the tests on every Go version the fork is used with (`go test ./...`,
   then `issue_test` and `generic_test`). Upstream tests added since the last
   rebase may expect `0` where this fork writes `0.0`.

5. Compare the output with the previous branch before switching to the new
   commit. Encode and decode large JSON documents, random floats and
   integers, and strings with NUL and U+FFFD.
