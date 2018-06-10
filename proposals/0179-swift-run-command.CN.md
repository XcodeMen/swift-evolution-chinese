# 0179 - Swift `run` Command

* Proposal: [SE-0179](0179-swift-run-command.md)
* Authors: [David Hart](http://github.com/hartbit/)
* Review Manager: [Daniel Dunbar](https://github.com/ddunbar)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170529/036909.html)
* Implementation: [apple/swift-package-manager#1187](https://github.com/apple/swift-package-manager/pull/1187)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

The proposal introduces a new `swift run` command to build and run an executable defined in the current package.

本提案引入一个新的 `swift run` 命令，用于编译和运行当前包中定义的可执行文件。

## Motivation 动机

It is common to want to build and run an executable during development. For now, one must first build it and then execute it from the build folder:

在开发过程中，通常需要编译并运行一个可执行文件。目前，我们必须先进行编译，然后再在 build 文件夹中执行它:

```bash
$ swift build
$ .build/debug/myexecutable
```

In Swift 4, the Swift Package Manager will build to a different path, containing a platform sub-folder (`.build/macosx-x86_64/debug` for mac and `.build/linux-x86_64/debug` for linux), making it more cumbersome to run the executable from the command line.

在 Swift 4 中，Swift 包管理器（SwiftPM）的编译路径与之前有所不同，将包含一个平台子文件夹（Mac 下为 `.build/macosx-x86_64/debug`，Linux 下为 `.build/linux-x86_64/debug`），这将导致要在命令行中执行编译后的可执行文件会更加繁琐。

To improve the development workflow, the proposal suggests introducing a new first-level `swift run` command that will build if necessary and then run an executable defined in the `Package.swift` manifest, reducing the above steps to just one.

为了改进开发工作流程，本提案建议引入一个新的 `swift run` 一级命令，它会先进行编译（如果需要），然后运行 `Package.swift` 清单中定义的可执行文件，减少上述步骤至只要一步。

## Proposed solution 建议的解决方案

The swift `run` command would be defined as:

Swift 的 `run` 命令将被定义如下：

```bash
$ swift run --help
OVERVIEW: Build and run executable

USAGE: swift run [options] [executable [arguments]]

OPTIONS:
  --build-path            Specify build/cache directory [default: ./.build]
  --chdir, -C             Change working directory before any other operation
  --color                 Specify color mode (auto|always|never) [default: auto]
  --configuration, -c     Build with configuration (debug|release) [default: debug]
  --enable-prefetching    Enable prefetching in resolver
  --skip-build            Skip building the executable product
  --verbose, -v           Increase verbosity of informational output
  -Xcc                    Pass flag through to all C compiler invocations
  -Xlinker                Pass flag through to all linker invocations
  -Xswiftc                Pass flag through to all Swift compiler invocations
  --help                  Display available options
```

If needed, the command will build the product before running it. As a result, it can be passed any options `swift build` accepts. As for `swift test`, it also accepts an extra `--skip-build` option to skip the build phase.

如果必要，该命令会在运行产品之前对其先进行编译。因此，它可以被传递任何 `swift build` 命令所接收的参数选项。对于 `swift test` 命令，同样可接收一个额外的 `--skip-build` 来跳过编译过程。

After the options, the command optionally takes the name of an executable product defined in the `Package.swift` manifest and introduced in [SE-0146](0146-package-manager-product-definitions.md). If called without an executable and the manifest defines one and only one executable product, it will default to running that one. In any other case, the command fails.

在 options 之后，该命令可选地接收在 [SE-0146](0146-package-manager-product-definitions.md) 引入的 `Package.swift` 清单中所定义的可执行产品的名称。如果调用该命令时没有传递一个可执行文件，那么它会默认执行清单中所唯一定义的那个可执行产品。除此之外其他情况，命令都会报错。

If the executable is explicitly defined, all remaining arguments are passed as-is to the executable.

如果显式定义了可执行文件，则所有剩余的参数都会被原样传递给可执行文件。

```bash
$ swift run # .build/debug/exe
$ swift run exe # .build/debug/exe
$ swift run exe arg1 arg2 # .build/debug/exe arg1 arg2
```

## Alternatives considered 考虑过的替代方案

One alternative to the Swift 4 change of build folder would be for the Swift Package Manager to create and update a symlink at `.build/debug` and `.build/release` that point to the latest build folder for that configuration. Although that should probably be done to retain backward-compatibility with tools that depended on the build location, it does not completely invalidate the usefulness of the `run` command.

Swift 4 更改编译文件夹的一个替代方案是，让 Swift 包管理器为指向其配置的最新编译文件夹的 `.build/debug` and `.build/release` 创建并更新一个符号链接。尽管这可能是为了保留依赖于编译位置的工具的向后兼容性，但它并不能完全否定 `run` 命令的实用性。




