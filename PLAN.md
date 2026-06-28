# Codex CLI native on iPhone — build via GitHub Actions (no Mac needed at home)

## Goal
Produce a native `aarch64-apple-ios` build of the real Codex CLI Rust engine
(`codex-rs/cli`, binary name `codex`) using a free GitHub Actions `macos-latest`
runner, then drop it into the existing `.deb` so `codex` works in NewTerm 3
exactly like `claude` does.

## Why this needs a workflow (not a local build)
- Codex v0.142.3 is pure Rust (`codex-rs/` workspace). The `codex` binary is
  built from `codex-rs/cli` (`[[bin]] name = "codex"`).
- Official releases ship only 4 targets: `aarch64-apple-darwin`,
  `x86_64-apple-darwin`, `x86_64-unknown-linux-musl`,
  `aarch64-unknown-linux-musl`. **No `aarch64-apple-ios`.**
- The darwin-arm64 binary dies on iOS with `exit 137` (SIGKILL): it links
  AppKit.framework (absent on iOS; iOS uses UIKit) and has
  `LC_BUILD_VERSION platform=1` (macOS), not iOS.
- So we must cross-compile to `aarch64-apple-ios`, which requires the iOS SDK
  (from Xcode). Xcode only runs on macOS → use a `macos-latest` GH Actions
  runner (free for public repos; included minutes for private).

## The two real blockers (and how the workflow solves them)

### Blocker 1 — `arboard` → AppKit (clipboard)
- `codex-rs/tui/Cargo.toml` pulls `arboard` under
  `cfg(not(target_os = "android"))`. iOS is not Android, so on iOS `arboard`
  is included → it links `objc2-app-kit` → AppKit.framework → link failure
  (AppKit headers/framework absent in iPhoneOS SDK).
- `arboard` is used only for clipboard copy (`/copy`, Ctrl+O) and image paste.
  The TUI already has `cfg(target_os = "android")` stubs that return a clear
  "unsupported" error and skip `arboard` entirely.
- **Fix**: a patch that treats iOS like Android for clipboard purposes:
  - `tui/Cargo.toml`: change `cfg(not(target_os = "android"))` →
    `cfg(not(any(target_os = "android", target_os = "ios")))` for the
    `arboard` dependency.
  - `tui/src/clipboard_copy.rs`: add an `ios` arm to `arboard_copy` mirroring
    the Android stub (or broaden the existing Android cfg to `any(android, ios)`).
  - `tui/src/clipboard_paste.rs`: broaden the Android stubs for
    `paste_image_as_png` / `paste_image_to_temp_png` to also cover `ios`.
- Result: no AppKit link. Clipboard features return a clear error in the TUI;
  everything else works. (iOS has no system clipboard API reachable from a
  CLI process anyway.)

### Blocker 2 — `aws-lc-sys` (TLS via rustls `aws_lc_rs` feature)
- `rustls` is configured with `aws_lc_rs` → builds `aws-lc-sys` (C + asm).
  Needs a C compiler for the iOS target + cmake + a toolchain file.
- On a `macos-latest` runner with Xcode, `xcrun --sdk iphoneos --find clang`
  gives us the right clang. We set:
  - `CC_aarch64-apple-ios=$(xcrun --sdk iphoneos --find clang)`
  - `CXX_aarch64-apple-ios=$(xcrun --sdk iphoneos --find clang++)`
  - `AR_aarch64-apple-ios=$(xcrun --sdk iphoneos --find ar)`
  - `CMAKE_TOOLCHAIN_FILE` pointing at a generated toolchain file that
    references the iOS SDK.
  - `AWS_LC_SYS_NO_JITTER_ENTROPY=1` (same trick the official musl build uses).
- If `aws-lc-sys` still fights us, fallback: switch rustls to `ring` provider
  via a workspace patch (`rustls = { features = ["ring"] }` instead of
  `aws_lc_rs`). `ring` has prebuilt asm for aarch64-apple-ios and is much
  friendlier. This is plan B inside the workflow.

## Things that are NOT blockers (verified)
- `keyring` `apple-native` → `security-framework` crate → Security.framework,
  which **exists on iOS**. Should link fine.
- `system-configuration` crate → SystemConfiguration.framework (present on iOS).
- `linux-sandbox` (landlock/seccomp) → `cfg(target_os = "linux")`, skipped on iOS.
- `codex-windows-sandbox` → windows-only cfg, skipped.
- `v8` / `codex-v8-poc` / `rusty_v8` → only used by the `codex-v8-poc` crate,
  which is **not** in the `codex` bin dependency graph (not in cli/core/tui).
  We do NOT call `setup-rusty-v8` in our workflow.
