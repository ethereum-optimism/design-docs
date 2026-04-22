# Kona Prestate Build Refactor: Native + Docker Paths

| | |
|---|---|
| **Author(s)** | Adrian Sutton |
| **Status** | Draft |
| **Created** | 2026-04-02 |

## Purpose

Refactor the kona prestate build so that a single definition of the build process works both inside Docker (reproducible) and natively (non-Docker). This enables faster local development iteration while preserving reproducible builds for releases.

## Summary

The kona prestate build is currently embedded in Docker multi-stage builds. Three Docker stages build the cannon binary (Go), cross-compile kona-client to a MIPS64 ELF (Rust nightly), and run `cannon load-elf` + `cannon run` to produce prestate artifacts. This means Docker is required for any prestate build, even local development.

This design extracts the build logic into `just` targets in `rust/justfile` and makes the Dockerfile a thin environment wrapper that calls those same targets. The `cannon-builder` Docker image is reduced to only the apt-level MIPS64 cross-compilation packages (the only component that cannot be deterministically version-pinned outside of Docker). All other tools (Go, Rust, just, jq) are installed via `mise` from the shared `mise.toml`, eliminating version duplication between Docker and the host environment.

The result is:
- **Reproducible Docker path:** `just reproducible-prestate-kona` works exactly as before, producing identical prestate hashes.
- **Native path:** `cd rust && just build-kona-prestates` builds prestates directly on a Linux host with the MIPS64 cross-toolchain installed.
- **Single definition:** Build commands exist in one place (`rust/justfile`), called by both paths.

## Problem Statement + Context

### Current Architecture

The kona prestate build uses a multi-stage Dockerfile (`cannon-repro.dockerfile`) as both the build environment and the build system:

1. **Stage 1** (`golang:alpine`): Builds the cannon binary from Go source.
2. **Stage 2** (`cannon-builder:v1.0.0`): Cross-compiles kona-client to a MIPS64 ELF using Rust nightly with a custom target spec.
3. **Stage 3** (`ubuntu:22.04`): Runs `cannon load-elf` and `cannon run` to produce `prestate.bin.gz` and `prestate-proof.json`.
4. **Stage 4** (`scratch`): Exports artifacts.

The `cannon-builder:v1.0.0` image bundles two orthogonal concerns:
- **MIPS64 cross-compilation apt packages** (`g++-mips64-linux-gnuabi64`, `libc6-dev-mips64-cross`, etc.) which are hard to pin deterministically.
- **A complete Rust nightly toolchain** with cross-compilation environment variables, which is already version-pinned in `rust/justfile` as `NIGHTLY`.

Tool versions are duplicated and have already drifted: Go is `1.24.10` in the Dockerfile but `1.24.13` in `mise.toml`; Rust nightly is hardcoded independently in both `cannon.dockerfile` and `rust/justfile`.

### Why This Matters

- **No native build path.** Building prestates requires Docker even for local development iteration. Every source change triggers Docker layer rebuilds.
- **Version drift.** Tool versions are defined in multiple places (Dockerfile, `mise.toml`, `rust/justfile`) with no mechanism to keep them in sync.
- **Build logic is opaque.** The actual compilation commands are spread across Dockerfile `RUN` directives rather than composable build targets.

### Prior Art: op-program

The op-program prestate build (`op-program/Dockerfile.repro` + `op-program/repro.justfile`) already follows the pattern this design adopts: build logic in a justfile, Docker as a thin environment wrapper calling `just -f repro.justfile build-${EXPORT_TARGET}`.

## Proposed Solution

### Architecture

```
just reproducible-prestate-kona                          # top-level (unchanged)
  └── cd rust && just build-kona-reproducible-prestate   # Docker wrapper
        └── docker build ... -f cannon-repro.dockerfile  # thin Dockerfile
              └── just build-kona-prestate $VARIANT ...  # actual build logic

cd rust && just build-kona-prestates                     # native path (new)
  ├── just build-kona-prestate kona-client ...           # actual build logic
  └── just build-kona-prestate kona-client-int ...       # same targets
```

Both paths execute the same `just` targets. The Docker path provides a fixed environment; the native path uses locally installed tools.

### Build Logic in `rust/justfile`

Three targets encapsulate the build:

1. **`build-kona-client-elf VARIANT`** — Sets up the MIPS64 cross-compilation environment (env vars, nightly toolchain) and runs `cargo build` with `-Zbuild-std=core,alloc` and the custom target spec. Supports custom chain configs via the `KONA_CUSTOM_CONFIGS_DIR` environment variable.

2. **`build-kona-prestate VARIANT OUTPUT_DIR`** — Orchestrates the full pipeline: builds the cannon binary (via `cannon/justfile`), calls `build-kona-client-elf`, then runs `cannon load-elf` and `cannon run` to produce prestate artifacts.

