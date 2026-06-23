# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Swift Build is a high-level build system based on [llbuild](https://github.com/swiftlang/swift-llbuild) with strong Swift support. It is used by SwiftPM, Xcode, and Swift Playground. It runs as a **separate service process** (`SWBBuildService`) per client; clients and the service communicate via serialized messages over a pipe. llbuild is linked into the service's address space (not a separate process) and the two communicate bidirectionally.

## Build, Test, and Format

This is a Swift Package (`Package.swift`, tools version 6.0). It also has a CMake build used only for bootstrapping/validation.

```bash
swift build                              # build the package
swift test                               # run the full test suite
swift test --no-parallel                 # how CI runs tests (more deterministic)
swift test --filter SWBUtilTests         # run one test target
swift test --filter SWBUtilTests.CacheTests   # run one suite
swift test --filter WebAssemblyIntegrationTests   # run an SDK integration test target
```

Formatting / lint (CI enforces both):
- `swift format` is configured by `.swift-format` (4-space indent, `lineLength` effectively unlimited at 10000). The repo targets the Swift toolchain in `.swift-version` (6.2.0).
- `.github/scripts/format-check.sh` rejects trailing whitespace and missing final newlines on all tracked files. `.editorconfig` enforces LF, UTF-8, final newline, 4-space indent (2 for YAML/CMake).
- New source files need the Apache license header — see `.license_header_template`.

### Toolchain selection gotcha (swiftly vs `.swift-version`)
If `swift` on `PATH` is the **swiftly shim** (`~/.swiftly/bin/swift`), it reads this repo's `.swift-version` pin (`6.2.0`) and refuses to run when that toolchain isn't installed (e.g. `launch-xcode` silently never launches Xcode). Fix: prefix commands with **`xcrun`** (`xcrun swift package …`) to use the selected Xcode's bundled toolchain directly (it ignores `.swift-version`); or `swiftly install 6.2.0`; or temporarily rename `.swift-version`. A newer Xcode toolchain (e.g. Swift 6.4) builds the repo fine.

### Testing in clients (SwiftPM / Xcode / xcodebuild)
- **SwiftPM:** enabled by default in nightly snapshots; `--build-system swiftbuild` in Swift 6.2/6.3. To co-develop against a local SwiftPM checkout: `SWIFTCI_USE_LOCAL_DEPS=1 swift build --package-path /path/to/swiftpm`.
- **Xcode:** `swift package --disable-sandbox launch-xcode` launches the selected Xcode wired to your modified build service.
- **xcodebuild:** `swift package --disable-sandbox run-xcodebuild -- <args>` (args after `--` are forwarded).
- **Debugging the service:** attach to process name `SWBBuildServiceBundle` (e.g. Xcode → Debug → Attach to Process by PID or Name), then trigger a build. For the SwiftPM path, the build runs in-process: `swift run --debugger swift-build ...`.

### Environment variables
- `SWIFTCI_USE_LOCAL_DEPS=1` — resolve `swift-driver`, `swift-system`, `swift-argument-parser`, `swift-tools-protocols`, `llbuild` from sibling directories (`../`) instead of fetching from git.
- `SWIFTBUILD_STATIC_LINK=1` — static build; drops all test and `*TestSupport` targets.
- `SWIFTBUILD_LLBUILD_FWK=1` — link llbuild as a prebuilt framework instead of the package product.
- `SAVE_TEMPS=1` — keep the temporary directories tests create (for inspection).

## Architecture

Swift Build is a vertical stack of framework targets that lean heavily on protocols for layering. From lowest to highest (full detail in `SwiftBuild.docc/Architecture/build-system-architecture.md`):

