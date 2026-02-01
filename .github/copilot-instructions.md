# GitHub Copilot instructions for iSH 🐧📱

## Quick summary
- Purpose: iSH provides a usermode x86 Linux shell on iOS via an emulator + syscall translation (see `README.md`).
- Primary code areas: 
  - Emulator / CPU + JIT: `emu/`, `jit/` (assembly-heavy gadgets under `jit/gadgets-*`).
  - Kernel/syscall emulation: `kernel/` (core mapping in `kernel/calls.c`).
  - Platform glue & UI: `app/` (Objective-C iOS app + `iSH.xcconfig`, `xcode-meson.sh`).
  - Tools & tests: `tools/` (e.g., `fakefsify`, `ptraceomatic`) and `tests/` (e2e and unit tests).

## Be productive quickly (must-knows) ✅
- Build (CLI test harness):
  - Local: `meson build -Dengine=jit -Dkernel=ish && ninja -C build`
  - CI uses: `meson build -Dengine=jit -Dkernel=${kernel}` and `ninja -C build` / `ninja -C build test` (see `.github/workflows/ci.yml`).
- iOS / Xcode: open the Xcode project, update `ROOT_BUNDLE_IDENTIFIER` and team ID in the project or use `xcodebuild -project iSH.xcodeproj -scheme iSH -arch arm64 -sdk iphoneos CODE_SIGNING_ALLOWED=NO` for CI-style builds. `app/xcode-meson.sh` integrates Meson build dir with the Xcode build.
- Tests & E2E:
  - Unit/E2E: `ninja -C build test` (CI runs tests only for `kernel=ish`).
  - E2E harness: `tests/e2e/e2e.bash` automates tests and expects an Alpine i386 minirootfs TAR (it downloads one when needed). Use `./build/tools/fakefsify <alpine.tar.gz> alpine` to create the test filesystem (requires `libarchive`).
- Submodules: repo uses submodules (libapps, libarchive, etc.). Clone with `--recurse-submodules` or run `git submodule update --init`.

## Important patterns & project conventions 🔧
- Logging channels: Many files define a `DEFAULT_CHANNEL` and use `TRACE`/`STRACE` macros. Enable channels with `meson configure -Dlog="strace verbose"` or set `ISH_LOG` in `app/iSH.xcconfig`.
- Syscall surface: `kernel/calls.c` contains the syscall table mapping number → `sys_*` function. Add syscalls by implementing `sys_<name>` and updating the table. Missing syscalls often use harmless stubs (look for `syscall_stub` / `syscall_success_stub`).
- JIT & assembly:
  - The JIT is not typical machine-code JIT; it generates chains of gadget functions (see `jit/gen.c`, `jit/jit.c`). Gadgets are implemented in assembly under `jit/gadgets-*`.
  - Be cautious when editing assembly: gadget conventions and `jit/offsets.c` must be respected.
- Testing & diffs: E2E tests live under `tests/e2e/<test>/expected.txt`. The E2E harness compares output with these files (use it for regression tests).
- Debugging helpers:
  - `tools/ptraceomatic` and `tools/unicornomatic` are used to compare single-stepped behavior against a reference (requires 64-bit Linux for ptraceomatic).
  - Use `ISH_LOG` / `meson configure -Dlog=...` to enable `strace` and `instr` logging for diagnosing syscalls or instruction execution.

## Typical change workflow for agents ✍️
1. Reproduce: build with Meson + Ninja and run unit/e2e tests locally (`ninja -C build test` and `tests/e2e/e2e.bash -f alpine -y`).
2. Add behavior: implement `sys_*` in `kernel/` or modify emulator/JIT in `emu/` / `jit/`.
3. Add tests: create a small e2e test under `tests/e2e/<name>/` with a `test.sh` and `expected.txt`.
4. Run the E2E harness to validate.
5. For JIT/ASM changes: add targeted unit tests and smoke tests; ensure offsets/gadget invariants keep passing.

> Note: E2E tests rely on downloading an Alpine minirootfs; network access and `libarchive` are required to create the test filesystem.

## Useful file references (start here) 📚
- High-level: `README.md` (developer setup and notes on the JIT)
- Build & options: `meson_options.txt`, `.github/workflows/ci.yml`, `app/xcode-meson.sh` and `app/iSH.xcconfig`
- Syscalls: `kernel/calls.c` (syscall table), and `kernel/` for syscall implementations
- Emulator: `emu/*` (CPU, interp), `jit/*` (JIT and gadgets)
- Tests: `tests/` (unit and `tests/e2e/`) and `tests/e2e/e2e.bash`
- Tools: `tools/fakefsify` (builds rootfs), `tools/ptraceomatic` (single-step comparison)

## Do not assume / common pitfalls ⚠️
- The JIT uses handwritten assembly; changing it requires careful attention to calling conventions and gadget layout.
- Many features are intentionally incomplete—look for `TODO` macros and comments; add tests for any behavioral change.
- Building the CLI tools used in tests requires `libarchive` installed (otherwise `tools/fakefsify` won't be generated).

## PR expectations
- Keep changes focused and test-backed (add `tests/e2e` entries where applicable).
- Run the build/test matrix locally for the variant you change (`kernel=ish` is used for most tests).
- For platform changes (iOS/macOS): prefer small incremental changes and include notes for Xcode configuration when needed.

---
If anything here is unclear or you'd like more detail about a specific area (JIT, syscall table, e2e tests, or iOS build flow), tell me which section to expand and I'll update this file. 👇