3. **`build-kona-reproducible-prestate`** — Docker wrapper that runs `docker build` with the thin Dockerfile for each variant (kona-client, kona-client-int), producing reproducible artifacts.

Cross-compilation environment variables (`CC_mips64_unknown_none`, `CARGO_BUILD_TARGET`, `RUSTFLAGS`, etc.) are defined once in `build-kona-client-elf`. The `MIPS64_TARGET_SPEC` variable points to the existing `mips64-unknown-none.json` target spec file. The `NIGHTLY` variable (already in `rust/justfile`) is the single source of truth for the Rust nightly version.

### Thin Dockerfile

The new `cannon-repro.dockerfile` replaces the multi-stage build:

```
FROM cannon-builder:v2.0.0          # apt packages only (frozen)
  ├── COPY ops/scripts/install_mise.sh → install mise (version-pinned + checksum-verified)
  ├── COPY mise.toml → mise install  # Go, Rust stable, just, jq (pinned)
  ├── COPY rust/justfile → just install-nightly  # Rust nightly (pinned)
  ├── COPY source (minimal set)
  └── just build-kona-prestate $VARIANT ...
FROM scratch                        # export artifacts
```

Layers are ordered for cache efficiency: mise installation and tool versions change rarely, nightly pin changes occasionally, source changes frequently.

Mise is installed using the monorepo's `ops/scripts/install_mise.sh`, which pins a specific mise version (`v2025.3.2`) with hardcoded checksums for all platforms. This is the same script used by CI, ensuring the Docker build uses an identical mise version.

Only the tools needed for the build (Go, Rust, just, jq) are installed from `mise.toml`. The full `mise.toml` includes many other tools (foundry, pipx packages, etc.) that would bloat the image and break hermetic builds. The Dockerfile creates a filtered minimal `mise.toml` at build time.

Docker cache mounts (`--mount=type=cache`) are used for the cargo registry, cargo target directory, Go module cache, and Go build cache.

### Reduced `cannon-builder` Image

The `cannon-builder` image is reduced to contain only apt packages:

```dockerfile
FROM ubuntu:22.04
RUN apt-get install ... \
  g++-mips64-linux-gnuabi64 \
  libc6-dev-mips64-cross \
  binutils-mips64-linux-gnuabi64 \
  llvm clang build-essential cmake git curl ca-certificates
```

Rust, the target spec, and all environment variables are removed. These are now installed on top via mise and the justfile targets. The image is published as `cannon-builder:v2.0.0`.

This separation is deliberate: apt packages are the only component where versions cannot be deterministically pinned (package repositories evolve over time). Freezing them in a Docker image is the standard approach. Everything else — Go, Rust, just, jq — has deterministic version pinning through mise and rustup.

### Reproducibility Model

| Component | Pinning Mechanism | Where Defined |
|-----------|-------------------|---------------|
| MIPS64 cross-toolchain (apt) | Frozen Docker image | `cannon-builder:v2.0.0` |
| Go | mise version pin | `mise.toml` |
| Rust stable | mise version pin | `mise.toml` |
| Rust nightly | rustup version pin | `rust/justfile` (`NIGHTLY`) |
| just, jq | mise version pin | `mise.toml` |
| Source code | git commit | Deterministic |
| Build commands | justfile targets | `rust/justfile` |

The Docker path (`just reproducible-prestate-kona`) is the canonical path for reproducible builds. It produces identical prestate hashes for the same source code because all inputs are pinned. The native path does not guarantee reproducibility (different host toolchain versions may produce different binaries) but produces functionally equivalent prestates.

### Native Path Dependencies

Tools managed by mise (`mise.toml`): Go, Rust (stable), just, jq.

Installed by the justfile: Rust nightly (via `rustup toolchain install $NIGHTLY` with `rust-src` component).

Manual installation required:
- **Linux (Ubuntu/Debian):** `sudo apt install g++-mips64-linux-gnuabi64 libc6-dev-mips64-cross binutils-mips64-linux-gnuabi64`
- **macOS:** Docker is the recommended path. Native builds may work if a MIPS64 cross-toolchain is available via Homebrew, but this is not a primary supported configuration.

### CI Impact

All CI jobs invoke `just` targets rather than calling `docker build` directly. The CircleCI `kona-publish-prestate-artifacts` job currently calls `cd docker/fpvm-prestates && just cannon ...` which delegates to `docker buildx bake`. This job is updated to call the appropriate `just` targets from `rust/justfile` and `rust/kona/justfile`, which internally handle Docker invocation. This keeps CI configuration simple (just call `just`) and ensures CI always exercises the same code paths as local development.

### Additional Updates

