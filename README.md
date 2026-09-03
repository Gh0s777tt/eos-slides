# E-OS Slides

Presentation editor for E-OS — slides, outline and full-screen show.

**Status: skeleton.** The window opens, the headless model has tests, the CI
gates are wired — and no product feature is implemented. Nothing in this
repository does the job its name promises yet. That sentence is the honest
state and stays here until it stops being true.

- **Platform:** E-OS (Redox microkernel) is the primary and only *supported*
  target. Linux, macOS and Windows are **secondary** — a development
  convenience so the UI can be iterated without booting a VM. A host build is
  not a supported product.
- **UI:** [Slint](https://slint.dev) 1.17, software renderer. Under E-OS the
  platform comes from [`eos-ui`](https://gitlab.com/e-os/eos-ui) (Slint over
  Orbital, plus the font bootstrap); on a host it comes from the optional
  `host-backend` feature (winit).
- **Licence:** AGPL-3.0-or-later. Slint is used under its GPL-3.0-only arm.

## Download

Roadmap `PR-008`: one file per operating system, no Rust toolchain needed to
run it. `packaging/release.sh` builds one target, **looks at the file it
produced** (format via `file`, plus a 1 MiB size floor), and packages it with
a `.sha256` beside it. The `package-windows` and `package-linux` CI jobs run
that script and publish `dist/` as a pipeline artefact.

Measured on macOS (Apple Silicon, rustc 1.98.0-nightly `23a3312d9`, slint
1.17.1) on 2026-09-03. The middle columns are `file -b` and `wc -c` output,
not a description of it:

| target | binary | `file -b` says | bytes | packaged |
|---|---|---|---|---|
| `x86_64-pc-windows-gnu` | `eos-slides.exe` | `PE32+ executable (console) x86-64, for MS Windows` | 22 818 304 | yes — `.zip`, 10 433 615 B |
| `x86_64-unknown-linux-gnu` | `eos-slides` | `ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.0.0, stripped` | 17 001 800 | **no** — see below |

Check the archive before running anything out of it:

```sh
shasum -a 256 -c eos-slides-0.1.0-x86_64-pc-windows-gnu.zip.sha256
```

### The Linux binary builds, but is not packaged yet

`packaging/release.sh` sends only `*windows-gnu` through `cargo zigbuild`;
every other target falls through to a plain `cargo build`, and on macOS that
link is performed by Apple's `ld`, which does not accept GNU linker flags. The
whole dependency graph compiles first and then this happens:

```
error: linking with `cc` failed: exit status: 1
  ld: unknown options: --as-needed -Bstatic -Bdynamic --eh-frame-hdr -z --gc-sections -z -z --strip-debug
  clang: error: linker command failed with exit code 1
error: could not compile `eos-slides` (bin "eos-slides") due to 1 previous error
```

The binary in the table above was produced by going round the script:

```sh
cargo zigbuild --release --features host-backend --target x86_64-unknown-linux-gnu
```

So the *product* cross-compiles for Linux; the *packager* cannot drive that
from macOS. Until release.sh grows a Linux branch, or `eos-heavy` grows a
Linux executor, `package-linux` is expected to be red — deliberately red,
rather than hidden behind `allow_failure` (CLAUDE.md §13).

### The binaries are unsigned

No Authenticode, no minisign, no cosign. Windows SmartScreen will say so, and
it is right to. Signing product downloads needs a key a human generates and
holds outside this repository (CLAUDE.md §14, §19); until that key exists the
`.sha256` is the only integrity check on offer, and it only proves the file
matches the pipeline that built it.

### What is not proven

Neither binary has been **executed** on its own operating system. What was
checked is the file — its format and its size — not its behaviour. Nobody has
opened a window on Windows or on Linux from these builds. And, as the top of
this README says, a host build is a development convenience: E-OS is the
target that counts.

## Building

### On a host (development)

```sh
cargo test                                   # model tests, no display needed
cargo run -- --selftest                      # prints EOS-SLIDES-SELFTEST-OK
cargo run --features host-backend            # opens the window
cargo run --features host-backend -- --check-backend   # prints EOS-SLIDES-BACKEND-OK
```

`host-backend` is **off by default** on purpose (fail-closed): the default
dependency graph carries no windowing backend, so a Redox cross build cannot
pick winit up by accident and a CI container needs no display libraries. Without
the feature, `--check-backend` exits **2** — "the check could not run" — and the
plain GUI run refuses with the same code. That refusal is the negative test of
the backend gate; it is expected to be red, and CI asserts that it is.

The host backend build was measured on macOS (Apple Silicon, rustc 1.98.0,
slint 1.17.1) and compiles. Windows and Linux now **cross-compile** too, and
each needed a Cargo.toml fix to get there — see the `muda` and `i-slint-common`
comments next to those target sections, and the "Download" section above for
the binaries they produce. Still **[UNVERIFIED]: neither host binary has been
run**, only built and inspected; the Linux target section's `x11` and `wayland`
features have never been exercised against a real display server.

### For E-OS (the real target)

Built as a cookbook recipe in the meta-repo, not from this directory:

```sh
# in the E-OS meta-repo
bash scripts/eos-build.sh x86_64      # or aarch64
```

The recipe lives at `recipes/gui/eos-slides/recipe.toml` (a copy ready to install is
in `packaging/` here) and pins this repository by revision. Bumping that pin is
step 2 of the loop in CLAUDE.md §20.5 — push to **both** remotes, bump
`repos.toml` **and** the recipe, `scripts/eos-repos.sh pins --strict`, resync the
build tree, rebuild.

## First commit checklist

The generator does not run cargo, so the tree it produces has **no `Cargo.lock`**
and CI runs `--locked`. Before the first push:

1. `cargo build` — writes `Cargo.lock`; commit it (this is a binary crate, the
   lock belongs in git).
2. Replace the placeholder icon `assets/eos-slides.png` (48×48 Crimson placeholder
   drawn by the generator) with the real one.
3. Create the GitLab project `e-os/eos-slides` and the GitHub mirror, push to both.
4. Add the repository to the meta-repo `repos.toml` as **type A**, and to the
   type-A list in `CLAUDE.md` §11 — `scripts/eos-check-repo-types.py`
   (ci-integrity check 7) fails on a mismatch between the two, which is exactly
   what should happen if only one is updated.
5. Copy `packaging/recipes/gui/eos-slides/recipe.toml` into the meta-repo and put
   the real first-commit revision in it; `scripts/eos-repos.sh pins --strict`
   must be green.

## Exit codes

| code | meaning |
|---|---|
| 0 | success |
| 1 | a check found a defect (selftest failed, window could not be built) |
| 2 | the check could not run (no windowing backend linked in) |

The split is the same one `scripts/verify.sh` and `ci-integrity.sh` use in the
meta-repo: a broken tree and a broken toolbox need opposite reactions, so they
must not report the same thing (CLAUDE.md §13).

## Hosting

GitLab `gitlab.com/e-os/eos-slides` is the source of truth; GitHub
`github.com/Gh0s777tt/eos-slides` is a read-only mirror the build recipes may fetch
from (ADR-0001). This repository is **type A** — E-OS's own code — so every
rule for own code applies: tests with every change, a negative test for every
gate, docs in the same MR.

## Layout

```
src/main.rs      CLI (--selftest, --check-backend, --version, --help) + window wiring
src/model.rs     the headless half: catalogue, search, status line, selftest
ui/app.slint     the window (E-OS Crimson palette, identical to eos-notes)
build.rs         compiles the .slint file
assets/          Orbital launcher entry + 48×48 icon (placeholder — needs a designer)
packaging/       release.sh (per-OS packager) + the cookbook recipe for the meta-repo
deny.toml        cargo-deny: licences, sources, bans, advisories (feature graph)
osv-scanner.toml osv-scanner: advisory exceptions, each with an `ignoreUntil`
```

## Supply chain: two databases, and why both

`.gitlab-ci.yml` runs `cargo deny check` **and** `osv-scanner`. They disagree,
which is the point: cargo-deny walks the resolved **feature graph**, osv-scanner
reads **`Cargo.lock`**. Measured on a freshly generated tree —

```
cargo deny check advisories                    -> advisories ok            (exit 0)
osv-scanner scan source --lockfile Cargo.lock  -> 4 vulnerabilities        (exit 1)
```

The extra one is RUSTSEC-2025-0141 (bincode), which sits in the lockfile behind
an optional feature nothing enables, so cargo-deny cannot see it.

The second reason for `osv-scanner.toml` is **expiry**. cargo-deny 0.20.2 takes
only `id` and `reason` under `[[advisories.ignore]]`:

```
cargo deny --config <deny.toml with expires="…"> check advisories
  -> error[unexpected-keys]: found 1 unexpected keys, expected: ["id", "reason"]
```

so deny.toml's ignore list, which calls itself a debt register, has no mechanism
to expire. `osv-scanner.toml`'s `ignoreUntil` does: the finding comes back on the
date written next to it, and a stale exception fails the build.

`secret-scan` (gitleaks, full history) is the first job for the same reason it is
in the meta-repo (CLAUDE.md §13): a leaked credential is already too late by the
time the tests run.
