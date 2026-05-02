---
layout: post
article: true
title: "Exploration C2 0.16: Multi-Arch Windows Builds, ARM64, Donut, CI, and Codex"
date: 2026-05-02
category: c2
series: "Building a Modern C2"
tags: [c2, release-engineering, arm64, windows-internals, ci, codex, donut]
description: "Technical notes on the 0.16 release: multi-architecture Windows builds, ARM64 support, Donut integration work, CI validation, and Codex-assisted low-level engineering."
permalink: /BuildingAModernC2/Release016MultiArchArm64CI.html
---

# Exploration C2 0.16: Multi-Arch Windows Builds, ARM64, Donut, CI, and Codex

Exploration C2 0.16 is mainly a release-engineering and architecture milestone.

The visible result is simple: the Windows implant release now ships separate artifacts for `x64`, `x86`, and `ARM64`. The less visible part is the work required to make that reliable: architecture-specific builds, ARM64 test execution, Donut loader changes, syscall bridge, hardware breakpoint behavior, and a CI pipeline able to prove that the release package is working.

This article is a technical write-up of that work.

## What Shipped

The `0.16.0` implant release introduced Windows release artifacts for three architectures:

- `C2Implant-windows-x64.zip`
- `C2Implant-windows-x86.zip`
- `C2Implant-windows-arm64.zip`

![C2Implant 0.16.0 release assets]({{ "/assets/images/exploration-c2-0-16/c2implant-0.16.0-release.png" | relative_url }})

The `0.16.1` TeamServer release then packaged the server, client, TeamServer modules, and validated implant assets into the final `Release.tar.gz` bundle.

![C2TeamServer 0.16.1 release bundle]({{ "/assets/images/exploration-c2-0-16/c2teamserver-0.16.1-release.png" | relative_url }})

That split matters. The implant repository is responsible for producing architecture-specific Windows beacons and modules. The TeamServer repository is responsible for consuming those artifacts, validating that the expected assets exist, and publishing a complete framework release.

The important change is not only "there is an ARM64 zip now". The important change is that ARM64 became part of the release contract.

## Why ARM64 Changed the Shape of the Release

Adding ARM64 was not just a CMake flag.

The project already had boundaries between the TeamServer, the Windows implant, Windows modules, and third-party loader components. ARM64 made those boundaries concrete. The build matrix had to treat `x64`, `x86`, and `ARM64` as real targets. The Windows ARM build needed a native runner instead of only cross-compilation. Dependencies had to be configured with architecture-specific assembly support in mind. Low-level code had to follow the Windows ARM64 ABI, and loader generation had to emit the right architecture-specific stubs.

Most importantly, the tests had to run before release packaging. Without that, ARM64 would have been a build variant. With it, ARM64 became a supported release target.

This is the kind of change where "it compiles" is not enough. For low-level tooling, the calling convention, stack layout, exception context, and generated loader bytes are part of the product.

## CI Became the Release Gate

The central piece of the 0.16 work was CI.

![C2Implant CI architecture matrix]({{ "/assets/images/exploration-c2-0-16/c2implant-ci-matrix.png" | relative_url }})

The Windows implant workflow now builds and tests the architecture matrix on GitHub Actions. The ARM64 entry uses the `windows-11-arm` runner, which means ARM64 code is not only cross-built from an x64 host. It runs through a native Windows-on-ARM build environment.

At a high level, the CI contract became:

1. Configure the project for the target architecture.
2. Build the implant and modules.
3. Run the CTest suite.
4. Validate that expected release artifacts are produced.
5. Upload architecture-specific packages.

This was particularly important for technical modules such as `inject`, `AssemblyExec`, and `DotnetExec`, where architecture differences are not cosmetic.

For ARM64, that runner was the difference between guessing and knowing. It exposed the practical issues that only appear when the toolchain, ABI, libraries, and tests are all running in the target environment.

The TeamServer release workflow then consumes the implant release assets and validates the final package. That makes the release chain more deterministic:

