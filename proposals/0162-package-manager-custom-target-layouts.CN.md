# 0162 - Package Manager Custom Target Layouts

* Proposal: [SE-0162](0162-package-manager-custom-target-layouts.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Rick Ballard](https://github.com/rballard)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170410/035471.html)
* Bug: [SR-29](https://bugs.swift.org/browse/SR-29)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal enhances the `Package.swift` manifest APIs to support custom
target layouts, and removes a convention which allowed omission of targets from
the manifest.

本提案增强了 `Package.swift` 清单的 API，以支持自定义 target 的布局，并且删除了允许从清单中省略 targets 的约定。

## Motivation 动机

The Package Manager uses a convention system to infer targets structure from
disk layout. This works well for most packages, which can easily adopt the
conventions, and frees users from needing to update their `Package.swift` file
every time they add or remove sources. Adopting the conventions is more
difficult for some packages, however – especially existing C libraries or large
projects, which would be difficult to reorganize. We intend to give users a way
to make such projects into packages without needing to conform to our
conventions.

包管理器使用一套约定系统来从磁盘布局中推断 target 的结构。这适用于大多数的包，它可以轻松地采用这些约定，并且每当添加或移除代码时，用户将不用再修改 `Package.swift` 文件。然而，对于一些包来说，采用这些约定是比较困难的，尤其是对现有的 C 语言库或大型工程，将很难重构。我们打算给用户一种方式，可以使这些项目成为包，而不必遵循我们的约定。

The current convention rules make it very convenient to add new targets and
source files by inferring them automatically from disk, but they also can be
confusing, overly-implicit, and difficult to debug; for example, if the user
does not follow the conventions correctly which determine their targets, they
may wind up with targets they don't expect, or not having targets they did
expect, and either way their clients can't easily see which targets are available
by looking at the `Package.swift` manifest. We want to retain convenience where it
really matters, such as easy addition of new source files, but require explicit
declarations where being explicit adds significant value. We also want to make
sure that the implicit conventions we keep are straightforward and easy to
remember.

当前的约定规则可以非常方便地从磁盘中推断出新的 targets 和源文件并添加它们，但这可能是容易混淆，过于隐式，并且很难调试；例如，如果用户没有正确地遵循确定其 targets 的约定，那么他们可能会结束他们不期望的 targets，或无法获得他们所期望的 targets，而且他们的客户端也无法轻易地通过查看 `Package.swift` 清单得知哪些 targets 是可用的。我们希望在真正重要的地方保留便利性，例如轻松添加新的源文件，但需要显式的声明才能显式地添加有效值。我们还希望确保我们保留的隐式约定是简单易懂的。

## Proposed solution 建议的解决方案

* We propose to stop inferring targets from disk. They must be explicitly declared
  in the manifest file. The inference was not very useful, as targets eventually
  need to be declared in order to use common features such as product and target
  dependencies, or build settings (which are planned for Swift 4). Explicit
  target declarations make a package easier to understand by clients, and allow us
  to provide good diagnostics when the layout on disk does not match the
  declarations.
  
* 我们建议停止从磁盘中推断 targets。它们必须在清单文件中显示声明。这一推断并不是很有用，因为 targets 最终需要被声明，以便使用例如 product 和 target 依赖关系等常见功能，或者编译设置（为 Swift 所计划的）。显示的 target 声明使得客户端更易于理解 package，并允许我们在磁盘上的布局与所声明不的匹配时提供良好的诊断。

* We propose to remove the requirement that name of a test target must have
  suffix "Tests". Instead, test targets will be explicitly declared as such
  in the manifest file.
  
* 我们建议移除测试 target 的名称必须具有后缀 "Tests" 的要求。相反地，测试 target 将在清单文件中显示地声明成这样。

* We propose a list of pre-defined search paths for declared targets.

* 我们为声明过的 target 提出了一个预先定义的搜索路径列表。

  When a target does not declare an explicit path, these directories will be used
  to search for the target. The name of the directory must match the name of
  the target. The search will be done in order and will be case-sensitive.
  
  当一个 target 没有声明显示路径时，以下这些目录将用于搜索 target。目录的名称必须与 target 的名称相匹配。搜索将按顺序进行，并区分大小写。

    Regular targets: package root, Sources, Source, src, srcs.
    Test targets: Tests, package root, Sources, Source, src, srcs.

  It is an error if a target is found in more than one of these paths. In
  such cases, the path should be explicitly declared using the path property
  proposed below.
  
  如果一个 target 在多个路径中被找到，则将报错。在这种情况下，搜索路径应该使用下面提出的 path 属性来显示地声明。

* We propose to add a factory method `testTarget` to the `Target` class, to define
  test targets.
  
* 我们建议向 `Target` 类添加一个工厂方法 `testTarget`，用来定义测试 targets。

    ```swift
    .testTarget(name: "FooTests", dependencies: ["Foo"])
    ```

* We propose to add three properties to the `Target` class: `path`, `sources` and
  `exclude`.
  
* 我们建议向 `Target` 类添加 3 个属性：`path`，`sources` 和 `exclude`。

    * `path`: This property defines the path to the top-level directory containing the
      target's sources, relative to the package root. It is not legal for this path
      to escape the package root, i.e., values like "../Foo", "/Foo" are invalid.  The
      default value of this property will be `nil`, which means the target will be
      searched for in the pre-defined paths. The empty string ("") or dot (".") implies
      that the target's sources are directly inside the package root.
      
    * `path`：该属性定义了包含 target 源代码的顶级目录的路径，相对于包的根目录。这个路径如果跳过包的根目录是不合法的，就像 "../Foo", "/Foo" 这样的值都是无效的。该属性的默认值为 `nil`，表示此时将在预定义的路径中搜索 target。空字符串 ("") 或句点 (".") 意味着 target 的源代码直接位于包的根目录下。

    * `sources`: This property defines the source files to be included in the
      target. The default value of this property will be nil, which means all
      valid source files found in the target's path will be included. This can
      contain directories and individual source files.  Directories will be
      searched recursively for valid source files. Paths specified are relative
      to the target path.
      
    * `sources`：该属性定义要被包含在 target 中的源文件。该属性的默认值为 `nil`，表示 target 路径中找到的所有有效源文件都将被包含在内。它可以包含目录和单独的源文件。目录将被递归搜索有效的源文件。 指定的路径是相对于 target 路径。

      Each source file will be represented by String type. In future, we will
      consider upgrading this to its own type to allow per-file build settings.
      The new type would conform to `CustomStringConvertible`, so existing
      declarations would continue to work (except where the strings were
      constructed programatically).
      
      每个源文件将通过字符串类型表示。将来，我们会考虑将其升级到自己的类型以允许单个文件的编译设置。新类型将符合 `CustomStringConvertible`，所以现有的声明将继续工作（除非字符串是以编程方式构建的）。

    * `exclude`: This property can be used to exclude certain files and
      directories from being picked up as sources. Exclude paths are relative
      to the target path. This property has more precedence than `sources`
      property.
      
    * `exclude`：该属性可用于排除将某些文件和目录作为源文件获取。排除路径是相对于 target 路径。该属性的优先级高于 `sources` 属性。

      _Note: We plan to support globbing in future, but to keep this proposal short
      we are not proposing it right now._
      
      _注：我们计划在将来支持通配符，但为了保持本提案简短，我们现在先不提出来。_

* It is an error if the paths of two targets overlap (unless resolved with `exclude`).

* 如果两个 target 的路径重叠（除非使用 `exclude` 进行解析），则会出现错误。

    ```swift
    // This is an error:
    .target(name: "Bar", path: "Sources/Bar"),
    .testTarget(name: "BarTests", dependencies: ["Bar"], path: "Sources/Bar/Tests"),

    // This works:
    .target(name: "Bar", path: "Sources/Bar", exclude: ["Tests"]),
    .testTarget(name: "BarTests", dependencies: ["Bar"], path: "Sources/Bar/Tests"),
    ```

* For C family library targets, we propose to add a `publicHeadersPath`
  property.
  
* 对于 C 系列库的 targets，我们建议添加一个 `publicHeadersPath` 属性。

    This property defines the path to the directory containing public headers of
    a C target. This path is relative to the target path and default value of
    this property is `include`. This mechanism should be further improved
    in the future, but there are several behaviors, such as modulemap generation,
    which currently depend of having only one public headers directory. We will address
    those issues separately in a future proposal.
    
    该属性定义包含 C target 的公共头文件目录的路径。此路径与 target 的路径有关，它的默认值为 `include`。这一机制今后有待进一步提高，但有一些行为，比如模块映射的生成，目前依赖于只有一个公共头文件目录。我们将在将来的提案中分别讨论这些问题。

    _All existing rules related to custom and automatic modulemap remain intact._
    
    _所有与自定义和自动模块映射相关的现有规则保持不变。_

* Remove exclude from `Package` class.

* 从 `Package` 类中移除 exclude。

    This property is no longer required because of the above proposed
    per-target exclude property.
    
    由于上述提出了每个 target 自己的 exclude 属性，这里的 exclude 属性不再需要。

* The templates provided by the `swift package init` subcommand will be updated
  according to the above rules, so that users do not need to manually
  add their first target to the manifest.
  
* `swift package init` 子命令提供的模板将根据上述规则进行更新，因此用户不需要手动将其第一个 target 添加到清单中。

## Examples 范例:

* Dummy manifest containing all Swift code.
* 包含所有 Swift 代码的虚拟清单。

```swift
let package = Package(
    name: "SwiftyJSON",
    targets: [
        .target(
            name: "Utility",
            path: "Sources/BasicCode"
        ),

        .target(
            name: "SwiftyJSON",
            dependencies: ["Utility"],
            path: "SJ",
            sources: ["SwiftyJSON.swift"]
        ),

        .testTarget(
            name: "AllTests",
            dependencies: ["Utility", "SwiftyJSON"],
            path: "Tests",
            exclude: ["Fixtures"]
        ),
    ]
)
```

* LibYAML

```swift
let packages = Package(
    name: "LibYAML",
    targets: [
        .target(
            name: "libyaml",
            sources: ["src"]
        )
    ]
)
```

* Node.js http-parser

```swift
let packages = Package(
    name: "http-parser",
    targets: [
        .target(
            name: "http-parser",
            publicHeaders: ".",
            sources: ["http_parser.c"]
        )
    ]
)
```

* swift-build-tool

```swift
let packages = Package(
    name: "llbuild",
    targets: [
        .target(
            name: "swift-build-tool",
            path: ".",
            sources: [
                "lib/Basic",
                "lib/llvm/Support",
                "lib/Core",
                "lib/BuildSystem",
                "products/swift-build-tool/swift-build-tool.cpp",
            ]
        )
    ]
)
```

## Impact on existing code 对现有代码的影响

These enhancements will be added to the version 4 manifest API, which will
release with Swift 4. There will be no impact on packages using the version 3
manifest API.  When packages update their minimum tools version to 4.0, they
will need to update the manifest according to the changes in this proposal.

这些增强功能将被添加到版本 4 manifest API 中，它将随 Swift 4 一起发布。对于使用版本 3 manifest API 的包将不会有任何影响。当包将其最低工具版本更新为 4.0 时，他们将需要根据此提案中的更改来更新清单。

There are two flat layouts supported in Swift 3:

1. Source files directly in the package root.
2. Source files directly inside a `Sources/` directory.

在 Swift 3 中支持两种 flat layouts ：

1. 源文件直接放在包的根目录中。
2. 源文件直接放在一个 `Sources/` 目录里。

If packages want to continue using either of these flat layouts, they will need
to explicitly set a target path to the flat directory; otherwise, a directory
named after the target is expected. For example, if a package `Foo` has
following layout:

如果包希望继续使用这些 flat layouts，则需要显式将一个 target 路径设置为 flat 目录，否则，一个目录预期将在 target 之后命名。例如，如果一个包 `Foo` 有以下 layout ：

```
Package.swift
Sources/main.swift
Sources/foo.swift
```

The updated manifest will look like this:

则更新后的清单将如下所示：

```swift
// swift-tools-version:4.0
import PackageDescription

let package = Package(
    name: "Foo",
    targets: [
        .target(name: "Foo", path: "Sources"),
    ]
)
```

## Alternatives considered 考虑过的解决方案

We considered making a more minimal change which disabled the flat layouts
by default, and provided a top-level property to allow opting back in to them.
This would allow us to discourage these layouts – which we would like
to do before the package ecosystem grows – without needing to add a fully
customizable API. However, we think the fuller API we've proposed here is fairly
straightforward and provides the ability to make a number of existing projects
into packages, so we think this is worth doing at this time.

我们考虑过一个更小的改动，默认情况下禁用 flat layouts，并提供一个高级属性以允许选择重新开启它们。这将允许我们阻止这样一些 layouts —— 那些我们期望在 package 生态系统增长之前做的 —— 以及不需要添加完全可定制的 API。不过，我们认为我们在这里提出的更为丰富的 API 是非常简单的，并且提供了将许多现有的项目打成包的能力，因此，我们认为这是目前值得做的。