- `cli/build.rs` adds `-ObjC` linker arg only under `cfg(target_os = "macos")`.
  iOS target is `target_os = "ios"`, so this won't fire automatically — we add
  `-ObjC` via `RUSTFLAGS` so objc2 categories link correctly.

## Deliverables the workflow produces
1. `codex` — native Mach-O arm64, `LC_BUILD_VERSION platform=7` (iOS),
   min-os 16.0, ad-hoc signed with `ldid -S<entitlements.xml>`.
2. A tarball `codex-aarch64-apple-ios.tar.gz` containing `codex` + a small
   `entitlements.xml`, uploaded as a GH Actions artifact (14-day retention).
3. A `SHA256SUMS` file alongside it.

You download the artifact on your iPhone (or on any computer and AirDrop/SCP
it to the phone), extract, and drop `codex` into
`/var/jb/usr/local/lib/codex/codex-ios`, then reinstall (or just chmod+x and
re-sign). The existing wrapper already execs it directly.

## File layout this plan adds
```
codex-ios-build/                          ← new folder for the GH Actions project
├── .github/workflows/build-ios.yml       ← the workflow
├── patches/
│   ├── 001-arboard-ios-stub.patch        ← tui/Cargo.toml + clipboard_*.rs
│   └── 002-cli-build-objc-ios.patch      ← cli/build.rs: also add -ObjC on iOS
├── entitlements.xml                      ← ldid entitlements (copy of deb's)
└── README.md                             ← how to fork, run, download, install
```

These live **outside** the deb tree (`codex-ios/`), as a separate mini-repo
you push to your GitHub fork. The deb itself stays as-is on the phone; only
the `codex-ios` native binary gets replaced after the workflow produces it.

## Step-by-step plan

### Step 1 — Create the build project scaffold
- `mkdir -p codex-ios-build/.github/workflows codex-ios-build/patches`
- Copy the existing `entitlements.xml` from
  `codex-ios/var/jb/usr/local/lib/codex/entitlements.xml` into
  `codex-ios-build/entitlements.xml`.

### Step 2 — Write the iOS clipboard patch
File `patches/001-arboard-ios-stub.patch`. A unified diff against
`openai/codex` tag `rust-v0.142.3` that:
- `codex-rs/tui/Cargo.toml`: replace
  `cfg(not(target_os = "android"))` (the arboard dep block) with
  `cfg(not(any(target_os = "android", target_os = "ios")))`.
- `codex-rs/tui/src/clipboard_copy.rs`:
  - Replace `#[cfg(target_os = "android")]` on `arboard_copy` stub with
    `#[cfg(any(target_os = "android", target_os = "ios"))]`.
  - Replace the non-android, non-linux `#[cfg(all(not(target_os = "android"), not(target_os = "linux")))]` on the real `arboard_copy` with
    `#[cfg(all(not(any(target_os = "android", target_os = "ios")), not(target_os = "linux")))]`.
  - Update the stub error string to mention iOS.
- `codex-rs/tui/src/clipboard_paste.rs`:
  - Broaden the three `#[cfg(target_os = "android")]` stubs to
    `#[cfg(any(target_os = "android", target_os = "ios"))]`.
  - Broaden the matching `#[cfg(not(target_os = "android"))]` real impls to
    `#[cfg(not(any(target_os = "android", target_os = "ios")))]`.

### Step 3 — Write the cli/build.rs iOS patch
File `patches/002-cli-build-objc-ios.patch`:
- `codex-rs/cli/build.rs`: change
  `if CARGO_CFG_TARGET_OS == "macos"` to
  `if CARGO_CFG_TARGET_OS == "macos" || CARGO_CFG_TARGET_OS == "ios"`.