- `C2Implant` proves that each Windows architecture can build and test.
- `C2TeamServer` proves that the framework release contains the expected implant assets.
- the final archive is published only after validation.

This is release engineering applied to offensive security research tooling: the pipeline is not just automation, it is part of the architecture.

## Donut Work: Making the Loader Path Architecture-Aware

Donut was one of the major technical areas for this release.

```bash
├── Makefile.msvc
├── Makefile_arm64.msvc
├── Makefile_x86.msvc
...
├── loader_exe_arm64.go
├── loader_exe_arm64.h
├── loader_exe_x64.go
├── loader_exe_x64.h
├── loader_exe_x86.go
├── loader_exe_x86.h
```

The integration had to move from an x86/x64-centric assumption toward an explicit architecture model. The fork now carries ARM64-aware build and loader paths: an ARM64 architecture identifier in the API and CLI flow, ARM64 loader header generation, an ARM64 MSVC makefile, architecture-specific loader selection from the implant build system, ARM64-specific ETW return stub bytes, and CI support for the Donut variants used by the implant pipeline.

```c
// mov x1, x30; preserve the caller's return address.
PUT_WORD(pl, 0xAA1E03E1);
// bl after_instance; x30 receives the address of the embedded instance.
PUT_WORD(pl, 0x94000000 | imm26);
PUT_BYTES(pl, c->inst, c->inst_len);
while(arm64_pad-- != 0) {
    PUT_BYTE(pl, 0x00);
}
// mov x0, x30; Windows ARM64 passes the first argument in x0.
PUT_WORD(pl, 0xAA1E03E0);
// mov x30, x1; restore the caller's return address.
PUT_WORD(pl, 0xAA0103FE);
PUT_BYTES(pl, LOADER_EXE_ARM64, sizeof(LOADER_EXE_ARM64));
```

The C2Implant third-party build step selects the Donut makefile based on the CMake/MSVC target architecture. That keeps the architecture decision in the build graph instead of requiring manual release steps.

This was also where CI was most useful. Loader code tends to fail in small, architecture-specific ways. A missing header conversion, a wrong machine type, or a stale makefile can break only one target while the others keep passing. The matrix made those failures visible.

![Donut CI run]({{ "/assets/images/exploration-c2-0-16/donut-ci.png" | relative_url }})

There was also a Windows 11 compatibility correction in the Donut loader path: the loader moved away from a section-mapping behavior that no longer matched stricter memory protection behavior and used a simpler allocation path instead. For this release, the point is compatibility rather than an operational claim: loader assumptions must be tested against current Windows runtime behavior.

## ARM64 Indirect Syscalls: The Shellcode Boundary

The most technical part of the ARM64 work was the syscall bridge.

The goal was direct: keep the same C++ syscall wrapper model, but make the last dispatch step work correctly on Windows ARM64. The x86/x64 implementation could not simply be copied. ARM64 has a different calling convention, different volatile registers, a link register, and a different assembly shape for the final branch.

The ARM64 bridge keeps the exported syscall wrappers stable while moving the architecture-specific dispatch into a small assembly trampoline. Conceptually, the flow is:

1. Preserve the argument registers and link register.
2. Resolve the syscall target through the hashed wrapper metadata.
3. Restore the original call state.
4. Branch to the resolved syscall address.

That gives the C++ layer a stable interface while keeping the ABI-sensitive logic in the architecture-specific assembly file.

The important engineering constraint was state preservation. On ARM64, `x0` through `x7` carry the first arguments. The link register carries the return path. If the resolver call destroys that state, the eventual syscall receives corrupted input. The trampoline therefore has to treat the resolver as a temporary detour and restore the syscall call frame before branching.

This is the kind of code where review alone is weak. The tests around syscall wrappers are what make the implementation maintainable. They exercise memory allocation, memory protection changes, thread creation, token/process handles, and file operations through the wrapper layer. The goal is not to prove every Windows behavior. The goal is to prove that the wrapper contract still holds across architectures.

## ARM64 Hardware Breakpoints

Hardware breakpoints were another place where ARM64 required a real implementation rather than a compile-time exception.

