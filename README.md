# Nub

Nub is a standalone, personality-generic execution engine for RISC-V: a KVM/Hyperlight sandbox, an interpreter, and an x86-64 JIT recompiler behind one uniform handle. It executes standard RISC-V bytecode (RV64E+C plus the Zbb/Zba/Zbs/Zicond extensions and Xjar) under deterministic gas metering, but it deliberately knows nothing about what the programs *mean* — no capability model, no state semantics, no kernel policy. That semantics layer is a **personality** that plugs into two static-dispatch seams: the host-side [`Personality`](nub/nub/src/personality.rs) trait (what published objects mean, how invocations resolve a root object) and the guest-side [`GuestPersonality`](nub/nub-arch-x86/src/personality.rs) trait (ecall dispatch, state store, gas sourcing). JAVM's capability system — in the `rust/` tree of [alleyos/alley](https://github.com/alleyos/alley) — is the first personality.

The caller-facing surface is `Nub<P: Personality>`, which hides the choice of substrate behind a single publish/invoke API. The wire protocol is personality-agnostic on purpose: the host ships opaque byte blobs into the engine and addresses them by 32-byte content hashes (`ObjHash`); the personality decides how to decode, validate, and hash them. Two backends implement that surface:

- **Local** — the personality's in-process kernel (a `LocalKernel` impl driving the `nub-arch-local` interpreter substrate). No sandbox, no cross-compilation; used for tests, deterministic replay, and any host that doesn't need real isolation.
- **Hyperlight** — invocations ship as RPCs into a bare-metal guest kernel (the personality's guest binary over the generic `nub-arch-x86` lib) running ring-0 inside a KVM hardware-virtualization sandbox, driven by `nub-host-kvm`. The JIT runs *inside* the guest: untrusted PVM2 bytecode compiles to native x86-64 and executes without leaving the VM, so hostcall round-trips are the exception rather than the rule, and the per-instance memory model maps directly onto guest page tables.

## Crates

| Crate | Role |
|-------|------|
| [`nub/nub`](nub/nub) | Uniform `Nub<P: Personality>` handle over backends (in-process interpreter / KVM guest); personality-agnostic publish + invoke surface. |
| [`nub/nub-kernel`](nub/nub-kernel) | `Arch` substrate trait, generic `Kernel<A: Arch>`, shared types (`ObjHash`, invoke options/outcomes). |
| [`nub/nub-exec`](nub/nub-exec) | Pure PVM2 execution engine: interpreter, gas metering, memory pages, registers, `EcallHandler` seam. No personality awareness. |
| [`nub/nub-recompiler-x86`](nub/nub-recompiler-x86) | x86-64 JIT recompiler for PVM2 bytecode. |
| [`nub/nub-arch-local`](nub/nub-arch-local) | In-process `Arch` impl: simulates the CPU + MMU substrate with Rust data structures; backs the Local backend. |
| [`nub/nub-arch-x86`](nub/nub-arch-x86) / [`nub-arch-x86-abi`](nub/nub-arch-x86-abi) | Generic bare-metal guest-kernel lib (ring-0 boot, IDT, page tables, JIT trampoline, `GuestPersonality` traits, `register_guest_kernel!`) + the host↔guest wire ABI. |
| [`nub/nub-host-kvm`](nub/nub-host-kvm) | Host-side KVM sandbox driver (diverged fork of `hyperlight-host`; Linux/KVM/x86-64 only); ships opaque objects + invocations into the guest. |
| [`nub/nub-host-common`](nub/nub-host-common) | Types shared between host and guest (fork of `hyperlight-common`): rkyv RPC envelopes, vmem/layout/handshake constants. |
| [`nub/nub-host-guest-macro`](nub/nub-host-guest-macro) | `#[guest_function(fn_id = N)]` proc macro (fork of `hyperlight-guest-macro`): integer-id guest-function registration. |
| [`nub/nub-arch-guestbin`](nub/nub-arch-guestbin) | Vendored `no_std` guest runtime (fork of `hyperlight-guest-bin`): ELF entrypoint, talc allocator, fn_id dispatch table. Workspace-excluded; only built on the `x86_64-unknown-none` cross-compile. |
| [`nub/nub-build`](nub/nub-build) | build.rs helpers: `pvm2` cross-compiles + links a PVM2 program; `arch_x86` cross-compiles a bare-metal guest kernel for `x86_64-unknown-none`. |
| [`nub/nub-program`](nub/nub-program) | The PVM2 program blob: code, data-region geometry, entry points, plus the address-space ABI constants. `no_std`, zero deps. |
| [`nub/nub-linker`](nub/nub-linker) | RISC-V ELF → PVM2 linker: section concatenation, AUIPC-pair resolution, ecall rewriting, fallthrough injection, PVM2 validation. Emits a `ProgramBlob`. |
| [`nub/nub-rt`](nub/nub-rt) / [`nub-rt-macro`](nub/nub-rt-macro) | Guest-side runtime for PVM2 programs: compiler builtins, panic handler, `#[nub_rt::endpoint(N)]`. |
| [`nub/programs`](nub/programs) | PVM2 compute programs (sieve, keccak, blake2b, ed25519, ecrecover, Goldilocks/Poseidon2, STARK-shaped kernels). Each kernel is a plain `pub fn -> u32`, so it also compiles to native and wasm. |
| [`nub/nub-bench`](nub/nub-bench) | Benchmarks + end-to-end coverage: every program through the interpreter, with pinned `(return_value, gas)` vectors. |
| [`nub/bench-compare`](nub/bench-compare) | Cross-engine comparison against native / PolkaVM / Wasmtime / Wasmer. Its own workspace, excluded — see its [README](nub/bench-compare/README.md). |
| [`rust/build-crate`](rust/build-crate) | Generic sub-cargo build runner used by `nub-build` (separate `CARGO_TARGET_DIR`, isolated guest builds). |

## Building a personality

A downstream kernel personality is a pair of crates on either side of the sandbox boundary, plus a build step that ties them together:

1. **Host side** — depend on `nub` and implement `Personality`, whose `Local` associated type is your `LocalKernel` impl: the in-process object store + interpreter wiring (decode/validate/content-hash published bytes, resolve an invocation's root object, run it on `nub-arch-local`).
2. **Guest side** — depend on `nub-arch-x86` and implement `GuestPersonality`: ecall dispatch, the guest-resident state store, page sourcing for the CoW fault handler, gas sourcing.
3. **Guest binary** — a `no_std`/`no_main` guest-bin crate that invokes `register_guest_kernel!(YourGuestPersonality)`. The macro stamps out the substrate's `#[guest_function]` registrations (invoke, publish, JIT-evict, optional heap-stats) over your personality; exactly one personality per guest binary, enforced structurally.
4. **Build the blob** — in your entrypoint crate's `build.rs`, call `nub_build::build(<guest crate dir>, <bin name>, &features)` (crate `nub-build`) to cross-compile the guest-bin crate for `x86_64-unknown-none` and get the ELF path back, then expose it to the host binary (e.g. via `cargo:rustc-env`).
5. **Construct** — `Nub::new_local(...)` for the in-process backend, or `Nub::create_hyperlight(blob_path, options)` for the sandbox.

Two substrate-wide constraints to design around:

- **One live Hyperlight sandbox per process.** The guest-VA window is a single process-wide reservation; a second `create_hyperlight` — concurrent or sequential, even after dropping the first sandbox — fails loudly with `SandboxAlreadyCreated`. Personality entrypoint crates typically wrap the constructor in a process-wide singleton and reuse the one sandbox across callers.
- **fn_id bands.** Nub owns guest-function id space `[0, 0x100)` (production RPCs in `nub-arch-x86-abi`, generic test probes in `nub-arch-x86::test_abi`); personalities own `[0x100, ...)` for their own test/bench probes.

## Requirements

The sandbox path is **Linux x86-64 only** and needs **/dev/kvm** access (the host driver is a KVM-specific fork; other hypervisor backends were deliberately stripped). The interpreter-side crates build and test without KVM.

## Build & test

```bash
cargo build --workspace
cargo test --workspace                # incl. the PVM2 program conformance vectors
cargo bench -p nub-bench              # interpreter throughput + JIT emission throughput
```

`cargo test` cross-compiles every crate in `nub/programs` to the custom
`riscv64emc-pvm2` target, so it needs the `rust-src` component. Set
`SKIP_GUEST_BUILD=1` to skip that when you only want to typecheck.

Cross-engine comparison lives in its own workspace:

```bash
cd nub/bench-compare
cargo run --release -p bench-build    # fan every kernel out to 4 targets
cargo run --release -- validate       # do all engines agree?
./scripts/run.sh                      # measure -> BENCHMARKS.md
```

The toolchain is pinned in `rust-toolchain.toml` (see the comment there — the pin tracks the alleyos/alley monorepo and bumps in lockstep with it).

## Mirror of alleyos/alley

This repository is a **two-way manual mirror** of the `nub/` tree (plus `rust/build-crate`) of the [alleyos/alley](https://github.com/alleyos/alley) monorepo, where nub is developed alongside its first personality. The layout is **path-identical by design**: crates sit under `nub/` here, and `build-crate` under `rust/`, exactly as they do in the monorepo — that is why a standalone repo keeps its crates in a `nub/` subdirectory instead of at the top level. Identical paths mean zero file-content divergence between the two checkouts, and migrations in either direction are plain `git format-patch` / `git am` with no path munging.

History was extracted from the monorepo with git-filter-repo: historical file locations were renamed into the current layout so `git log --follow` works across the extraction, and `Co-Authored-By` AI trailers were rewritten to `Assisted-by` per the project's attribution rules.

### Mirror boundary

The mirror boundary **is** the pathspec `-- nub/ rust/build-crate`. Everything else in this repo — `README.md`, `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml`, `.gitignore`, `LICENSE`, `.github/`, `.mirror-state.toml` — is repo-local and never syncs in either direction.

[`.mirror-state.toml`](.mirror-state.toml) anchors the sync state: `jar-synced` is the last alley master commit whose mirrored trees are fully reflected here, `nub-synced` the last local master commit fully reflected in alley. The procedures below read and update it.

The commands assume `~/jar` is a checkout of alleyos/alley and `~/nub` is this repo.

### jar → nub (alley updated first)

This repo's master is updated directly:

```bash
# In ~/jar (master up to date):
git format-patch --no-renames --stdout <jar-synced>..master -- nub/ rust/build-crate > /tmp/sync.patch

# In ~/nub:
git am /tmp/sync.patch
```

Then update `.mirror-state.toml`: set `jar-synced` to the new jar master SHA and `nub-synced` to the new local tip (the trees agree again), commit, and push origin master.

`--no-renames` matters when a change *crosses* the boundary — e.g. a
crate moving from `rust/` into `nub/`. With rename detection on, git may
emit a `rename from` whose source the pathspec filtered out, which will
not apply. With it off, the deletion half is simply dropped (correctly:
the other repo never had that path) and the patch is a clean add.

Fallback for gnarly patch failures (heavy conflicts, rename storms):

```bash
git -C ~/jar diff <jar-synced> master -- nub/ rust/build-crate | git -C ~/nub apply
```

then a single commit citing the jar commit range it imports.

### nub → jar (this repo updated first)

jar changes go via PR:

```bash
# In ~/nub (master up to date):
git format-patch --no-renames --stdout <nub-synced>..master -- nub/ rust/build-crate > /tmp/sync.patch

# In ~/jar, on a branch:
git am /tmp/sync.patch
```

Open a PR on alleyos/alley from that branch. After it merges, update `.mirror-state.toml` (`nub-synced` = the local tip that was exported, `jar-synced` = the alley merge commit), commit, and push.

### Verifying a sync point

A manifest inside the boundary must never inherit a dependency with
`workspace = true` unless *both* root manifests define it. `thiserror`
and `proptest` qualify; anything else must be a path dep, or the
standalone build fails to load the manifest.

Guest dependency versions are pinned exactly (`=x.y.z`) in
`nub/programs/*`. The `(return_value, gas)` vectors depend on the
emitted instruction stream, so a caret range letting the two repos
resolve different patch releases silently changes them — it moved
ed25519's gas by 3.3x once.

At every sync point the mirrored trees must be byte-identical:

```bash
diff -r ~/jar/nub ~/nub/nub && diff -r ~/jar/rust/build-crate ~/nub/rust/build-crate
```

Both diffs must be empty. Concurrent updates on both sides are resolved manually at `git am` time, like any conflict.

## License

Apache-2.0 (see [LICENSE](LICENSE)).
