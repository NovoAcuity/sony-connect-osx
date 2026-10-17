# Issue Handoff: Security review of Stetco-lab/sony-connect-osx

- **Origin:** ad-hoc session of 2026-10-17, started from `~/dev` (not a
  repository); operator asked for a security review of
  <https://github.com/Stetco-lab/sony-connect-osx>
- **Transferred:** 2026-10-17
- **Owner:** CTO workstream (jpo-hq/cto)
- **Tracking:** none yet. No backlog item, issue or pull request exists for
  any finding below. Whether to file one, or to report upstream, is the
  operator's call.
- **Status:** review complete, by reading the code only. Nothing was built,
  run or fuzzed. No change was made to the reviewed repository.

## Why a manual review

`/security-review` reviews the pending diff of the current branch in the
current repository. The session's working directory was `~/dev`, which is
not a repository, and the command does not accept a GitHub URL. The
repository was cloned into the session scratchpad and every file was read
by hand instead.

## Scope

- Commit `5d52a78` ("fix: persist custom EQ by sending bands under
  UNSPECIFIED preset id"), the tip of the default branch on 2026-10-17.
- All 20 tracked files: 15 Swift sources under `Sources/SonyConnect/`,
  `Makefile`, `Package.swift`, `Resources/Info.plist`, `README.md`,
  `.gitignore`. 2,351 lines in total.
- All 15 commits in history, grepped for added lines that look like
  committed access keys or signing-key blocks.

The app is a macOS menu-bar controller for Sony WH-1000XM3/4/5 headphones.
It talks to the paired headphones over an RFCOMM channel (Sony's
proprietary SPP service), reads and sets output volume through CoreAudio,
and pauses media through the private `MediaRemote` framework.

## Attack surface

The only untrusted input is the byte stream from the paired headphones over
RFCOMM. There are no network sockets, no inter-process interfaces, no URL
handlers, and no shell-outs from application code. `Info.plist` requests
only the Bluetooth usage permissions.

## Result

No high- or medium-severity vulnerability found. Three low-severity
findings.

### 1. Frame parser buffer has no size limit

- **Where:** `Sources/SonyConnect/SonyPacket.swift:84-101`,
  `SonyFrameParser.feed`
- **What:** in the `readingBody` state, every byte that is not an end
  marker (`0x3C`), an escape (`0x3D`) or a start marker (`0x3E`) is
  appended to `buffer`. Nothing caps its length. A stream that sends a
  start marker and never an end marker grows the buffer until the process
  is killed.
- **Reachability:** low. The app opens RFCOMM only to an already-paired
  device whose name matches `SupportedDevices.nameHints`. Exploitation
  needs headphones running malicious firmware, or an attacker who holds the
  pairing link key and impersonates them. Bluetooth Classic throughput
  also makes the growth slow.
- **Evidence status:** established by reading the code. Not triggered in a
  test.
- **Fix:** reset the parser and drop the frame when `buffer.count` exceeds a
  cap (for example 4096 bytes; the largest real payload seen in the code is
  the EQ capability list).

### 2. Log file grows without limit and records all packet traffic

- **Where:** `Sources/SonyConnect/FileLogger.swift`;
  `Sources/SonyConnect/BluetoothClient.swift:174` (tx) and `:249` (rx)
- **What:** `FileLogger` appends to `~/Library/Logs/SonyConnect.log` with no
  rotation. Every sent and received packet is hex-dumped.
- **Impact:** low. The content is headphone settings, battery level and
  device name. The cost is disk use over time and a small privacy surface.
- **Fix:** rotate or cap the file, or move packet dumps to `os.Logger` at
  `.debug` level, where the unified logging system manages retention.

### 3. Build signing hides failures

- **Where:** `Makefile:18`
- **What:** `codesign --force --sign - --entitlements /dev/null $(APP_BUNDLE) 2>/dev/null || true`
  produces an ad-hoc signature without hardened runtime, and discards any
  signing error.
- **Impact:** acceptable for a local build. Not acceptable for a
  distributed `.app`.
- **Fix:** sign with a Developer ID identity and `--options runtime` for
  distribution, and let a signing failure fail the build.

## Checked, no issue found

- **Payload parsers** in `Sources/SonyConnect/HeadphonesController.swift`
  (`parseGsCapability`, `parseEqCapability`, `parseEqParam`, `parseNcasm`,
  `parseBattery`, `parseSystem`, and the touch-sensor cases in `interpret`):
  every device-supplied length or index has a bounds check before use.
- **Frame decode** (`SonyFrameParser.decodeBody`): checks that the declared
  length equals the actual body length before slicing, and verifies the
  checksum.
- **Outgoing writes** (`BluetoothClient.send`): `UInt16(buffer.count)` would
  trap above 65,535 bytes, but every outgoing payload is built by the app;
  the largest is the EQ band list, at most 4 + 255 bytes.
- **Private framework load** (`MediaController`): `MediaRemote` is loaded
  from the fixed path `/System/Library/PrivateFrameworks`, which System
  Integrity Protection guards.
- **Device selection:** only paired devices are considered; an unpaired
  device with a matching name is never opened.
- **Committed access keys:** the history grep matched nothing in any of the
  15 commits.

## Limits

- Static reading only. The parser was not fuzzed and the app was not run.
- Swift memory safety removes most of the classic parser bug classes; the
  review did not cover IOBluetooth or CoreAudio internals.
- The clone used for the review lived in the session scratchpad and is
  discarded with it. To reproduce, clone
  `https://github.com/Stetco-lab/sony-connect-osx` and check out `5d52a78`.

## Open decisions for the operator

1. Whether to report findings 1-3 upstream to Stetco-lab, and in what form
   (issue or pull request).
2. Whether any finding needs a cto backlog item. None is filed.