### Step 4 — Write the workflow
File `.github/workflows/build-ios.yml`:
- Trigger: `workflow_dispatch` (manual) + `push` to `main` (optional).
- Runner: `macos-15` (has Xcode + iOS SDK).
- Steps:
  1. `actions/checkout@v4` of `openai/codex` at `rust-v0.142.3` (or a
     submodule/pinned ref). Actually: checkout the user's fork, then `git
     pull` upstream tag, OR use a second checkout with `repository: openai/codex`
     and `ref: rust-v0.142.3`. Simplest: a single checkout of openai/codex at
     the tag into the workspace root, then apply our patches on top.
  2. `dtolnay/rust-toolchain@stable` with `targets: aarch64-apple-ios`.
  3. `apply patches/*.patch` via `git apply` (or `patch -p1`).
  4. Install `cmake`, `ldid` (brew). `ldid` is needed for ad-hoc signing.
  5. Set env: `CC_aarch64-apple-ios`, `CXX_...`, `AR_...` via
     `xcrun --sdk iphoneos --find ...`; `IOS_SDK=$(xcrun --sdk iphoneos --show-sdk-path)`;
     `AWS_LC_SYS_NO_JITTER_ENTROPY=1`;
     `RUSTFLAGS="-C link-arg=-ObjC -C link-arg=-isysroot -C link-arg=$IOS_SDK"`.
  6. Generate a CMake toolchain file for `aws-lc-sys` (CC/CXX/AR + sysroot).
  7. `cargo build --target aarch64-apple-ios --release --bin codex` from
     `codex-rs/`. (Do NOT build `codex-responses-api-proxy` etc. unless wanted;
     keep the matrix tiny.)
  8. `strip` the binary.
  9. `ldid -S<entitlements.xml>` the binary.
  10. Verify: `file`, `otool -L`, `otool -l | grep LC_BUILD_VERSION` → expect
      platform 7 (iOS), minos 16.0.
  11. Tar `codex` + `entitlements.xml` → `codex-aarch64-apple-ios.tar.gz`.
  12. `actions/upload-artifact@v4` with the tarball + SHA256SUMS.
- Fallback job (plan B for TLS): a second job that sets
  `CARGO_FEATURE_RING=1`-style override by patching the workspace
  `rustls` features from `aws_lc_rs` to `ring`, in case job A fails at the
  `aws-lc-sys` link step. Mark `continue-on-error` or make it a separate
  workflow file `build-ios-ring.yml` to iterate independently.

### Step 5 — Write the build-project README
- How to fork `openai/codex` is NOT required: the workflow checks out the
  upstream tag directly. The user only needs to create a repo containing
  `.github/workflows/build-ios.yml` + `patches/` + `entitlements.xml` and
  push it to their GitHub.
- How to run: Actions tab → "Build Codex iOS" → Run workflow → wait →
  download artifact.
- How to install on the phone:
  - Download artifact on iPhone (Safari can download GH Actions artifacts
    if logged in; or use a computer + AirDrop/SCP).
  - `tar xzf codex-aarch64-apple-ios.tar.gz` → get `codex`.
  - `scp`/AirDrop to `/var/jb/var/mobile/codex/`.
  - As root: `cp codex /var/jb/usr/local/lib/codex/codex-ios`,
    `chmod 0755`, `ldid -S/var/jb/usr/local/lib/codex/entitlements.xml /var/jb/usr/local/lib/codex/codex-ios`.
  - Run `codex` in NewTerm.

### Step 6 — Update the deb README
- Add a section pointing at the GH Actions project as the canonical way to
  produce `codex-ios` without a Mac.

## Risks / unknowns we will only learn by running the workflow
- `aws-lc-sys` may need extra flags for iOS (asm path). Plan B = `ring`.
- `keyring` `apple-native` may pull a macOS-only API; if so, feature-disable
  it and store credentials in `~/.codex/auth.json` plaintext (already
  supported by `codex-auth`).
- `syntect`/`two-face` use `onig` (Oniguruma C lib). `onig` needs a C
  compiler — same `CC_aarch64-apple-ios` should cover it. If it fails, switch
  to the `regex-fancy` backend via a patch (syntect supports it).
- `sqlite-bundled` (libsqlite3-sys) builds its own SQLite from C source —
  same `CC_aarch64-apple-ios` covers it.
- iOS min-version: we target 16.0 to match the device (iOS 16.1). Set via
  `IPHONEOS_DEPLOYMENT_TARGET=16.0`.

## What the user does next (after this plan is approved)
1. I create the `codex-ios-build/` project with the workflow + patches + README.
2. User creates a new GitHub repo (public = free minutes), pushes the
   `codex-ios-build/` contents to `main`.
3. User triggers the workflow manually.
4. If it fails, user pastes the failing log here; I iterate on patches
   (most likely on `aws-lc-sys` → `ring`).
5. Once green, user downloads the artifact, drops `codex` into the deb's
   `codex-ios` path, rebuilds the deb (without `--force`), reinstalls.
6. `codex` runs natively in NewTerm, mirroring `claude`.