- **SWBUtil / SWBCSupport / SWBLibc / SWBCLibc** — general utilities (covers for Foundation/POSIX, `OrderedSet`, `PropertyList`, `FSProxy`, `Process`); not Swift-Build-specific. SWBCSupport is the small C/Objective-C bridge.
- **SWBLLBuild** — thin Swift shim over the llbuild C API.
- **SWBMacro** — the build-setting evaluation engine. Settings are typed; string expressions are parsed once and evaluated against a `MacroEvaluationScope`.
- **SWBCore** — types shared across higher layers: the **Settings** binding engine, the **Project Model** (the "PIF" — Project Interchange Format, both the serialized client→service form and the in-memory model), **Specifications** ("specs", data-driven definitions of tools/file types, see below), **Platforms & SDKs**, and core task-planning types (nodes, tasks, build request).
- **SWBProtocol** — shared message definitions for client↔service communication.
- **SWBTaskConstruction** — "task planning." Takes a `WorkspaceContext` + `BuildRequest` and produces a `BuildPlan` (the dependency graph). `ProductPlanner` creates a `ProductPlan` per target and runs `TaskProducer`s (roughly one per build phase) in parallel, aggregating serially. Tool specs in SWBCore create the individual tasks. Most of the graph is static; some inputs (Clang/Swift compilation) request dynamic work.
- **SWBTaskExecution** — what actually runs during a build. Holds the `BuildDescription` + build manifest (the JSON file llbuild consumes), and **Task Actions** (commands that run *in Swift Build's address space* rather than as subprocesses). `BuildDescriptionManager` caches build descriptions in-memory/on-disk, so **task construction may be skipped entirely** for a given build.
- **SWBBuildSystem** — coordinates planning + execution; `BuildManager` owns build operations, `BuildOperation` is a single build.
- **SWBServiceCore** — reusable generic service infrastructure (decoupled from Swift Build specifics).
- **SWBBuildService** — the actual service logic; manages one session per open workspace and calls back to clients via `ClientExchangeDelegate`.
- **SWBBuildServiceBundle** — the executable bundle implementing the service over a pipe (thin shim on SWBBuildService).
- **SwiftBuild** — the public client-facing API framework.
- **swbuild** — the CLI for exercising the public API and service.

### Platform plugins
Per-platform logic lives in `SWB*Platform` targets (`SWBApplePlatform`, `SWBAndroidPlatform`, `SWBWindowsPlatform`, `SWBQNXPlatform`, `SWBWebAssemblyPlatform`, `SWBGenericUnixPlatform`, `SWBUniversalPlatform`). These act as plugins: `Package.swift` injects them as dependencies of `SWBBuildService` and `SWBTestSupport` (there is no true SwiftPM plugin-target mechanism here), and `USE_STATIC_PLUGIN_INITIALIZATION` is defined so they register statically. Each ships `Specs` resources.

### Specs
Many tools, file types, and build settings are defined data-driven as "xcspecs" rather than in code (`Specs` resource bundles in SWBCore and the platform targets). See `SwiftBuild.docc/Core/xcspecs.md`.

## Command line tool: `swbuild`

`swbuild` (`Sources/swbuild`, with logic in the `SwiftBuild` module's `SWBTerminal.swift` / `ConsoleCommands/`) is a thin REPL/console front-end over the build-service console — **not** a flag-driven tool like `xcodebuild`. It is primarily a testing/bring-up tool.

- **Dispatch:** the first token of a command line must be a registered subcommand *name* (looked up via `commandLine.first` in `SWBBuildServiceConsole.sendCommand`). A leading flag fails with `unknown command: '...'`. With no args, `swbuild` starts a REPL. Multiple commands can be batched on one line separated by a bare `--`.
- **`build` subcommand** (`ConsoleCommands/SWBServiceConsoleBuildCommand.swift`) recognizes *only*: `--action <name>`, `--target <name>` (repeatable) / `--allTargets`, `--configuration <name>`, `--derivedDataPath <path>`, `--buildParametersFile <path>`, `--buildRequestFile <path>`, plus exactly **one positional** = the container path (`.xcworkspace`/`.xcodeproj`). Any other `--foo` → `error: unknown argument`. Other registered commands include `createBuildDescription`, `prepareForIndex`, `createXCFramework`.
- **There is no `--platform` flag.** Platform/SDK/arch is the *run destination*, carried inside the build parameters (`SWBBuildParameters.activeRunDestination`, an `SWBRunDestinationInfo`). The only way to supply it on the CLI is a JSON `--buildParametersFile` (or a full `--buildRequestFile`). Without a run destination the build fails to infer a platform.
  - The `toolchainSDK` run destination encodes/decodes its keys **flat** (not nested under `buildTarget`): `platform`, `sdk`, `sdkVariant`, `targetArchitecture`, `supportedArchitectures`, `disableOnlyActiveArch`, optional `hostTargetedPlatform`. Example for iOS Simulator: `{"platform":"iphonesimulator","sdk":"iphonesimulator","sdkVariant":"iphonesimulator","targetArchitecture":"arm64","supportedArchitectures":["arm64","x86_64"],"disableOnlyActiveArch":false}`. Run-destination presets for all platforms live in `Sources/SWBTestSupport/RunDestinationTestSupport.swift`.
- **Precedence:** `--buildParametersFile` is loaded first; `--action` then overrides `action`, but `--configuration` is ignored if the file already sets `configurationName`. `--buildRequestFile` (a serialized `SWBBuildRequest`) overrides *everything* (parameters and targets) — useful for replaying an exact Xcode build. `--derivedDataPath` is highest precedence for the arena (maps to `DerivedData/{Products,Intermediates.noindex}`).
- **Output:** build events stream as length-prefixed JSON (same framing as the Swift compiler's structured output), not human-readable text.
- **Reproducing Xcode faithfully:** prefer `swift package --disable-sandbox run-xcodebuild -- <args>` (or `launch-xcode`), which lets real Xcode/`xcodebuild` construct the PIF and run destination against your modified service, rather than hand-authoring parameters for `swbuild build`.

## Static vs. dynamic linking of SwiftPM package products

How a SwiftPM library product (`.automatic`/`.static`/`.dynamic` linkage) ends up statically vs. dynamically linked (i.e. embedded `.framework`/dylib or not):

- **The automatic choice is made upstream, not here.** Swift Build only *reads* the boolean `PACKAGE_BUILD_DYNAMICALLY` on the package target (`Sources/SWBCore/DependencyResolution.swift:579-583`, building `dynamicallyBuildingTargets`). The macro is declared at `Sources/SWBCore/Settings/BuiltinMacros.swift:956` and is **never set/defaulted anywhere in this repo** — its value arrives in the PIF generated by the client (SwiftPM's `PIFBuilder` / Xcode). So a dynamic↔static change across Xcode/SwiftPM versions originates in SwiftPM's PIF generation. The user-facing lever is the package manifest: `.library(name:, type: .dynamic|.static, targets:)`.
- **The one dynamic behavior Swift Build itself drives** is the diamond-problem resolver in `Sources/SWBTaskConstruction/ProductPlanning/ProductPlan.swift:913-986`: if a *static* package library would be linked into more than one top-level dynamic binary, it is auto-switched to its dynamic variant via `dynamicTargetVariantGuid` (`Target.swift:178`; PIF key `PIFKey_Target_dynamicTargetVariantGuid`). This only goes **static → dynamic**, never the reverse. Gated by `DISABLE_DIAMOND_PROBLEM_DIAGNOSTIC` (`BuiltinMacros.swift:632`). Dynamic Mach-O types considered: `mh_execute`, `mh_dylib`, `mh_bundle` (`ProductPlan.swift:198`).
- **Related macros:** `MACH_O_TYPE` (`staticlib` vs `mh_dylib`/`mh_bundle` — the actual output format), `PRODUCT_TYPE` (`com.apple.product-type.library.static` / `.dynamic` / `.objfile`), `PACKAGE_TARGET_NAME_CONFLICTS_WITH_PRODUCT_NAME`. Static package targets surface as `com.apple.product-type.objfile`.
- **Distinct from mergeable libraries** (`AUTOMATICALLY_MERGE_DEPENDENCIES` etc., see `SwiftBuild.docc/TaskConstruction/mergeable-libraries.md`) — that feature merges dylibs into another binary; it is not the static/dynamic linkage decision.
- Background on why a package builds differently per dependent: `PackageProductTarget` is a "build to order" target whose settings are propagated from its dependents (`SwiftBuild.docc/Core/target-specialization.md`).
- **SwiftPM 6.3 → 6.4 behavior change (verified, incl. intent):** the dynamic variant of an `.automatic` library product is generated by SwiftPM's PIF builder, not swift-build. In SwiftPM 6.3, `Sources/SwiftBuildSupport/PackagePIFProjectBuilder+Products.swift` unconditionally created a `.dynamic` variant (setting `dynamicTargetVariantId`, which becomes swift-build's `dynamicTargetVariantGuid`) for every automatic library product with source targets — enabling an embedded `.framework`. SwiftPM 6.4 (commit `da7e637`, PR swiftlang/swift-package-manager#9831, "Fixes windows linking against a .dynamic product") added a gate `&& pifBuilder.createDynamicVariantsForLibraryProducts` to that condition.
  - **Intent (from PR review, decisive):** the disabling is *deliberate*, not a Windows side effect, but it was **scoped to the SwiftPM CLI build system only**. The flag **defaults to `true`**; only `SwiftBuildSupport/SwiftBuildSystem.swift` (the `swift build`/`test`/`run` path) explicitly passes `false`. Design quote: "clients of the PIFBuilder can opt into, and the SwiftPM CLI opts out of, rather than inferring it based on platform." Motivation: once dynamic products' direct modules are built without `-static`, auto-creating dynamic variants everywhere produced "too many exported symbols when linking."
  - The genuinely Windows-gated part of the same PR is separate: `settings[.SWIFT_COMPILE_FOR_STATIC_LINKING, .windows] = "NO"` for a dynamic product's direct source modules (`PackagePIFProjectBuilder+Modules.swift`).
  - **`da7e637` is NOT what flips an Xcode app dependency dynamic→static.** That gate only controls whether the dynamic variant is *created*; it still is for Xcode (PIFBuilder clients inherit the `true` default). Verified by dumping the Xcode 27 PIF (`xcrun xcodebuild -dumpPIF out.pif -workspace X.xcworkspace -scheme S`): the static package target still carries a `dynamicTargetVariantGuid` and `…-dynamic` framework variants still exist.
  - **The real Xcode 26.5 → 27.0 change (empirically proven by building the same workspace with both Xcodes):** the *default linkage policy* for SwiftPM library products flipped from **dynamic embedded framework** to **static**. Evidence from build logs: Xcode 26.5 builds `PackageFrameworks/<Product>.framework` and embeds it (and even trips a project's "unexpected framework installed" validation); Xcode 27.0 emits no framework, instead partial-linking the product to a single `<Product>.o` (`ld -r` + `lipo`) that links statically into the app. Both Xcodes use the same swift-build-based system — this is a default-policy change, **not** a build-system migration and **not** a diamond effect (single consumer in both).
  - **The decision is NOT a value in the PIF.** Verified by diffing both PIFs: in both, the app depends on the same `packageProduct` (build-to-order) edge, and the PIF supplies *both* a static library target and a dynamic framework variant (`dynamicTargetVariantGuid`) with no linkage boolean (`MACH_O_TYPE`/`PACKAGE_BUILD_DYNAMICALLY`) on the edge. swift-build picks static vs dynamic when it resolves that `packageProduct` at task construction (`Sources/SWBTaskConstruction/ProductPlanning/ProductPlan.swift`). The default in that resolution differs between the swift-build compiled into each Xcode. (A cosmetic PIF difference exists — package target product type `com.apple.product-type.objfile`+`MACH_O_TYPE=mh_object` in 26.5 vs `org.swift.product-type.library.static` in 27.0 — but both are *static* representations and the app links the product, not the target, so it is not the cause.) The open-source SwiftPM 6.3 vs 6.4 `SwiftBuildSupport` `.staticLibrary → .commonStaticArchive` mapping is *identical*, so the change is not in that path. Reflects the broader Apple direction (fewer embedded dylibs / faster launch).
  - **Inspecting linkage decisions:** (1) `xcrun xcodebuild -dumpPIF <out> -workspace <ws> -scheme <scheme>` then grep target `productTypeIdentifier` (`org.swift.product-type.library.static`/`com.apple.product-type.objfile` = static vs `com.apple.product-type.framework` = dynamic) and `dynamicTargetVariantGuid`; (2) definitive: build and check for `PackageFrameworks/<Product>.framework` + an embedded `.app/Frameworks/<Product>.framework` (dynamic) vs a `<Product>.o` partial-link (static). To A/B across toolchains, set `DEVELOPER_DIR=/path/to/Xcode-N.app/Contents/Developer` per build.
  - User-facing fix to force dynamic regardless: declare the product `type: .dynamic` in the dependency's `Package.swift` (→ produces a `.framework`). The default static/dynamic selection is not exposed as a build setting/env var/CLI flag.

### The "diamond problem" / package-product variant resolution

This is the swift-build mechanism that decides whether a SwiftPM package product links statically or switches to its dynamic framework variant. There is no DocC page for it; the authoritative source is the code + comments below.

- **Concept:** a package product links **statically by default**. The *diamond problem* is when the same static package target/product would end up linked into **more than one top-level dynamic binary** (e.g. app + app-extension, app + another framework, or a single target reached through multiple products). That would duplicate its code/symbols in the process. To avoid that, swift-build switches the product to its **dynamic framework variant** (the `dynamicTargetVariantGuid` that SwiftPM's PIF generation emits) so there is one shared copy. The switch only ever goes **static → dynamic**, never the reverse.
- **Code (`Sources/SWBTaskConstruction/ProductPlanning/ProductPlan.swift`):** `resolveDiamondProblemsInPackages()` (~794), `checkForDiamondProblemsInPackageProductLinkage()` (~842), `isStaticallyLinkedPackageProduct()` / `isStaticallyLinkedPackageTarget()` (~1043 / ~1061). Top-level-binary boundaries are computed in `Sources/SWBTaskConstruction/ProductPlanning/TopLevelLinkingTargetResolver.swift`. Dynamic Mach-O types considered: `mh_execute`, `mh_dylib`, `mh_bundle` (`ProductPlan.swift:198`).
- **Controls/inputs:** `DISABLE_DIAMOND_PROBLEM_DIAGNOSTIC` (skip detection; `BuiltinMacros.swift:632`), `PACKAGE_TARGET_NAME_CONFLICTS_WITH_PRODUCT_NAME` (diagnostic wording only — *not* a linkage driver; `BuiltinMacros.swift:959`), and the PIF-supplied `dynamicTargetVariantGuid` on the static target.
- **Recent behavior changes (git log on `ProductPlan.swift`):** `8698d7a7` "Fix package target diamond resolution across products" (a package target shared across multiple products could be static on one path and dynamic on another, leaving duplicate copies — introduced `TopLevelLinkingTargetResolver`); `df71c8b2` "Use post-dynamic-linkage target graph for dependency diagnostics". These shifted resolution toward consistent static linking and are the kind of change responsible for the Xcode 26.5→27.0 flip.
- **Empirical notes (verified by building one workspace under both Xcodes — Xcode 26.5 vs 27.0):** the static/dynamic outcome is *not* a value in the PIF (same `packageProduct` edge + both variants present in both Xcodes) and is *not* tied to GUID namespacing, the target/product name clash, or resource bundles. The real driver is the **number of in-process top-level binaries** that link the package product. Example: `RevenueCat` is linked by 3 in-process binaries (the app + `BlackInkUI.framework` + `PuzzleKit.framework`) → a genuine diamond; `OrderedCollections` is linked only by `RSTouchFoundation` in the iOS process (its other consumer `RSFoundation` is the macOS platform variant, not loaded) → no diamond.
- **Observed regression (Xcode 26.5 → 27.0) and its upstream fix:** 26.5 resolved RevenueCat's diamond correctly — one shared embedded `RevenueCat.framework`. 27.0 static-links `RevenueCat.o` into *multiple* in-process binaries (the app **and** `PuzzleKit.framework`) → **duplicate copies of the package's code/state in one process** (breaks runtime type identity for stateful SDKs; not just size). This is a **bug in Xcode 27.0's bundled swift-build**, not an intentional static-by-default policy.
  - **Fix:** PR swiftlang/swift-build#1424 "Fix package diamond resolution by removing package target filtering" (commit `8698d7a7`, 2026-06-11; merged `73bfc081`, 2026-06-15) removes the `packageTargetsToSkip`/target-level-exclusion that left a shared package target static on one path while dynamic on another. It **postdates** the Xcode 27.0 snapshot (build `27A5209h`), which is why 27.0 has the bug; the fix is in swift-build `main`. Independently confirmed in the PR thread ("this PR fixes the duplicated symbols issue I was seeing").
  - **Follow-on caveat:** that PR introduced a Copy Files regression — a dynamic-variant framework product's reference name is `$(EXECUTABLE_NAME)`, resolving to the bare binary path instead of the `.framework` wrapper, so embedding fails with "No such file or directory" (see #1424 comments; proposed `CopyFilesTaskProducer.swift` patch). As of `main`/HEAD `7f55ba80` that follow-on fix had **not** landed — bleeding-edge swift-build fixes the duplication but may hit the embed error for some packages.
  - Couldn't cleanly date when the regression was first introduced (the filtering predates the open-source history; 26.5 still resolved correctly, so manifestation depends on graph topology). Solid: 26.5 correct → 27.0 duplicates → #1424 (post-27) fixes.
  - **`type: .dynamic` is NOT a reliable workaround on Xcode 27 (verified, corrects an earlier assumption).** Declaring the product `type: .dynamic` IS honored (the product builds as a `.framework`), but the bug is at the package-*target* diamond level, *below* the product type — so a dependent framework still static-links the shared package target. Verified on the real workspace via a local-package override: with `RevenueCat` set to `type: .dynamic`, `RevenueCat.framework` was built, yet `PuzzleKit.framework` (`TouchPuzzleKit`) still static-linked `RevenueCat.o`/`RevenueCatUI.o` (read from its `*.LinkFileList`) → duplication persisted, and the build broke a "verify embedded frameworks" run-script. On Apple platforms `type: .dynamic` always yields a `.framework` (not a bare dylib; the dylib choice is the build-system `createDylibForDynamicProducts` flag, not manifest-exposed). Real fixes: a toolchain with PR #1424, or restructure so the package is linked by only one in-process binary.

## Tests

- Tests use **swift-testing** (`import Testing`, `@Suite`, `@Test`), not XCTest. Suites are typically `fileprivate struct`s.
- Each `SWB*` module has a matching `SWB*Tests` target; `*PerfTests` targets hold performance tests.
- **Project tests** are the most important integration-test class. `TaskConstructionTester` plans a build from in-memory test projects (`TestWorkspaces.swift`) and a checker block inspects the resulting tasks (rule info, command line, env, inputs/outputs). `BuildOperationTester` (in SWBBuildSystem tests) actually runs builds and checks results — most run real, not simulated, builds. See `SwiftBuild.docc/Development/test-development-project-tests.md`.
- Most newer tests are scenario-based (one new feature or fixed bug per test) rather than large catch-all tests.
- `SWBTestSupport` / `SwiftBuildTestSupport` provide shared test infrastructure.

## Swift language modes

The package builds with both `.v5` and `.v6` language modes; `swiftSettings(languageMode:)` in `Package.swift` sets per-target mode and a large set of upcoming/future features (`ExistentialAny`, `MemberImportVisibility`, `InternalImportsByDefault`, etc.). When adding a target, match the language mode and settings of its peers. `cxxLanguageStandard` is C++20.

## Documentation

Technical docs live in `SwiftBuild.docc` (DocC). Preview with `xcrun docc preview SwiftBuild.docc` (macOS) or `docc preview SwiftBuild.docc`. Read these before deep work in an unfamiliar subsystem — they cover dynamic tasks, indexing, the Swift driver integration, macro evaluation, target specialization, mutable outputs, mergeable libraries, and discovered dependencies.

## Contributing

This is part of the Swift open source project. Bug reports go to the [swift-build issue tracker](https://github.com/swiftlang/swift-build/issues); follow the [Swift contributing guidelines](https://swift.org/contributing/). Run the full test suite before submitting a PR.
