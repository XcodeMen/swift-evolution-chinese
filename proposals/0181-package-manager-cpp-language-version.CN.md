# 0181 - Package Manager C/C++ Language Standard Support

* Proposal: [SE-0181](0181-package-manager-cpp-language-version.md)
* Authors: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Daniel Dunbar](https://github.com/ddunbar)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170717/038142.html)
* Implementation: [apple/swift-package-manager#1264](https://github.com/apple/swift-package-manager/pull/1264)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal adds support for declaring the language standard for C and C++
targets in a SwiftPM package.

本提案增加了支持在 SwiftPM 软件包中声明 C 和 C++ targets 的语言标准。

## Motivation 动机

The C++ language standard is one of the most important build setting needed to
compile C++ targets. We want to add some mechanism to declare it until we get
the complete build settings feature, which is deferred from the Swift 4 release.

C++ 语言标准是编译 C++ targets 所需的最重要的编译配置之一。我们需要添加一些机制来声明它，除非我们能获得完整的编译配置功能，而这项功能在 Swift 4 发布版本中被推迟了。

## Proposed solution 建议的解决方案

We will add support to the package manifest declaration to specify the C and C++
language standards:

我们将对包清单声明添加支持以指定 C 和 C++ 语言标准：

```swift
let package = Package(
    name: "CHTTP",
    ...
    cLanguageStandard: .c89,
    cxxLanguageStandard: .cxx11
)
```

These settings will apply to all the C and C++ targets in the package. The
default value of these properties will be `nil`, i.e., a language standard flag
will not be passed when invoking the C/C++ compiler.

这些配置将适用于包中所有的 C 和 C++ targets。这些属性的默认值为 `nil`，即当调用 C/C++ 编译器时不会传递相应的语言标准标识符。

_Once we get the build settings feature, we will deprecate these properties._

_一旦我们能获得编译配置功能，我们将废弃这些属性。_

## Detailed design 详细设计

The C/C++ language standard will be defined by the enums below and
updated as per the Clang compiler [repository](https://github.com/llvm-mirror/clang/blob/master/include/clang/Frontend/LangStandards.def).

C/C++ 语言标准将由下面的枚举定义，并根据 Clang 编译器[仓库](https://github.com/llvm-mirror/clang/blob/master/include/clang/Frontend/LangStandards.def)进行更新。

```swift
public enum CLanguageStandard {
    case c89
    case c90
    case iso9899_1990
    case iso9899_199409
    case gnu89
    case gnu90
    case c99
    case iso9899_1999
    case gnu99
    case c11
    case iso9899_2011
    case gnu11
}

public enum CXXLanguageStandard {
    case cxx98
    case cxx03
    case gnucxx98
    case gnucxx03
    case cxx11
    case gnucxx11
    case cxx14
    case gnucxx14
    case cxx1z
    case gnucxx1z
}
```
## Impact on existing code 对现有代码的影响

There will be no impact on existing packages because this is a new API and the
default behaviour remains unchanged.

这一改动对现有的包将不会有任何影响，因为这是一个新的 API，且其默认行为保持不变。

## Alternatives considered 考虑过的替代方案

We considered adding this property at target level but we think that will
pollute the target namespace. Moreover, this is a temporary measure until we get
the build settings feature.

我们考虑过在 target 级别添加此属性，但我们认为这样会污染 target 命名空间。此外，这是一个临时措施，直到我们能获得编译配置功能。


