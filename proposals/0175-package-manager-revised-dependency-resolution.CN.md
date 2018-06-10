# 0175 - Package Manager Revised Dependency Resolution

* Proposal: [SE-0175](0175-package-manager-revised-dependency-resolution.md)
* Author: [Rick Ballard](https://github.com/rballard)
* Review Manager: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170508/036577.html)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal makes the package manager's dependency resolution behavior clearer and more intuitive. It removes the pinning commands (`swift package pin` & `swift package unpin`), replaces the `swift package fetch` command with a new `swift package resolve` command with improved behavior, and replaces the optional `Package.pins` file with a `Package.resolved` file which is always created during dependency resolution.

本提案使得包管理器的依赖解析行为更清晰，更直观。它删除了 pinning 命令（`swift package pin` 和 `swift package unpin`），用一个具有改进行为的新的 `swift package resolve` 命令替换了 `swift package fetch` 命令，并在依赖解析期间总是创建的 `Package.resolved` 文件替换可选的 `Package.pins` 文件。

## Motivation 动机

When [SE-0145 Package Manager Version Pinning](https://github.com/apple/swift-evolution/blob/master/proposals/0145-package-manager-version-pinning.md) was proposed, it was observed that the proposal was overly complex. In particular, it introduced a configuration option allowing some packages to have autopinning on (the default), while others turned it off; this option affected the behavior of other commands (like `swift package update`, which has a `--repin` flag that does nothing for packages that use autopinning). This configuration option has proved to be unnecessarily confusing.

当提出 [SE-0145 Package Manager Version Pinning](https://github.com/apple/swift-evolution/blob/master/proposals/0145-package-manager-version-pinning.md) 时，就发现该提案过于复杂。它引入了一个配置选项，允许一些包默认开启 autopinning，而其它包是关闭的；这个选项影响了其他命令的行为（比如 `swift package update` 命令有一个 `--repin` 标志，它对使用 autopinning 的包没有任何作用）。这个配置选项被证明是不必要的且令人混淆的。

In the existing design, when autopinning is on (which is true by default) the `swift package pin` command can't be used to pin packages at specific revisions while allowing other packages to be updated. In particular, if you edit your package's version requirements in the `Package.swift` manifest, there is no way to resolve your package graph to conform to those new requirements without automatically repinning all packages to the latest allowable versions. Thus, specific, intentional pins can not be preserved without turning off autopinning.

在现有的设计中，当 autopinning 开启时（默认情况下是这样），`swift package pin` 命令不能被用于在特定修订版本的情况下锁定包，同时允许更新其他包。特别是，如果你在 `Package.swift` 清单中编辑你的包的版本要求时，在没有自动 repinning 所有包到最新的允许版本的情况下，就没有办法解析你的 package graph 以符合这些新的要求。因此，在未关闭 autopinning 时，特定的，有意的 pins 无法被保存。

The problems here stem from trying to use one mechanism (pinning) to solve two different use cases: wanting to record and share resolved dependency versions, vs wanting to keep a badly-behaved package at a specific version. We think the package manager could be simplified by splitting these two use cases out into different mechanisms ("resolved versions" vs "pinning"), instead of using an "autopinning" option which makes these two features mutually-exclusive and confusing.

这里的问题源于尝试使用一种机制（pinning）来解决两种不同的用例：想要记录和共享已解析的依赖版本，和想要在特定版本中保留一个性能不佳的包。我们认为包管理器可以通过将这两个用例分解成不同的机制来简化（"resolved versions" vs "pinning"），而不是使用使这两个特性互斥且令人混淆的 "autopinning" 选项。

Additionally, some dependency resolution behaviors were not well-specified and do not behave well. The package manager is lax about detecting changes to the versions specified in the `Package.swift` manifest or `Package.pins` pinfile, and fails to automatically update packages when needed, or to issue errors if the version requirements are unsatisfiable, until the user explicitly runs `swift package update`, or until a new user without an existing checkout attempts to build. We'd like to clarify and revise the rules around when and how the package manager performs dependency resolution.

此外，一些依赖解析行为之前没有得到很好的规定，并且表现不好。包管理器对于检测 `Package.swift` 清单中的指定版本或 `Package.pins` pinfile 的变化不够严格，并且无法在需要时自动更新包，或者在版本不可满足要求的情况下发出错误，直到用户显式地运行 `swift package update`，或一个不存在现有检出（checkout）的新用户尝试编译时。我们想阐明并修订包管理器何时以及如何执行依赖解析的规则。

## Proposed solution 建议的解决方案

The pinning feature will be removed. This removes the `swift package pin` and `swift package unpin` commands, the `--repin` flag to `swift package update`, and use of the `Package.pins` file.

pinning 功能将被移除。这将移除 `swift package pin` 和 `swift package unpin` 命令，以及 `swift package update` 命令的 `--repin` 标志和 `Package.pins` 文件的使用。

In a future version of the package manager we may re-introduce pinning. If we do, pins will only be recorded in the `Package.pins` file when explicitly set with `swift package pin`, and any pinned dependencies will _not_ be updated by the `swift package update` command; instead, they would need to be unpinned to be updated. This would be a purely additive feature which packages could use in addition to the resolved versions feature when desired.

在包管理器未来的某一版本中，我们或许会重新引入 pinnning。如果我们这么做，只有用 `swift package pin` 命令显式设置 pins 时，它才会被记录在 `Package.pins` 文件中，并且任何已经 pined 的依赖关系都 _不会_ 被 `swift package update` 更新。相反地，它们需要先 unpinned 才能被更新。这将是一个纯粹的附加功能，如果需要包可以将它用于做除了解析版本功能之外的事。

A new "resolved versions" feature will be added, which behaves very similarly to how pinning previously behaved when autopinning was on. The version of every resolved dependency will be recorded in a `Package.resolved` file in the top-level package, and when this file is present in the top-level package it will be used when performing dependency resolution, rather than the package manager finding the latest eligible version of each package. `swift package update` will update all dependencies to the latest eligible versions and update the `Package.resolved` file accordingly.

一个新的 "resolved versions" 功能将会被添加，该功能的行为非常类似之前在 autopinning 开启时 pinning 的表现。每个已解析的依赖项的版本将被记录在顶层包中的 `Package.resolved` 文件中，并且当该文件出现在顶层包中时，它将在执行依赖关系解析时使用，而不是让包管理器自己寻找每个包最新的合适版本。`swift package update` 命令将更新所有依赖项到最新的合适版本，并相应地更新 `Package.resolved` 文件。

Resolved versions will always be recorded by the package manager. Some users may chose to add the `Package.resolved` file to their package's `.gitignore` file. When this file is checked in, it allows a team to coordinate on what versions of the dependencies they should use. If this file is gitignored, each user will seperately choose when to get new versions based on when they run the `swift package update` command, and new users will start with the latest eligible version of each dependency. Either way, for a package which is a dependency of other packages (e.g. a library package), that package's `Package.resolved` file will not have any effect on its client packages.

解析的版本将始终由包管理器记录。有些用户可能会选择将 `Package.resolved` 文件添加到其包的 `.gitignore` 文件中。当这个文件被检入（checked in）时，它允许团队成员协调他们应该使用哪个版本的依赖关系。当这个文件被忽略时，每个用户将根据他们何时运行 `swift package update` 命令来单独选择何时获得新版本，而新用户从每个依赖项的最新的合适版本开始选择。无论哪种方式，对于一个被其它包（例如 library package）依赖的包，那么该包的 `Package.resolved` 文件将不会对客户端的包产生任何影响。

The existing `swift package fetch` command will be deprecated, removed from the help message, and removed completely in a future release of the Package Manager. In its place, a new `swift package resolve` command will be added. The behavior of `resolve` will be to resolve dependencies, taking into account the current version restrictions in the `Package.swift` manifest and `Package.resolved` resolved versions file, and issuing an error if the graph cannot be resolved. For packages which have previously resolved versions recorded in the `Package.resolved` file, the `resolve` command will resolve to those versions as long as they are still eligible. If the resolved versions file changes (e.g. because a teammate pushed a new version of the file) the next `resolve` command will update packages to match that file. After a successful `resolve` command, the checked out versions of all dependencies and the versions recorded in the resolved versions file will match. In most cases the `resolve` command will perform no changes unless the `Package.swift` manifest or `Package.resolved` file have changed. 

现有的 `swift package fetch` 命令将被启用，已经从帮助消息中删除，并在包管理器未来的发布版本中完全删除。取而代之的是，将会添加一个新的 `swift package resolve` 命令。`resolve` 的行为将是解析依赖关系，同时考虑了当前 `Package.swift` 清单和 `Package.resolved` 解析版本文件中的版本限制问题，并在当 graph 无法解析时报错。对于之前已经解析过版本并记录在 `Package.resolved` 文件中的包，只要它们仍然符合条件，`resolve` 命令就会解析为这些版本。如果已经解析过的版本文件发生了变化（例如，因为团队成员推送了一个新版本的文件），在下一次执行 `resolve` 命令时，将更新包以匹配该文件。成功执行 `resolve` 命令后，所有依赖关系的检出版本和解析版本文件（resolved versions file）中记录的版本将会匹配。在大多数情况下，除非 `Package.swift` 清单或 `Package.resolved` 文件发生变化，否则 `resolve` 将不会执行任何更改。

The following commands will implicitly invoke the `swift package resolve` functionality before running, and will cancel with an error if dependencies cannot be resolved:

以下命令将在运行之前隐式调用 `swift package resolve` 功能，如果依赖关系无法被解析时，这些命令将被取消并报错：

* `swift build`
* `swift test`
* `swift package generate-xcodeproj`

The `swift package show-dependencies` command will also implicitly invoke `swift package resolve`, but it will show whatever information about the dependency graph is available even if the resolve fails.

`swift package show-dependencies` 命令同样也会隐式调用 `swift package resolve`，但即使解析失败，它也会显示任何有关依赖关系图的信息。

The `swift package edit` command will implicitly invoke `swift package resolve`, but if the resolve fails yet did identify and fetch a package with the package name the command supplied, the command will allow that package to be edited anyway. This is useful if you wish to use the `edit` command to edit version requirements and fix an unresolvable dependency graph. `swift package unedit` will unedit the package and _then_ perform a `resolve`.

`swift package edit` 命令将隐式调用 `swift package resolve`，但如果已经确定解析失败了，并使用命令提供的包名称获取了一个包，那么命令仍然会允许该包被编辑。当您希望使用 `edit` 命令来编辑版本需求并修复无法解析的依赖关系图时，这将非常有用。`swift package unedit` 将会先撤销包的编辑，然后执行 `resolve` 命令。

## Detailed design 详细设计

The `resolve` command is allowed to automatically add new dependencies to the resolved versions file, and to remove dependencies which are no longer in the dependency graph. It can also automatically update the recorded versions of any package whose previously-resolved version is no longer allowed by the version requirements from the `Package.swift` manifests. When changed version requirements force a dependency to be automatically re-resolved, the latest eligible version will be chosen; any other dependencies affected by that change will prefer to remain at their previously-resolved versions as long as those versions are eligible, and will otherwise update likewise.

允许 `resolve` 命令自动将新的依赖关系添加到已解析的版本文件中，并删除不再在依赖关系图中的依赖关系。它还可以自动更新任何包的记录版本，而这些包的先前解析的版本不再被 `Package.swift` 清单的版本要求所允许。当更改后的版本要求强制依赖项自动重新解析时，将选择最新的符合条件的版本；任何受该更改影响的其他依赖关系，只要这些版本符合条件，就会保留其先前解析的版本，否则将会同样进行更新。

The `Package.resolved` resolved versions file will record the git revision used for each resolved dependency in addition to its version. In future versions of the package manager we may use this information to detect when a previously-resolved version of a package resolves to a new revision, and warn the user if this happens.

`Package.resolved` 解析版本文件将记录每个已解析的依赖项的版本及其所使用的 git 修订版本。在包管理器的未来版本中，我们可以使用这些信息来检测先前解析的包版本何时解析了新的修订版本，并在发生这种情况时提醒用户。

The `swift package resolve` command will not actually perform a `git fetch` on any dependencies unless it needs to in order to correctly resolve dependencies. As such, if all dependencies are already resolved correctly and allowed by the version constraints in the `Package.swift` manifest and `Package.resolved` resolved versions file, the `resolve` command will not need to do anything (e.g. a normal `swift build` won't hit the network or make unnecessary changes during its implicit `resolve`).

`swift package resolve` 命令实际上不会在任何依赖项上执行 `git fetch`，除非它需要正确地解析依赖关系。因此，如果所有的依赖关系都已经被正确解析并且被 `Package.swift` 清单和 `Package.resolved` 解析版本文件中的版本约束所允许，那么 `resolve` 命令将不需要做任何事情（例如，一个普通的 `swift build` 不会在其隐含的 `resolve` 命令执行过程中触发网络或进行不必要的更改）。

If a dependency is in edit mode, it is allowed to have a different version checked out than that recorded in the resolved versions file. The version recorded for an edited package will not change automatically. If a `swift package update` operation is performed while any packages are in edit mode, the versions of those edited packages will be removed from the resolved versions file, so that when those packages leave edit mode the next resolution will record a new version for them. Any packages in the dependency tree underneath an edited package will also have their resolved version removed by `swift package update`, as otherwise the resolved versions file might record versions that wouldn't have been chosen without whatever edited package modifications have been made.

如果一个依赖项处于编辑模式，则它将被允许检出与解析版本文件中记录的版本不同的版本。为已编辑的包所记录的版本将不会自动更改。如果在包处于编辑模式时执行 `swift package update` 操作，那些已编辑包的版本将从解析版本文件中删除，所以当这些包离开编辑模式时，下一个 resolution 将为它们记录一个新版本。已编辑过的包下的依赖关系树中的任何包也将通过 `swift package update` 移除它们已解析的版本，否则解析版本文件可能会记录已编辑的包在没有进行修改的情况时不会被选中的版本。

## Alternatives considered 考虑过的替代方案

We considered repurposing the existing `fetch` command for this new behavior, instead of renaming the command to `resolve`. However, the name `fetch` is defined by `git` to mean getting the latest content for a repository over the network. Since this package manager command does not always actually fetch new content from the network, it is confusing to use the name `fetch`. In the future, we may offer additional control over when dependency resolution is allowed to perform network access, and we will likely use the word `fetch` in flag names that control that behavior.

我们考虑过重新再利用现有的 `fetch` 命令来实现这个新行为，而不是将命令重新命名为 `resolve`。但是 `fetch` 这个名字是由 `git` 定义的，意思是通过网络获取一个版本库的最新内容。鉴于这个包管理器命令并不总是从网络中获取新的内容，所以使用 `fetch` 这个名字会令人困惑。将来，我们可以提供额外的控制权，以允许依赖项解析执行网络访问，而且我们可能会在控制该行为的标志名称中使用 `fetch` 一词。

We considered continuing to write out the `Package.pins` file for packages whose [Swift tools version](https://github.com/apple/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md) was less than 4.0, for maximal compatibility with the Swift 3.1 tools. However, as the old pinning behavior was a workflow feature and not a fundamental piece of package compatibility, we do not consider it necessary to support in the 4.0 tools.

我们考虑过继续为那些 [Swift 工具版本](https://github.com/apple/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md) 低于 4.0 的包写出 `Package.pins` 文件，以最大程度地兼容 Swift 3.1 工具。然而，旧的 pinning 行为是工作流特性，而不是基本的包兼容性，我们不认为在 4.0 工具中支持它是必要的。

We considered keeping the `pin` and `unpin` commands, with the new behavior as discussed briefly in this proposal. While we think we may wish to bring this feature back in the future, we do not consider it critical for this release; the workflow it supports (updating all packages except a handful which have been pinned) is not something most users will need, and there are workarounds (e.g. specify an explicit dependency in the `Package.swift` manifest).

我们考虑过保留 `pin` 和 `unpin` 命令，让它们支持本提案中所简要讨论过的新行为。虽然我们认为我们可能希望在未来重新使用这个功能，但我们不认为它对于这次的发布版本很重要。它支持的工作流程（更新所有的软件包，除了一小部分已被固定的）不是大多数用户所需要的，并且已经有其他解决方案了（例如在 `Package.swift` 清单中指定一个显式的依赖项）。

We considered using an `install` verb instead of `resolve`, as many other package managers use `install` for a very similar purpose. However, almost all of those package managers are for non-compiled languages, where downloading the source to a dependency is functionally equivalent to "installing" it as a product ready for use. In contrast, Swift is a compiled language, and our dependencies must be built (e.g. into libraries) before they can be installed. As such, `install` would be a misnomer for this workflow. In the future we may wish to add an `install` verb which actually does install built products, similar to `make install`.

我们考虑过使用 `install` 动词而不是 `resolve`，因为许多其他软件包管理器使用 `install` 来实现类似的目的。但是，几乎所有的这类包管理器都是针对非编译语言的，将源代码下载到依赖项上在功能上等同于将其“安装”为准备使用的产品。相反地，Swift 是一种编译语言，我们的依赖项必须在安装之前先编译好（例如编译成一个库）。因此，`install` 对于这个工作流程来说是不恰当的。在将来，我们可能希望添加一个 `install`，它实际上会安装编译过的产品，类似于 `make install`。

### Why we didn't use "Package.lock"
### 为什么我们没有使用 "Package.lock"

We considered using the `.lock` file extension for the new resolved versions file, to be consistent with many other package managers. We expect that the decision not to use this extension will be controversial, as following established precedent is valuable. However, we think that a "lockfile" is a very poor name for this concept, and that using that name would cause confusion when we re-introduce pins. Specifically:

我们考虑过为新的解析版本文件（resolved versions file）采用 `.lock` 文件扩展名，以便与许多其他包管理器保持一致。我们预计不使用这种扩展的决定将是有争议的，因为遵循既定的先例往往是有价值的。但是，我们认为 "lockfile" 对于这个概念来说是一个非常糟糕的名字，并且当我们重新引入 pins 时，使用这个名字就会造成困惑。尤其是：

- Calling this a "lock" implies a stronger lockdown of dependencies than is supported by the actual behavior. As a simple `update` command will reset the locks, and a change to the specified versions in `Package.swift` will override them, they're not really "locked" at all. This is misleading.
  称呼这个为 "lock"，暗示着一个比实际行为所支持的更强的依赖关系锁定。由于一个简单的 `update` 命令将重置锁，并且对 `Package.swift` 中的指定版本的改变将覆盖它们，所以它们并不是真正被“锁定”。这是有误导性的。

- When we re-introduce pinning, it would be very confusing to have both "locks" and "pins". Having "resolved versions" and "pins" is not so confusing.
  当我们重新引入 pinning 时，同时拥有 "locks" 和 "pins" 将会令人非常困惑。而使用 "resolved versions" 和 "pins" 则不会这么混乱。

- The term "lock" is already overloaded between POSIX file locks and locks in concurrent programming.
  术语 "lock" 早已被大量使用于 POSIX 文件锁和并发编程中的锁。

For comparison, here is a list of other package managers which implement similar behavior and their name for this file:

作为比较，下面是一组其它包管理器列表，它们都为这个文件实现了相似的行为和名称：

| Package Manager | Language | Resolved versions file name |
| --- | --- | --- |
| Yarn | JS | yarn.lock |
| Composer | PHP | composer.lock |
| Cargo | Rust | Cargo.lock |
| Bundler | Ruby | Gemfile.lock |
| CocoaPods | ObjC/Swift | Podfile.lock |
| Glide | Go | glide.lock |
| Pub | Dart | pubspec.lock |
| Mix | Elixir | mix.lock |
| rebar3 | Erlang | rebar.lock |
| Carton | Perl | carton.lock |
| Carthage | ObjC/Swift | Cartfile.resolved |
| Pip | Python | requirements.txt |
| NPM | JS | npm-shrinkwrap.json |
| Meteor | JS | versions |

Some arguments for using ".lock" instead of ".resolved" are:

使用了 ".lock" 而不是 ".resolved" 的一些理由是：

- Users of other package managers will already be familiar with the terminology and behavior.
  其它包管理器的用户已经熟悉术语和行为。

- For packages which support multiple package managers, it will be possible to put "\*.lock" into the gitignore file instead of needing a seperate entry for "\*.resolved".
  对于支持多个包管理器的软件包，可以将 "\*.lock" 放入 gitignore 文件中，而不需要为 "\*.resolved" 单独输入。

However, we do not feel that these arguments outweigh the problems with the term "lock". If providing feedback asking that we reconsider this decision, please be clear about why the above decision is incorrect, with new information not already considered.

然而，我们并不认为这些论点超过了 "lock" 这个词的问题。如果提出反馈要求我们重新考虑这一决定，请清楚地说明上述决定不正确的原因，以及我们还没有考虑到新信息。


