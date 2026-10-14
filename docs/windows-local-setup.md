# REA on Windows x64 — local setup notes

Practical, machine-specific setup notes for running REA (`rea-agents`) on a
Windows x64 workstation, including Ghidra-native analysis and the Android stack.
These notes describe one working configuration; they are not a replacement for
the upstream docs (`docs/windows-ghidra-p0.md`, `docs/android-analysis.md`,
`docs/installation.md`).

## 1. Prerequisites

| Component | Requirement | Notes |
|---|---|---|
| **Node.js** | `^22.19.0 \|\| ^24.11.0 \|\| >=26.0.0` | **Node 25.x is not supported.** An odd-numbered non-LTS line is skipped by REA's `engines`. Use Node 24 LTS. |
| **JDK** | 64-bit, JDK 21+ (no maximum) | Needed by Ghidra. Install separately — REA never downloads Java. |
| **Ghidra** | 12.1.x (verified with 12.1.4) | Extract a release; point `GHIDRA_INSTALL_DIR` at its root. |
| **jadx-headless-mcp JAR** | 0.7.1, caller-supplied | **Optional.** Only for the jadx-backed Android provider. REA does not ship or download it. |
| **Hopper** | macOS/Linux only | Not available on Windows. |

### Why Node 24 matters

REA's `engines` field refuses Node 25. With an unsupported Node the MCP server
still starts and can analyse targets, but `rea doctor` reports `node` as
`missing_dependency` and `rea setup` refuses to run
(`"Install Node.js 22.x (>=22.19), 24.x (>=24.11), or 26+ and rerun setup."`).
Installing a supported Node and pointing the launcher/registration at it fixes
doctor and unblocks setup.

## 2. Environment variables

```bat
set "GHIDRA_INSTALL_DIR=C:\bin\ghidra_12.1.4"
set "JAVA_HOME=C:\bin\jdk21"
set "REA_LOG_LEVEL=info"
REM jadx-backed Android provider (optional, but required for APK analysis):
set "REA_JADX_MCP_JAR=C:\bin\jadx-headless\jadx-headless-mcp-0.7.1-all.jar"
REM IDA provider only (optional):
REM set "REA_IDA_MCP_CONFIG=C:\path\to\ida-mcp-registration.json"
```

`GHIDRA_INSTALL_DIR` must be absolute. `JAVA_HOME` is optional when `java` and
`javac` resolve from `PATH`, but pinning a JDK 21 avoids picking up an older JDK.

## 3. Launcher

A small batch shim keeps the environment consistent for every entry point
(CLI and MCP):

```bat
@echo off
REM REA launcher — pins the supported Node and the Ghidra/JDK/jadx paths.
REM Node 24 LTS is required: REA's engines reject the system Node 25.x.
set "GHIDRA_INSTALL_DIR=C:\bin\ghidra_12.1.4"
set "JAVA_HOME=C:\bin\jdk21"
set "REA_LOG_LEVEL=info"
set "REA_JADX_MCP_JAR=C:\bin\jadx-headless\jadx-headless-mcp-0.7.1-all.jar"
"C:\bin\node24\node.exe" "C:\bin\rea-agents\scripts\rea.mjs" %*
```

Usage:

```bat
rea doctor
rea mcp doctor
rea providers
```

## 4. Agent registration (MCP)

Register REA so the agent spawns it through the pinned Node with the Ghidra
environment attached. Example for a JSON `mcpServers`-style config:

```json
{
  "mcpServers": {
    "rea": {
      "command": "C:\\bin\\node24\\node.exe",
      "args": ["C:\\bin\\rea-agents\\scripts\\rea.mjs", "mcp"],
      "env": {
        "GHIDRA_INSTALL_DIR": "C:\\bin\\ghidra_12.1.4",
        "JAVA_HOME": "C:\\bin\\jdk21",
        "REA_LOG_LEVEL": "info"
      }
    }
  }
}
```

Then `rea setup --yes --all-detected` can refresh the other detected clients'
registrations and the bundled skill. Re-apply the explicit `command`/`env`
above to any client whose config REA does not manage, or that must use the
pinned Node.

## 5. Windows native controls (no bypass needed)

Since the released Windows package bundles a native addon
(`native/windows/build/rea-windows-x64.node`), REA establishes Job Object
ownership, a private runtime DACL, and reparse-safe handle admission by itself.
`rea doctor` should show:

```
- name: ghidra-native_authority
  detail: "Verified Windows native Job Object, private DACL, and reparse-safe handle controls; local NTFS only."
```

No environment flag or degraded mode is required, and none should be
introduced: the native controls are what make Windows Ghidra analysis safe.
Boundary limits (Windows 10+ x64, fixed local NTFS, native PE only, no
Ghidra GUI/annotation state) are documented in `docs/windows-ghidra-p0.md`.

## 6. Verification

```bat
rea mcp doctor
```
Expect `healthy: true`, `server.version` matching the package, and
`inventory:tools` `expected == observed` with no `missing`/`unexpected`.

```bat
rea doctor
```
`healthy: false` is expected on a workstation that intentionally omits the
optional engines. Review the individual checks — `hopper` (macOS/Linux only),
`ida-registration` (optional), and `jadx` (needs a caller-supplied JAR) will
report as missing without affecting the Ghidra or artifact providers.

End-to-end Ghidra check on a PE:

```bat
rea inspect C:\Windows\System32\notepad.exe
```
Expect a `rea-workflow` evidence record whose `analysis_profile` names Ghidra
12.1.4, plus populated segments / procedure / string counts. First analysis
takes a few minutes (Ghidra import + auto-analysis).

## 7. Android

The released package exposes the full Android CLI/MCP surface:

```
decode-android-resources      inspect-android-class
inspect-android-method        inspect-android-package
project-android-application-graph
search-android-classes        trace-android-references
```

These are backed by jadx (and apktool for resource decoding).

### Provider JAR

Set `REA_JADX_MCP_JAR` to a caller-supplied `jadx-headless-mcp` **0.7.1** JAR.
REA does not download or install it, and it verifies the JAR's bridge classes
(`com/atxx/jhmcp/JadxSession`, `com/atxx/jhmcp/SessionHolder`,
`jadx/api/JadxDecompiler`, `com/google/gson/Gson`,
`io/modelcontextprotocol/kotlin/sdk/server/Server`) before use.

Known-good source (community fork with the 0.7.1 JAR asset):

```
https://github.com/1013503897/jadx-headless-mcp/releases/tag/v0.7.1
asset: jadx-headless-mcp-0.7.1-all.jar
sha256: 6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75
```

Note: `Samael-Z/jadx-headless-mcp` publishes 0.7.0 as a native `jadx-mcp.exe`
and v1.x as a `jadx-headless-mcp-v2.jar` with a different package layout; the
`com/atxx/jhmcp` bridge classes required by REA 6.5.0 are in the 0.7.1
`-all.jar` above.

Verify and configure:

```bat
REM checksum
certutil -hashfile jadx-headless-mcp-0.7.1-all.jar SHA256
REM configure
set "REA_JADX_MCP_JAR=C:\bin\jadx-headless\jadx-headless-mcp-0.7.1-all.jar"
rea doctor
```

`rea doctor` should then report:

```
- name: jadx
  ok: true
  detail: "JADX MCP bridge JAR validated at <path>; its path is recorded for agent registrations."
```

Smoke test:

```bat
rea inspect-android-package C:\path\to\app.apk
```
Expect a `jadx`-provider evidence record with `package_name`, `version_name`,
`min_sdk`, `target_sdk`, `permissions`, `class_count`, and `resource_count`.

Without the JAR, Android operations report `capability_unavailable` and the
rest of REA continues to work.

## 8. Common pitfalls

- **Node 25 installed** → `doctor` shows `node: missing_dependency`; `setup`
  refuses to run. Point the launcher/registration at Node 24 LTS.
- **`GHIDRA_INSTALL_DIR` unset** → Ghidra availability is `not_configured`.
  It must be absolute and point at the extracted release root.
- **Wrong JDK** → Ghidra reports an unsupported Java version; pin `JAVA_HOME`
  at JDK 21+.
- **Missing native addon** → a development tarball built without the native
  lane reports Windows native controls unavailable. Use the released package;
  do not substitute a security bypass.
- **Stale client registrations** → `doctor` flags `registration:<client>`
  drift (e.g. a registration still requesting an old version). Run
  `rea setup --client <name>` (or `--all-detected`) and restart the client.
