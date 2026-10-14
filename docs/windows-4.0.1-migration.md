# Migration note: 4.0.1 → 6.5.0 on Windows x64

A short record of why a local 4.0.1 installation was replaced by 6.5.0, and why
the interim local patches are no longer used. Kept for context; not a code
artifact.

## What was happening

A workstation was running `npx rea-agents@4.0.1`. Two gaps were observed there:

1. **Android tool unwired.** `project_android_application_graph` existed in the
   4.0.1 codebase (`dist/application/AndroidApplicationService.js`,
   `dist/domain/androidApplication.js`) but was never registered in the CLI or
   the MCP catalogue. It was dead code in that release.
2. **Ghidra blocked on Windows.** `hasWindowsNativeAuthority()` hard-returned
   false on `win32`, so all Ghidra operations reported `unsupported_host` even
   though Ghidra 12.1.4 headless + JDK 21 demonstrably analysed PE files on the
   host.

## Interim patches (4.0.1, local fork — now retired)

- Wired `project_android_application_graph` into the CLI and MCP surfaces (116 →
  117 tools).
- Added an env-gated bypass (`REA_WINDOWS_GHIDRA=1`) that forced
  `hasWindowsNativeAuthority()` to true and let `PrivateRuntimeRoot.create()`
  proceed, enabling Ghidra analysis on Windows.

The bypass worked, but it disabled REA's process-isolation guarantees
(no Job Object ownership, no private runtime DACL, no reparse-safe path
admission). It was acceptable only on a personal machine with trusted targets.

## Why 6.5.0 supersedes both

- **Android is fully wired and broader.** 6.5.0 exposes seven Android CLI
  commands and the matching MCP tools (`inspect_android_package`,
  `inspect_android_class`, `inspect_android_method`, `search_android_classes`,
  `trace_android_references`, `decode_android_resources`,
  `project_android_application_graph`), backed by jadx/apktool.
- **Windows Ghidra is supported without a bypass.** The released Windows
  package bundles a native addon (`native/windows/build/rea-windows-x64.node`)
  that implements the real controls. `rea doctor` reports
  `ghidra-native_authority` verified, and Ghidra analysis runs with process
  isolation intact. See `docs/windows-ghidra-p0.md`.
- **Much larger surface.** 153 CLI commands and 207 MCP tools versus 78/116.

## Result

The 4.0.1 bypass and the Android wiring patch are both retired. The correct
configuration is the stock 6.5.0 package with a supported Node, Ghidra/JDK
paths, and agent registrations pointing at them — see
`docs/windows-local-setup.md`.

## Lesson

Before patching a shipped CLI/MCP package, check the current published version.
Both observed gaps were already resolved upstream, and the upstream Windows
solution is strictly safer than a local security bypass.