- **`lint-cannon` and `build-cannon-client`** in `rust/kona/justfile` continue to run inside Docker, but use the same environment as the reproducible prestate build rather than `docker run cannon-builder:v1.0.0`. This ensures they validate that kona-client compiles and lints correctly in the exact environment used for releases, and avoids requiring CI machines to have the MIPS64 cross-toolchain installed natively. `kona-build-fpvm-cannon-client` is a required CI gate that blocks merge, so it must keep working. The approach is to build a local Docker image from the thin Dockerfile's builder stage and run clippy/build inside it.
- **`install-nightly`** in `rust/justfile` is updated to include the `rust-src` component (required for `-Zbuild-std`).
- **`docker-bake.hcl`**: The `kona-cannon-prestate` bake target and its associated variables are removed (replaced by direct `docker build`). The `cannon-builder` bake target is retained for publishing the apt-only image.
- **`.dockerignore`** at repo root is updated to exclude `rust/target/` and other large directories, since the build context changes from `rust/` to the monorepo root.

## Resource Usage

Build time is expected to be comparable to the current Docker build. The main change is structural (where commands run), not computational. Docker cache mounts should improve rebuild times for iterative development.

The native path avoids Docker overhead entirely, which should be faster for local development.

## Single Point of Failure and Multi Client Considerations

This change is internal to the optimism monorepo build system. It does not affect the protocol, on-chain contracts, or the behavior of any client. The prestate artifacts produced are identical to those produced today.

The only external dependency is the `cannon-builder` Docker image published to the OP Labs artifact registry. If this image becomes unavailable, the Docker reproducible build path would fail. However, the native build path would still work (it does not depend on the image).

## Alternatives Considered

### Separate `repro.justfile` (op-program pattern)

A dedicated `repro.justfile` for the kona prestate build, separate from `rust/justfile`. This is the exact pattern op-program uses. However, the kona prestate targets already exist in `rust/justfile` with the `NIGHTLY` variable and other shared configuration. A separate file would duplicate these or require additional import plumbing. Since the targets fit naturally in the existing file, a separate file adds complexity without benefit.

### Keep Docker Multi-Stage Build, Add Parallel Native Targets

Maintain the current Dockerfile and add separate native-only targets in `rust/justfile`. This avoids changing the Docker build but violates the single-definition principle — build commands would be defined in both the Dockerfile and the justfile, inevitably drifting apart.

### Pin Apt Package Versions Explicitly

Instead of freezing apt packages in a Docker image, pin exact versions in the `apt-get install` command (e.g., `g++-mips64-linux-gnuabi64=4:11.2.0-*`). This is brittle because Ubuntu removes old package versions from repositories over time, causing builds to fail when the pinned version is no longer available.

### Use `rust-lld` as Sole Linker (Remove GCC Linker Dependency)

The MIPS64 target spec declares `"linker": "rust-lld"`, but the `CARGO_TARGET_MIPS64_UNKNOWN_NONE_LINKER` environment variable overrides this to use GCC. Since kona-client is pure Rust `no_std` with no `-sys` crate dependencies, `rust-lld` alone may suffice. If so, the apt cross-toolchain packages become irrelevant to the output binary and reproducibility depends solely on the Rust nightly version. This is worth experimenting with during implementation but is not part of the initial design to avoid risk.

## Risks & Uncertainties

- **Prestate hash stability.** The primary risk is that the refactored Docker build produces different prestate hashes than the current build. This must be verified by building with both the old and new Dockerfiles and comparing hashes. The mitigation is that the build commands are identical — only the environment setup changes.

- **Workspace resolution in Docker.** The Rust workspace `Cargo.toml` references all workspace members (including `op-reth`). Cargo requires all members to be present even when building a single package with `-p`. The Docker build must copy enough of the workspace for resolution to succeed. The initial approach copies all workspace member directories; a follow-up could create stub `Cargo.toml` files for unneeded members to reduce context size.

- **mise availability in Docker.** The Dockerfile installs mise using `ops/scripts/install_mise.sh`, which downloads from a pinned GitHub release with checksum verification. If GitHub releases are unavailable during a Docker build, it will fail. Mitigation: Docker layer caching means this only runs when the layer is invalidated (rarely).

- **GCC linker and reproducibility.** The `CARGO_TARGET_MIPS64_UNKNOWN_NONE_LINKER` env var causes GCC to be used as the linker. If apt package versions change and GCC behavior differs, the output binary could change. This risk exists today (the cannon-builder image was built once and frozen) and is mitigated by the same approach: freezing the apt packages in a Docker image. The `rust-lld` experiment (see Alternatives) could eliminate this risk entirely.

- **CI job compatibility.** The CircleCI `kona-publish-prestate-artifacts` job must be updated to use the new build path. The job currently invokes the `fpvm-prestates/justfile` which is being deleted. The update is straightforward (call `docker build` directly) but must be coordinated with the code change.
