# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What idb is

`idb` (iOS Development Bridge) automates iOS Simulators and Devices. It is split into two processes that
talk over gRPC:

- **`idb_companion`** — a macOS server (Swift + Objective-C) that drives a single target (simulator or
  device). It wraps Apple's private frameworks (CoreSimulator, SimulatorKit, etc.) to expose
  functionality Xcode keeps internal.
- **Python client** (`idb/`) — the `idb` CLI and library that can run anywhere and connects to one or more
  companions remotely. Published to PyPI as `fb-idb`.

The gRPC contract between them lives in `proto/idb.proto`. It is the source of truth: each `rpc` maps to one
companion method handler and one client method.

> This repo is a fork that builds with **XcodeGen + xcodebuild** via `build.sh`. Upstream `facebook/idb`
> builds with Buck; the `project.yml` comments reference the original `defs.bzl` setup. When in doubt about
> build mechanics, trust `build.sh` and `project.yml`, not Buck files.

## Build, test, and codegen

Everything goes through `./build.sh` (prerequisites: Xcode 14+, `brew install xcodegen protobuf
swift-protobuf`). It regenerates the Xcode projects from `project.yml` on **every** `build`/`test`, so do not
hand-edit `.xcodeproj` files — edit `project.yml` (root frameworks) or `idb_companion/project.yml`
(the companion) instead.

```bash
./build.sh build                     # frameworks + shims + idb_companion (Release)
./build.sh build frameworks          # just the four macOS frameworks
./build.sh build FBControlCore       # a single framework
./build.sh build idb_companion       # the gRPC server (builds its framework deps first)
./build.sh test                      # all framework test bundles
./build.sh test FBSimulatorControl   # one framework's tests
./build.sh generate                  # regenerate .xcodeproj from project.yml only
./build.sh generate-proto            # regenerate IDBGRPCSwift/ from proto/idb.proto
```

The companion binary lands at `Build/Products/Release/idb_companion`. CI (`.github/workflows`, `.circleci`)
only deploys the docs website — there is no automated build/test gate, so verify builds locally.

There is no single-test target in `build.sh`; to run one test, build the framework then invoke
`xcodebuild ... -only-testing:<Bundle>/<Class>/<method>` against `FBSimulatorControl.xcodeproj`, or run the
test from Xcode after `./build.sh generate`. Tests need a working Xcode/simulator environment; many are
integration tests, not pure units.

Python client: `FB_IDB_VERSION=0.0.0 python3 setup.py build` (the build step regenerates `idb/grpc/idb_grpc.py`
from the proto and rewrites its imports). Client entry point is `idb.cli.main:main`.

## Syncing with upstream (facebook/idb)

This fork has two remotes: `origin` (your fork, `TobiasLaross/idb`, push enabled) and `upstream`
(`facebook/idb`, fetch-only — its push URL is set to `no_push` on purpose). To pull in new commits from
Facebook:

```bash
git fetch upstream                       # fetch facebook/idb without merging
git log --oneline main..upstream/main    # preview what's new upstream
git checkout main
git merge upstream/main                  # or: git rebase upstream/main
git push origin main                     # publish the synced main to your fork
```

Keep using `git merge`/`git rebase` from `upstream/main` rather than re-cloning. Expect conflicts in the
files this fork added or diverged on — chiefly `build.sh`, `project.yml`, and `idb_companion/project.yml`
(upstream builds with Buck and has no XcodeGen build path). When resolving, preserve this fork's
XcodeGen/xcodebuild setup. After syncing, run `./build.sh build` to confirm the build still works before
pushing, since upstream changes are not exercised by this fork's (website-only) CI.

## Framework layering (Objective-C)

The macOS frameworks form a dependency chain — lower layers never depend on higher ones:

- **`FBControlCore`** — foundation shared by everything: process/task launching, file ops, crash log
  parsing, codesigning, and the async primitives. Has no idb-specific knowledge.
- **`XCTestBootstrap`** — running XCTest bundles on targets (test hosts, result streaming).
- **`FBSimulatorControl`** — simulator targets via CoreSimulator/SimulatorKit.
- **`FBDeviceControl`** — physical device targets via the device support frameworks.

These are usable standalone (the README pitches them as independent frameworks). Simulator/device commands
are exposed as Objective-C protocols (`FBSimulatorControl/Commands`, `FBDeviceControl/Commands`).

