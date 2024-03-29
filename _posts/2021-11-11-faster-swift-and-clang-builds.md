---
layout: post
title:  "Faster Swift and Clang Builds"
date:   2021-11-11 19:27:59 -0800
categories: builds
---

This is a collection of suggestions that reduce the time it takes to build the Swift and Clang compilers. Out of the box, the Swift build is designed to build everything and anything, because it doesn't know what you need and what you don't. This makes for longer build times. I think few people need the kitchen sink build, and by building less, you reduce build times. These suggestions are in no particular order. You can take my word for it, or you can try these out and measure the savings for yourself, but I have intentionally avoided numbers. The time savings of these optimizations will be most noticeable in full builds, but they can also help incremental builds too.

### Delete Sibling Repos (Swift)

The initial setup instructions for `swift-project` involves cloning 30+ related Swift repos. Most of these are optional, and you may never build. Review these repos and delete any that you don't need for your purposes. For example, to build the Swift compiler (the frontend), you need `cmark`, `llvm-project`, and of course `swift`. These three repos are typically all I ever use.

Some of the repos are optional, but still require you to explicitly opt out from building. At the time of writing, the new `swift-driver` is optional but you need to call `build-script` with `--skip-early-swift-driver` to opt-out from building it. On the flip side, if you want to build `swift-driver`, then you need to keep the `swift-driver`, `llbuild`, `yams`, `swift-argument-parser`, and `swift-tools-support-core` repos.

### Build for Release (Swift & Clang)

Release builds are faster in two ways. First, `Release` compile times are faster than `RelWithDebInfo`. This is because generating debug info for all of LLVM/Clang/Swift has its overhead. Second, the test suite runs faster with `Release` builds than with `Debug` builds. As an extra, if you're running low on disk space, skipping debug info results in a smaller build directory.

However this isn't "free", this is a tradeoff optimizing for faster build and test times, by pessimizing the debug experience. Optimized code is harder to debug, but it can at least be doable if you have debug info (`RelWithDebInfo`). Without debug info (`Release`), you'll be stepping through assembly.

One solution to this is to selectively compile some files for debugging. At any one time, most people are debugging a small number of files, sometimes just one or two files, and generally at most a whole module or two. Individual files can be compiled for debugging by doing the following:

1. Run `touch` on the files to update their timestamp
2. Re-run `ninja` using `CCC_OVERRIDE_OPTIONS="# O0 +-g"`