On x86/x64, the implementation is built around debug registers. On ARM64, the exception context exposes breakpoint value and control registers instead. That means the abstraction has to map the same high-level behavior onto different register models:

- x86/x64 uses `Dr0` through `Dr3` and debug status/control fields;
- ARM64 uses `Bvr` and `Bcr` entries;
- x86/x64 exception handling is based around the instruction pointer and single-step behavior;
- ARM64 uses the program counter, link register, stack pointer, and ARM64 breakpoint exception behavior.

The user-facing module should not need to care about those details. The breakpoint manager should expose the same operations: set a breakpoint, intercept the call, modify the context if needed, continue execution, and remove the breakpoint.

The ARM64 test case is especially valuable because it checks behavior rather than only compilation. It arms a breakpoint on a target function, verifies that the hook path is hit, changes the observed return value, and then confirms normal behavior returns after the breakpoint is removed.

That style of test is what made the feature releasable. It validates the architecture-specific context manipulation through observable behavior.

## Codex as the Low-Level Engineering Accelerator

Codex had a central role in this release, but not as a replacement for testing.

The useful pattern was:

1. Use Codex to reason through the architecture-specific implementation.
2. Make the change in a small, reviewable patch.
3. Let CI compile and execute the target matrix.
4. Feed the build or test failure back into the next iteration.

That loop was especially effective for the ARM64 shellcode, syscall bridge, hardware breakpoint context handling, and Donut integration work. These areas are hard because small details across assembly, C++, build configuration, and CI all have to line up at the same time.

Codex helped keep those details connected while the CI runner provided the hard truth. That distinction matters. For low-level security tooling, an assistant can propose and refactor quickly, but the release should trust the compiler, the tests, and the native runner.

In practice, Codex was most valuable when the task had strong feedback:

- compiler errors from the ARM64 MSVC toolchain;
- failing CTest output;
- missing release artifacts;
- Donut build failures;
- packaging validation errors.

The result was not "AI wrote an ARM64 port". The result was an engineering loop where Codex accelerated analysis and patching, while CI enforced correctness.

## What This Enables Next

The 0.16 release makes the project more portable and more testable.

Concretely, the project no longer treats ARM64 as an experiment hidden behind a local build. `x64`, `x86`, and `ARM64` now have named release artifacts, CI coverage, and packaging validation. The TeamServer release can rely on those assets instead of assuming they were built correctly somewhere else. Donut also becomes part of that multi-architecture contract, because its loader path is selected and validated through the same release flow.

The main lesson from 0.16 is simple: multi-arch support is not a feature flag. It is a contract between code, build system, tests, third-party tooling, release packaging, and the automation that helps maintain all of it.

## References

- [C2Implant 0.16.0 release](https://github.com/maxDcb/C2Implant/releases/tag/0.16.0)
- [C2TeamServer 0.16.1 release](https://github.com/maxDcb/C2TeamServer/releases/tag/0.16.1)
- [C2Implant CI workflow](https://github.com/maxDcb/C2Implant/blob/main/.github/workflows/CI.yml)
- [C2Implant release workflow](https://github.com/maxDcb/C2Implant/blob/main/.github/workflows/Release.yml)
- [C2TeamServer release workflow](https://github.com/maxDcb/C2TeamServer/blob/master/.github/workflows/Release.yml)
- [GitHub Actions runner reference](https://docs.github.com/en/enterprise-cloud@latest/actions/writing-workflows/choosing-where-your-workflow-runs/choosing-the-runner-for-a-job)
- [Donut fork](https://github.com/maxDcb/donut)
- [Donut ARM64 compatibility and CI commit](https://github.com/maxDcb/donut/commit/b1e74fed695e2953f70caf06c723a1472e9be1b1)
- [Donut x86/x64/ARM64 build commit](https://github.com/maxDcb/donut/commit/e52fa7622e52785e4565e514c3cd1e1a756ef0b5)
- [Donut Windows 11 loader compatibility commit](https://github.com/maxDcb/donut/commit/6c753f2f56e02db22e380ed2c35e7850f11eba6f)