### Async: FBFuture

Asynchronous work in the ObjC layer uses **`FBFuture`** (`FBControlCore/Async/`), a promise type, plus
`FBFutureContextManager` for resources with scoped acquire/release. Swift code bridges to `async/await`
through `AsyncFBFutureBridge` (`FBControlCore/Async/`) and the helpers in `CompanionLib/BridgeFuture/`
(`FBFutureContextBridge`, `BridgeQueues`). When calling framework APIs from Swift, await the bridge rather
than blocking on `FBFuture+Sync`.

## Companion (Swift)

The Swift layer sits on top of the frameworks:

- **`IDBCompanionUtilities`** — small Swift utilities (e.g. `@TaskLocal` helpers) with no framework deps.
- **`CompanionLib`** — the bridge between gRPC handlers and the frameworks. `FBIDBCommandExecutor.swift` is
  the central facade: most RPCs ultimately call a method on it, and it dispatches to the right framework
  command. Request value types live in `CompanionLib/Request/`.
- **`idb_companion`** — the executable. `main.swift` parses the large CLI surface (modes like `--udid`,
  `--boot`, `--list`; options like `--grpc-port`, `--grpc-domain-sock`, `--tls-cert-path`) and starts
  `GRPCSwiftServer`. `CompanionServiceProvider` (in `SwiftServer/`) implements the generated service and
  routes each RPC to a file in `SwiftServer/MethodHandlers/` (one handler per RPC, ~43 of them).
  `ValueTransformers/` convert between proto messages and framework types; `Interceptors/` add
  cross-cutting concerns.

`IDBGRPCSwift/` holds the generated `idb.pb.swift` / `idb.grpc.swift` (checked in; regenerate with
`./build.sh generate-proto`, which builds `protoc-gen-grpc-swift` from grpc-swift 1.x).

## Adding or changing an RPC

This is the most common cross-cutting change. The flow touches both processes:

1. Edit `proto/idb.proto`.
2. Regenerate Swift: `./build.sh generate-proto` (updates `IDBGRPCSwift/`). Regenerate the Python stubs via
   the `setup.py build` step.
3. Companion side: add/adjust a handler in `idb_companion/SwiftServer/MethodHandlers/`, wire it in
   `CompanionServiceProvider.swift`, add any proto↔framework conversion in `ValueTransformers/`, and add the
   underlying logic to `FBIDBCommandExecutor` (and the relevant framework command if it's new behavior).
4. Client side: implement the call in `idb/grpc/client.py` and surface it as a command under
   `idb/cli/commands/`.

## Build-system gotchas

`project.yml` encodes several non-obvious workarounds — read its comments before changing build settings:

- **Private Apple frameworks** (CoreSimulator, SimulatorKit, AXRuntime, SimulatorApp) are reverse-engineered
  headers under `PrivateHeaders/`, wrapped as Clang modules via `module.modulemap` files and fed to Swift
  with `-fmodule-map-file`. `FRAMEWORK_SEARCH_PATHS` points at `$(DEVELOPER_DIR)/Library/PrivateFrameworks`.
  CoreSimulator is weak-linked against a `.tbd` stub because it is loaded at runtime.
- **Mixed-language headers**: frameworks add `$(OBJECT_FILE_DIR_normal)/$(CURRENT_ARCH)` to
  `HEADER_SEARCH_PATHS` so `.m` files can find the generated `*-Swift.h`.
- **Xcode 26 workarounds** in `build.sh`: `SWIFT_ENABLE_EXPLICIT_MODULES=NO` for the companion (swift-nio
  module resolution), `-skipMacroValidation` + sandbox-disable flags for Swift macro plugins, and a `sed`
  patch that strips `IDBGRPCSwift.framework` from the companion's Embed Frameworks phase to avoid a
  duplicate-output error.
- **Build directory**: defaults to `./Build` (absolute path, shared SPM products dir); falls back to
  `/tmp/idb-build-*` via a symlink on filesystems without xattr support (e.g. EdenFS).

## Conventions

- Objective-C class/file prefix is `FB`. Companion-specific Swift request types are prefixed `FB` too
  (e.g. `FBXCTestRunRequest`).
- License header (MIT, "Meta Platforms, Inc. and affiliates") goes at the top of every source file.
- `// @oss-disable` / `// @oss-enable` markers in Swift gate Meta-internal lines from the open-source build —
  leave them intact.
