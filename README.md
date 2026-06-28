# Codex CLI — native iOS build via GitHub Actions

This is a tiny standalone repo that builds the **real OpenAI Codex CLI** Rust
binary for `aarch64-apple-ios` on a free GitHub Actions macOS runner, then
packages it so you can drop it into the jailbreak `.deb` on your iPhone.

No Mac required at home. You only need a GitHub account.

## What this produces

A GitHub Actions artifact `codex-aarch64-apple-ios.tar.gz` containing:

| File | What |
|---|---|
| `codex` | Native Mach-O arm64, `LC_BUILD_VERSION platform=7` (iOS), min 16.0, ad-hoc signed with ldid + entitlements |
| `entitlements.xml` | The ldid entitlements blob (no-sandbox, allow-jit, etc.) |
| `SHA256SUMS` | sha256 of the above |

You extract it on the phone and copy `codex` to
`/var/jb/usr/local/lib/codex/codex-ios`. The existing `codex` wrapper in the
deb execs it directly — same layout as Claude Code iOS.

## One-time setup (5 steps)

1. **Create a new GitHub repository.** Make it **public** (public repos get
   free Actions minutes with no monthly cap). Name it e.g. `codex-ios-build`.
   Do NOT initialize it with a README.

2. **Push the contents of this `codex-ios-build/` folder to the repo's
   `main` branch.** The tree must look like:
   ```
   .github/workflows/build-ios.yml
   patches/001-arboard-ios-stub.patch
   patches/002-cli-build-objc-ios.patch
   entitlements.xml
   README.md
   ```
   From a computer:
   ```sh
   cd codex-ios-build
   git init
   git add .
   git commit -m "Codex iOS build workflow"
   git branch -M main
   git remote add origin git@github.com:YOUR_USER/codex-ios-build.git
   git push -u origin main
   ```

3. **Enable Actions.** On GitHub: repo → Settings → Actions → General →
   "Allow all actions". Workflow runs on push to `main` and via manual
   dispatch.

4. **Run the workflow.** Repo → Actions tab → "Build Codex iOS" →
   "Run workflow" → `main` → click Run.

5. **Wait.** A release build of `codex` is ~30–50 min on a `macos-15`
   runner. When the job is green, scroll to the bottom of the run →
   download the `codex-aarch64-apple-ios` artifact.

## Install the binary on the iPhone

You need the artifact on the phone. Easiest paths:

- **From the phone (Safari):** log into github.com, open the run, download
  the artifact zip. Safari saves it to Files. (GH artifact downloads come
  as a `.zip` wrapping the `.tar.gz`.)
- **From a computer:** download the artifact, AirDrop or `scp` the
  `.tar.gz` to the phone.

Then in NewTerm 3 (as root — use `sudo su` or your root shell):

```sh
# 1. Get the tarball somewhere, e.g. ~/codex/codex-aarch64-apple-ios.tar.gz
cd /var/jb/var/mobile/codex
tar xzf codex-aarch64-apple-ios.tar.gz   # produces ./codex  ./entitlements.xml  ./SHA256SUMS
shasum -a 256 codex                       # verify against SHA256SUMS

# 2. Drop it into the deb's engine path.
cp codex /var/jb/usr/local/lib/codex/codex-ios
chmod 0755 /var/jb/usr/local/lib/codex/codex-ios

# 3. (Re)sign with the entitlements already shipped by the deb.
ldid -S/var/jb/usr/local/lib/codex/entitlements.xml /var/jb/usr/local/lib/codex/codex-ios

# 4. Sanity check.
file /var/jb/usr/local/lib/codex/codex-ios
otool -L /var/jb/usr/local/lib/codex/codex-ios | head
```

Then just run `codex` in NewTerm. It should start the real Rust TUI.

If you haven't authenticated: run `codex-auth` first (ships with the deb),
or set `OPENAI_API_KEY` in `~/.codex/credentials`.

## How the build works (technical)

Codex CLI v0.142.3 is pure Rust (`codex-rs/` workspace). The `codex` binary
is built from `codex-rs/cli` (`[[bin]] name = "codex"`). OpenAI ships
prebuilt binaries for 4 targets only — none for iOS. The darwin-arm64
binary dies on iOS with `exit 137` (SIGKILL) because it links
AppKit.framework (absent on iOS) and is tagged `platform=1` (macOS).

So we cross-compile to `aarch64-apple-ios` on a `macos-15` runner (which
has Xcode + the iPhoneOS SDK). Two patches make it link:

### Patch 1 — `arboard` / AppKit (clipboard)
`codex-rs/tui` pulls `arboard` (→ `objc2-app-kit` → AppKit) under
`cfg(not(target_os = "android"))`. iOS is not Android so `arboard` would
be included and the link fails. The TUI already has Android stubs that
return "unsupported" and skip `arboard`. The patch broadens those stubs
to `any(target_os = "android", target_os = "ios")` and excludes iOS from
the `arboard` dependency. iOS has no CLI-reachable clipboard anyway, so
nothing real is lost; OSC 52 / tmux clipboard (over SSH) still work.

### Patch 2 — `-ObjC` linker flag on iOS
`codex-rs/cli/build.rs` emits `-ObjC` only under `target_os = "macos"`.
The same flag is required on iOS for `objc2` categories to link. The
patch adds the `ios` arm.

### TLS — `aws-lc-sys`
`rustls` uses the `aws_lc_rs` provider → builds `aws-lc-sys` (C + asm).
The workflow sets `CC_aarch64-apple-ios` / `CXX_…` / `AR_…` to
`xcrun --sdk iphoneos --find clang/clang++/ar`, points CMake at a
generated iOS toolchain file, and sets `AWS_LC_SYS_NO_JITTER_ENTROPY=1`
(same trick the official musl build uses).

If `aws-lc-sys` still fails to link for iOS, the fallback is to switch
rustls to the `ring` provider (prebuilt asm for aarch64-apple-ios). That
lives in a separate workflow file `build-ios-ring.yml` (add it if needed —
see Troubleshooting).

### Things verified NOT to be blockers
- `keyring` `apple-native` → Security.framework (present on iOS).
- `system-configuration` → SystemConfiguration.framework (present on iOS).
- `linux-sandbox` (landlock/seccomp) → linux-only cfg, skipped on iOS.
- `codex-windows-sandbox` → windows-only cfg, skipped.
- `v8` / `codex-v8-poc` / `rusty_v8` → only used by the `codex-v8-poc`
  crate, which is NOT in the `codex` bin dependency graph. We do not call
  `setup-rusty-v8`.
- `sqlite-bundled`, `onig` (syntect) → C code, built with the same
  `CC_aarch64-apple-ios`.

## Troubleshooting

The first run may fail. The most likely failure points, in order:

1. **`aws-lc-sys` link/asm error.** This is the most fragile C dependency.
   If it fails, create `.github/workflows/build-ios-ring.yml` based on
   `build-ios.yml` but add a step before `cargo build` that patches the
   workspace `Cargo.toml` to use `ring` instead of `aws_lc_rs`:
   ```sh
   sed -i '' 's/"aws_lc_rs"/"ring"/g' codex-rs/Cargo.toml
   ```
   and remove the `aws_lc_rs` feature from the `rustls` line. `ring`
   ships prebuilt asm for aarch64-apple-ios and is much friendlier.

2. **`keyring` `apple-native` compile error** (uses a macOS-only API).
   Disable the feature by patching `codex-rs/keyring-store/Cargo.toml`:
   remove the `apple-native` feature from the `macos` target block, or
   add an `ios` block with no keyring feature. Auth then falls back to
   `~/.codex/auth.json` plaintext (already supported by `codex-auth`).

3. **`syntect`/`onig` C compile error.** Switch syntect to the
   `regex-fancy` backend by patching `codex-rs/tui/Cargo.toml`:
   replace `two-face`/`syntect` features with the pure-Rust backend.

4. **`error: linking with cc failed`** — means `RUSTFLAGS` isysroot is
   wrong. Verify `xcrun --sdk iphoneos --show-sdk-path` output and that
   `$IOS_SDK` env is set.

When a run fails, paste the **last 80 lines of the failing step's log**
into the chat and I will write the next patch. This is an iterative
process — expect 1–3 attempts before green.

## Files in this repo

```
.github/workflows/build-ios.yml   the workflow
patches/001-arboard-ios-stub.patch  clipboard/AppKit fix
patches/002-cli-build-objc-ios.patch  -ObjC linker flag for iOS
entitlements.xml                  ldid entitlements blob (copy of deb's)
README.md                         this file
```

## License note

The Codex source is Apache-2.0 (OpenAI). The iOS SDK is Apple's and is
used via Xcode on the GitHub runner — you do not redistribute the SDK,
only the compiled binary, which is fine for personal jailbreak use.
