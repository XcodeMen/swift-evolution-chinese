# 0150 - Package Manager Support for branches

* Proposal: [SE-0150](0150-package-manager-branch-support.md)
* Author: [Boris Bügling](https://github.com/neonichu)
* Review Manager: [Daniel Dunbar](https://github.com/ddunbar)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170130/031428.html)
* Bug: [SR-666](https://bugs.swift.org/browse/SR-666)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal adds enhancements to the package manifest to support development of packages without strict versioning. This is one of two features, along with "Package Manager Support for Top of Tree development", being proposed to enable use of SwiftPM to develop on "top of tree" of related packages.

本提案增强包清单（package manifest）的功能以支持开发不需要严格版本控制的包（package）。这是提出的能够使用 SwiftPM 开发 "top of tree" 相关联的软件包的两个特性之一，另外一个为 "Package Manager Support for Top of Tree development"。

## Motivation 动机

The package manager currently supports packages dependencies which are strictly versioned according to semantic versioning. This is how a package's dependencies should be specified when that package is released, but this requirement hinders some development workflows:

- bootstrapping a new package which does not yet have a version at all
- developing related packages in tandem in between releases, when one package may depend on the latest revision of another, which has not yet been tagged for release

现有的包管理器支持根据语义版本进行严格版本控制的包依赖关系。这就是当要发布一个软件包时应该如何指定该包的依赖关系，但这一要求阻碍了一些开发工作流程：

- 启动开发一个新的包，但它并没有一个版本号；
- 在两个版本之间串行开发相关联的软件包，当一个软件包可能依赖于另一个包的最新版本时，但这个新版本还没有被标记为已发布。

## Proposed solution 建议的解决方案

As a solution to this problem, we propose to extend the package manifest to allow specifying a branch or revision instead of a version to support revlocked packages and initial bootstrapping. In addition, we will also allow specifying a branch or revision as an option to the `pin` subcommand.

作为这个问题的解决方案，我们建议扩展包的清单以允许指定分支或修订版本，而不是一个支持 revlocked packages 和初始化引导的版本。此外，我们还将允许指定一个分支或修订版作为 `pin` 子命令的可选项。

## Detailed Design 详细设计

### Specifying branches or revisions in the manifest
### 在清单中指定分支或修订版本

We will introduce a second initializer for `.Package` which takes a branch instead of a version range:

我们将为 `.Package` 引入一个 second initializer，它需要一个分支而不是版本范围：

```swift
import PackageDescription

let package = Package(
    name: "foo",
    dependencies: [
        .Package(url: "http://url/to/bar", branch: "development"),
    ]
)
```

In addition, there is also the option to use a concrete revision instead:

此外，还可以选择使用具体的修订版本：

```swift
import PackageDescription

let package = Package(
    name: "foo",
    dependencies: [
        .Package(url: "http://url/to/bar", revision: "0123456789012345678901234567890123456789"),
    ]
)
```

Note that the revision parameter is a string, but it will still be sanity checked by the package manager. It will only accept the full 40 character commit hash here for Git and not a commit-ish or tree-ish. 

请注意，修订版本参数是一个字符串，但它仍将由包管理器检查正确性。它将仅接受 Git 的完整 40 字符提交哈希，而不是 commit-ish 或 tree-ish。

Whenever dependencies are checked out or updated, if a dependency on a package specifies a branch instead of a version, the latest commit on that branch will be checked out for that package. If a dependency on a package specifies a branch instead of a version range, it will override any versioned dependencies present in the current package graph that other packages might specify.

每当检出或更新依赖项时，如果包的依赖关系指定了一个分支而不是一个版本，则该分支上的最新提交将被检出给该包。如果包的依赖关系指定了一个分支而不是版本范围，则它将覆盖其它包可能指定的当前包图表中存在的任何版本化依赖关系。

For example, consider this graph with the packages A, B, C and D:

例如，考虑下图中的包 A、B、C 和 D：

A -> (B:master, C:master)
     B -> D:branch1
     C -> D:branch2

The package manager will emit an error in this case, because there are dependencies on package D for both `branch1` and `branch2`.

在这种情况下，包管理器将报错，因为这里对于包 D 的 `branch1` 和 `branch2` 都有依赖。

While this feature is useful during development, a package's dependencies should be updated to point at versions instead of branches before that package is tagged for release. This is because a released package should provide a stable specification of its dependencies, and not break when a branch changes over time. To enforce this, it is an error if a package referenced by a version-based dependency specifies a branch in any of its dependencies.

虽然此特性在开发过程中非常有用，但是在一个包被标记为发布之前，包的依赖关系应该被更新到指定版本而不是分支。这是因为一个发布的包应该提供一个稳定的依赖项规范，而且当一个分支随着时间的推移而改变时，它不会中断。为了确保这一点，如果一个基于版本的依赖项所引用的包在其任何依赖关系中指定了一个分支，那就是错的。

Running `swift package update` will update packages referencing branches to their latest remote state. Running `swift package pin` will store the commit hash for the currently checked out revision in the pins file, as well as the branch name, so that other users of the package will receive the exact same revision if pinning is enabled. If a revision was specified, users will always receive that specific revision and `swift package update` becomes a no op.

执行 `swift package update` 命令将更新包的引用分支到其远程最新的状态。执行 `swift package pin` 命令将在 pins 文件中存储当前检出的修订版本的提交哈希值和分支名字，这样当开启 pinning 时，包的其他用户将接收到完全相同的修订版本。如果指定了一个修订版本，则用户将始终接收到这个指定的修订版本，而 `swift package update` 将失效。

### Pinning to a branch or revision
### Pinning 一个分支或修订版

In addition to specifying a branch or revision in the manifest, we will also allow specifying it when pinning:

除了在清单中指定分支或修订，我们还将允许在 pinning 时指定它：

```bash
$ swift pin <package-name> --branch <branch-name>
$ swift pin <package-name> --revision <revision>
```

This is meant to be used for situations where users want to temporarily change the source of a package, but it is just an alternative way to get the same semantics and error handling described in the previous section.

这将适用于当用户希望临时更改包的来源的情况，但它只是获取上一节中描述的相同语义和错误处理的另一种方法。

## Impact on existing code 对现有代码的影响

There will no impact on existing code.

不会影响现有代码。

## Alternative considered 考虑过的替代方案

We decided to make using a version-based package dependency with unversioned dependencies an error, because a released package should provide a stable specification of its dependencies. A dependency on a branch could break at any time when the branch is being changed over time.

我们之前决定在一个基于版本的包依赖项上使用非版本化的依赖关系时遇到错误，因为一个发布的包应该提供一个稳定的依赖项规范。当分支随着时间的推移而改变时，分支的依赖项可能随时中断。


