# 0149 - Package Manager Support for Top of Tree development

* Proposal: [SE-0149](0149-package-manager-top-of-tree.md)
* Author: [Boris Bügling](https://github.com/neonichu)
* Review Manager: [Daniel Dunbar](https://github.com/ddunbar)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170130/031427.html)
* Bug: [SR-3709](https://bugs.swift.org/browse/SR-3709)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal adds enhancements to `swift package edit` to support development of packages without strict versioning ("top of tree" development).

本提案增强了 `swift package edit` 功能，以支持开发不需要严格版本控制的包（即 "top of tree" 开发）。

## Motivation 动机

The package manager currently supports package dependencies which are strictly versioned according to semantic versioning. This works well for users of packages and it is possible to edit a package in place using `swift package edit` already, but we want to allow developers to manually check out repositories on their machines as overrides. This is useful when developing multiple packages in tandem or when working on packages alongside an application.

现有的包管理器支持根据语义版本进行严格版本控制的包依赖关系。这对于软件包的用户来说非常有用，且现在已经可以通过使用 `swift package edit` 来编辑一个包，但我们希望开发者可以在他们的机器上通过重写来手动检出代码仓库。这对于串行开发多个包或者为一个应用程序的包进行开发来说是很有用的。

When a developer owns multiple packages that depend on each other, it can be necessary to work on a feature across more than one of them at the same time, without having to tag versions in between. The repositories for each package would usually already be checked out and managed manually by the developer, allowing them to switch branches or perform other SCM operations at will.

当一个开发者拥有多个彼此互相依赖的包时，就可能需要同时跨多个包来开发一个功能，而无需在两者之间标记版本。每一个包的代码仓库通常都将由开发者手动检出并管理，并允许他们随意切换分支或者执行其他 SCM 操作。

A similar situation will arise when working on a feature that requires code changes to both an application and a dependent package. Developers want to iterate on the package by directly in the context of the application without having to release spurious versions of the package. Allowing developers to provide their own checkouts to the package manager as overrides will make this workflow much easier than it currently is.

当开发一个功能需要同时更改应用程序及其所以依赖包的代码时，也会出现类似的情况。开发人员希望通过直接在应用程序的上下文中对包进行迭代，而无需发布软件包的临时伪版本。允许开发人员将自己的检出提供给包管理器作为覆盖，将使此开发流程比目前的更加容易。

## Proposed solution 建议的解决方案

As a solution to this problem, we propose to extend `swift package edit` to take an optional path argument to an existing checkout so that users can manually manage source control operations.

作为这个问题的解决方案，我们建议扩展 `swift package edit`，在现有的检出上采用一个可选路径参数，以便用户可以手动管理源码控制操作。

## Detailed Design 详细设计

We will extend the `edit` subcommand with a new optional argument `--path`:

我们将使用一个新的可选参数 `--path` 来扩展 `edit` 子命令：

```bash
$ swift package edit bar --path ../bar
```

This allows users to manage their own checkout of the `bar` repository and will make the package manager use that instead of checking out a tagged version as it normally would. Concretely, this will make `./Packages/bar` a symbolic link to the given path in the local filesystem, store this mapping inside the workspace and will ensure that the package manager will no longer be responsible for managing checkouts for the `bar` package, instead the user is responsible for managing the source control operations on their own. This is consistent with the current behavior of `swift edit`. Using `swift package unedit` will also work unchanged, but the checkout itself will not be deleted, only the symlink. If there is no existing checkout at the given filesystem location, the package manager will do an initial clone on the user's behalf.

这允许用户管理他们自己对 `bar` 代码仓库的检出（checkout），并让包管理器使用它，而不是像通常那样检出标记的版本。具体来说，这将创建一个 `./Packages/bar` 符号链接到本地文件系统给定的路径，然后在工作区内存储此映射关系，并将确保包管理器不再负责管理 `bar` 包的检出，而是由用户自己负责管理源码控制操作。这与 `swift edit` 当前的行为是一致的。使用 `swift package unedit` 也将保持不变，但检出本身不会被删除，只有符号链接会。如果在给定的文件系统位置不存在相应的检出，则包管理器将代替用户进行一次初始化克隆操作。

## Impact on existing code 对现有代码的影响

There will no impact on existing code.

不会影响现有代码。

## Alternative considered 考虑过的替代方案

We could have used the symlink in the `Packages` directory as primary data, but decided against it in order to be able to provide better diagnostics and distinguish use of `swift package edit` from the user manually creating symbolic links. This makes the symlink optional, but we decided to still create it in order to keep the structure of the `Packages` directory consistent independently of the use of this feature.

我们本可以使用 `Packages` 目录中的符号链接作为原始数据，但为了能够提供更好的诊断和区分用户使用 `swift package edit` 手动创建的符号链接，我们决定反对这么做。这使得符号链接是可选的，但我们仍然决定创建它，以保持 `Packages` 目录的结构与使用此特性无关。