If you haven't seen `CCC_OVERRIDE_OPTIONS` before, it's a Clang environment variable that allows you to override compile flags using its mini-commands. See the [`driver.cpp` source](https://github.com/llvm/llvm-project/blob/93a1fc2e18b452216be70f534da42f7702adbe1d/clang/tools/driver/driver.cpp#L79-L105) for its documentation:

This workflow uses three commands, `#`, `O` and `+`. These three, and a few other useful commands do the following:

1. `#` turns on quiet mode (verbose is default)
2. `O` sets the optimization level to a given level (removing all other `-O` flags)
3. `+` appends the given flag at the end of the command line
4. `x` deletes the given flag
5. `X` deletes the given flag _and_ the following argument (for space separated values)
6. `s/XXX/YYY/` substitutes regex "XXX" for string "YYY" on the command line

Using `O0` ensures the file is compiled with `-O0`, and using `+-g` adds the `-g` flag to generate debug info.

*Update*: [Jordan Rose points out](https://twitter.com/UINT_MIN/status/1460853194796331008) that "LLVM doesn't promise that optimized and non-optimized builds are compatible. In practice it does usually work though". In my experiences, I have not had any issues mixing opt and non-opt while working on LLDB, and I use this technique regularly.

Admittedly, this workflow is not ideal, the extra touch & build steps are manual and add a bit of time, but overall I find the workflow to be a net positive.

I use this workflow for my edit-compile-test cycle. For every file I edit, I want it compiled at `-O0`, to achieve this I use `CCC_OVERRIDE_OPTIONS` for all incremental builds. To make this easy, I use a zsh global macro:

```sh
alias -g DBG='CCC_OVERRIDE_OPTIONS="# O0 +-g"'
DBG ninja clang
```

There are other ways you could control which files are built for debug, without `CCC_OVERRIDE_OPTIONS`. One way is to create a "compiler launcher" script that adds or removes flags before clang gets called (see [`CMAKE_CXX_COMPILER_LAUNCHER`](https://cmake.org/cmake/help/latest/variable/CMAKE_LANG_COMPILER_LAUNCHER.html#variable:CMAKE_%3CLANG%3E_COMPILER_LAUNCHER)), but I haven't tried this.

### Use a `.noindex` Build Directory (Swift & Clang, macOS only)

*Update*: Richard Howell asked whether this is still valid. Admittedly, it's been years since I've measured the effect of `.noindex`. So, I did a pair of builds, with and without `.noindex`, and the time savings was negligible. This suggestion can be ignored.

On macOS, directories with a `.noindex` suffix are not indexed by Spotlight. Builds produce a lot of build artifacts, and you probably don't want any CPU contention created by Spotlight indexing during the build.

For `llvm-project`, name your build directory `build.noindex`. For `swift-project`, you can either use `SWIFT_BUILD_ROOT`, or `--build-subdir`. I use `--build-subdir Release.noindex`, which creates a swift build directory named `build/Release.noindex/swift-macosx-arm64`.

### Make a Non-Cross Compiler (Swift & Clang)

The common case when building a compiler is to use it only to produce binaries that run on the host machine. If this is true for you, you can save time by not compiling support for all the other architectures LLVM and Swift support.

* Clang: `LLVM_TARGETS_TO_BUILD=host` (or Native)
* Swift: `--llvm-targets-to-build host` and `--swift-darwin-supported-archs "$(uname -m)"`

### Enable Clang Modules (Swift & Clang)

[Clang modules](https://clang.llvm.org/docs/Modules.html) also serve the role of precompiled headers, which are designed to speed up compilation. For LLVM, I don't know why modules are not enabled by default.

* Clang: `LLVM_ENABLE_MODULES=ON`
* Swift: `--llvm-enable-modules`

### Skip Platforms (Swift)

For compiler development purposes, you may not need to support all platforms. You can skip platforms when invoking `build-script`, for example as a default I use `--skip-{watchos,tvos}`.

### Skip Building of Optional Subprojects (Swift & Clang)

The LLVM project is a monorepo, and you can configure which project to build, or not build, when running `cmake`. To get faster builds, don't enable any projects you don't need. See the [LLVM CMake docs](https://llvm.org/docs/CMake.html) for info about `LLVM_ENABLE_PROJECTS` and `LLVM_ENABLE_RUNTIMES`.

For Swift, I disable building `clang-tools-extra` and `benchmarks`, using `--skip-{clang-tools-extra,benchmarks}`.

### Optimize Link Times (Swift & Clang)

In recent years, new linkers have been developed giving us more to choose from. It's worth trying out new linkers from time to time and use the fastest one that works for you. For Clang, customize `LLVM_USE_LINKER`.

Whether you're using the fastest linker or not, there may be ways to speed up linking via linker flags. For macOS, here are some linker flags that I know of that can change the speed of linking:

* `-random_uuid`: don't spend time hashing the binary's contents to derive a UUID, instead use a random UUID which is more of a O(1) operation
* `-no_deduplicate`: don't spend time finding duplicate symbols & uniquing them
* `-dead_strip`: do spend time finding unused symbols & stripping them

These and some other linking tips come from the [zld README](https://github.com/michaeleisel/zld/blob/741a5bf2b73b15d26221443a5fd789494c042264/README.md#other-things-to-speed-up-linking).

To try these out, the first two can be added via `CMAKE_EXE_LINKER_FLAGS`. However the last one, `-dead_strip`, needs to be prevented from being added. There are cmake flags to disable it:

* Clang: `LLVM_NO_DEAD_STRIP`
* Swift: `SWIFT_DISABLE_DEAD_STRIPPING`

### Use a Build Cache (Swift & Clang)

The Swift docs also mention this, consider using a local build cache such as `sccache`. In my use, this really helps when switching branches. Consider this scenario:

1. Build on branch A
2. Switch to branch B, and build
3. Switch back to branch A, and build again

With a build cache, the third step could be up to 100% cache hits (depending on how much cache space is configured).

To use `sccache`, use the following flags:

* Clang: `-DCMAKE_CXX_COMPILER_LAUNCHER=$(which sccache)` (and `CMAKE_C_COMPILER_LAUNCHER` too)
* Swift: `--sccache`

Note that `sccache` isn't aware of `CCC_OVERRIDE_OPTIONS`, and using it could lead to mistaken cache hits. To get around this, I have a small wrapper script which checks for `CCC_OVERRIDE_OPTIONS`. If the env variable exists, then `sccache` is not used. If the env variable does not exist, then `sccache` is called.

The more branches you work on, the more you stand to gain from a build cache. But, if you do all your work on the main branch, like [stacked commits](https://kastiglione.github.io/git/2020/09/11/git-stacked-commits.html), then a build cache might not gain you much.

## Summary

For Swift's `build-script`, these are the flags I use to reduce build time:

```sh
build-script \
  --release \
  --build-subdir Release.noindex \
  --skip-build-{clang-tools-extra,benchmarks} \
  --skip-{ios,tvos,watchos} \
  --llvm-enable-modules \
  --llvm-targets-to-build host \
  --swift-darwin-supported-archs "$(uname -m)" \
  --extra-cmake-options='-DSWIFT_DISABLE_DEAD_STRIPPING=ON' \
  --extra-cmake-options='-DLLVM_NO_DEAD_STRIP=ON' \
  --extra-cmake-options='-DCMAKE_EXE_LINKER_FLAGS="-Wl,-random_uuid -Wl,-no_deduplicate"'
```

For LLVM and Clang, these are the cmake flags I use to save build time:

```sh
cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_TARGETS_TO_BUILD=host \
  -DLLVM_ENABLE_MODULES=ON \
  -DLLVM_NO_DEAD_STRIP=ON \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,-random_uuid -WL,-no_deduplicate" \
  -B build.noindex
```
