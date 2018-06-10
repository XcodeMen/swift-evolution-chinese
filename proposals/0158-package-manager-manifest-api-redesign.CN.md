# 0158 - Package Manager Manifest API Redesign

* Proposal: [SE-0158](0158-package-manager-manifest-api-redesign.md)
* Author: [Ankit Aggarwal](https://github.com/aciidb0mb3r)
* Review Manager: [Rick Ballard](https://github.com/rballard)
* Status: **Implemented (Swift 4)**
* Bug: [SR-3949](https://bugs.swift.org/browse/SR-3949)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170313/033989.html)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This is a proposal for redesigning the `Package.swift` manifest APIs provided
by Swift Package Manager.  
This proposal only redesigns the existing public APIs and does not add any
new functionality; any API to be added for new functionality will happen in
separate proposals.

这是一个用于重新设计 Swift 包管理器提供的 `Package.swift` 清单 API 的提案。
本提案只重新设计了现有的公共 API，并没有增加任何新的功能；任何为新功能添加的 API 都将出现在单独的提案中。

## Motivation 动机

The `Package.swift` manifest APIs were designed prior to the [API Design
Guidelines](https://swift.org/documentation/api-design-guidelines/), and their
design was not reviewed by the evolution process. Additionally, there are
several small areas which can be cleaned up to make the overall API more
"Swifty".

`Package.swift` 清单的 API 是在 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/) 之前设计的，并且这些在 Swift 演化过程中没有被重新评估。此外，还有几个可以小区域可以被清理掉，使得整个 API 更加 "Swifty"。

We would like to redesign these APIs as necessary to provide clean,
conventions-compliant APIs that we can rely on in the future. Because we
anticipate that the user community for the Swift Package Manager will grow
considerably in Swift 4, we would like to make these changes now, before
more packages are created using the old API.

我们希望在必要时重新设计这些 API，以提供将来我们可以依赖的简洁的、符合规定的 API。因为我们预计 Swift 包管理器的用户社区将在 Swift 4 中大幅增长，我们希望在使用旧 API 创建更多包之前进行这些更改。

## Proposed solution 建议的解决方案

Note: Access modifier is omitted from the diffs and examples for brevity. The
access modifier is `public` for all APIs unless specified.

注意：为了简洁起见，在 diffs 和 examples 中省略了访问修饰符。

* Remove `successor()` and `predecessor()` from `Version`.
* 从 `Version` 中 移除 `successor()` 和 `predecessor()`。

    These methods neither have well defined semantics nor are used a lot
    (internally or publicly). For e.g., the current implementation of
    `successor()` always just increases the patch version.

    这些方法既没有明确定义的语义，也没有被大量使用（内部或公开）。例如，目前 `successor()` 的实现总仅是增加补丁的版本。

    <details>
      <summary>View diff</summary>
      <p>

    ```diff
    struct Version {
    -    func successor() -> Version

    -    func predecessor() -> Version
    }
    ```
    </p></details>

* Convert `Version`'s `buildMetadataIdentifier` property to an array.
* 将 `Version` 的 `buildMetadataIdentifier` 转换为数组。

    According to SemVer 2.0, build metadata is a series of dot separated
    identifiers. Currently this is represented as an optional string property
    in the `Version` struct. We propose to change this property to an array
    (similar to `prereleaseIdentifiers` property). To maintain backwards
    compatiblility in PackageDescription 3 API, we will keep the optional
    string as a computed property based on the new array property. We will also
    keep the version initializer that takes the `buildMetadataIdentifier` string.
    
    根据 SemVer 2.0，编译元数据是一系列点分隔的标识符。目前这在 `Version` 结构体中表示为一个可选的字符串属性。我们建议将这个属性改成一个数组（类似于 `prereleaseIdentifiers` 属性）。为了在 PackageDescription 3 API 中保持向后兼容性，我们将基于新的数组属性保留可选字符串作为计算属性。我们还将保留采用 `buildMetadataIdentifier` 字符串的版本初始化方法。

* Make all properties of `Package` and `Target` mutable.
* 使 `Package` 和 `Target` 的所有属性都可变。

    Currently, `Package` has three immutable and four mutable properties, and
    `Target` has one immutable and one mutable property. We propose to make all
    properties mutable to allow complex customization on the package object
    after initial declaration.
    
    目前，`Package` 有三个不可变属性和四个可变属性，`Target` 有一个不可变属性和一个可变属性。我们建议在初始化声明后，使所有的属性都是可变的，以允许对包对象进行复杂的定制。

    <details>
      <summary>View diff and example</summary>
      <p>

    Diff:
    ```diff
    final class Target {
    -    let name: String
    +    var name: String
    }

    final class Package {
    -    let name: String
    +    var name: String

    -    let pkgConfig: String?
    +    var pkgConfig: String?

    -    let providers: [SystemPackageProvider]?
    +    var providers: [SystemPackageProvider]?
    }
    ```

    Example:
    ```swift
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(name: "Foo", dependencies: ["Bar"]),
        ]
    )

    #if os(Linux)
    package.targets[0].dependencies = ["BarLinux"]
    #endif
    ```
    </p></details>

* Change `Target.Dependency` enum cases to lowerCamelCase.
* 将 `Target.Dependency` 的枚举值改为 lowerCamelCase（小写驼峰型）。

    According to API design guidelines, everything other than types should be in lowerCamelCase.
    
    根据 API 设计指南，除了类型之外所有东西都应该为 lowerCamelCase 形式。

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    enum Dependency {
    -    case Target(name: String)
    +    case target(name: String)

    -    case Product(name: String, package: String?)
    +    case product(name: String, package: String?)

    -    case ByName(name: String)
    +    case byName(name: String)
    }
    ```

    Example:
    ```diff
    let package = Package(
        name: "FooPackage",
        targets: [
            Target(
                name: "Foo", 
                dependencies: [
    -                .Target(name: "Bar"),
    +                .target(name: "Bar"),

    -                .Product(name: "SwiftyJSON", package: "SwiftyJSON"),
    +                .product(name: "SwiftyJSON", package: "SwiftyJSON"),
                ]
            ),
        ]
    )
    ```
    </p></details>

* Add default parameter to the enum case `Target.Dependency.product`.
* 为 `Target.Dependency.product` 枚举 case 添加默认参数。

    The associated value `package` in the (enum) case `product`, is an optional
    `String`. It should have the default value `nil` so clients don't need to
    write it if they prefer using explicit enum cases but don't want to specify
    the package name i.e. it should be possible to write `.product(name:
    "Foo")` instead of `.product(name: "Foo", package: nil)`.
    
    枚举 `product` case 中的相关值 `package` 是一个可选的 `String`。它应该有默认值 `nil`，所以如果他们希望使用显式的枚举 cases，但不想指定包明时，客户端就不需要写它，也就是说，它可以写成 `.product(name: "Foo")` 而不是 `.product(name: "Foo", package: nil)`。

    If
    [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md)
    is accepted, we can directly add a default value. Otherwise, we will use a
    static factory method to provide default value for `package`.
    
    如果 [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md) 提案被接受，我们就可以直接添加一个默认值。否则，我们将使用静态工厂方法为 `package` 提供默认值。

* Rename all enums cases to have a suffix `Item` and favor static methods.
* 重命名所有枚举 cases，添加一个后缀 `Item`，并支持静态方法。

    Since static methods are more extensible than enum cases right now, we
    should discourage use of direct use of enum initializers and provide a
    static method for each case. It is not possible to have overloads on enum
    cases, so as a convention we propose to rename all enum cases to have a
    suffix "Item" for future extensibility.
    
    由于现在静态方法比枚举值更具可扩展行，所以我们不应该直接使用枚举初始化方法，而是为每种情况提供一个静态方法。枚举 cases 不能有重载，所以作为一个约定，我们建议重命名所有的枚举 cases，添加一个后缀 "Item"，以便将来可扩展。

* Change `SystemPackageProvider` enum cases to lowerCamelCase and their payloads to array.
* 将 `SystemPackageProvider` 枚举 cases 更改为 lowerCamelCase 格式，以及把它们的 payloads 改为数组。

    According to API design guidelines, everything other than types should be
    in lowerCamelCase.
    
    根据 API 设计指南，除了类型之外所有东西都应该为 lowerCamelCase 形式。

    This enum allows SwiftPM System Packages to emit hints in case of build
    failures due to absence of a system package. Currently, only one system
    package per system packager can be specified. We propose to allow
    specifying multiple system packages by changing the payload to be an array.
    
    这个枚举允许 SwiftPM System Packages 在由于没有 system package 而造成的编译失败的情况下发出提示。目前，每一个 system packager 只能指定一个 system package。我们建议允许通过将 payload 更改为数组来指定多个 system packages。

    <details>
      <summary>View diff and example</summary>
      <p>

     Diff:
    ```diff
    enum SystemPackageProvider {
    -    case Brew(String)
    +    case brew([String])

    -    case Apt(String)
    +    case apt([String])
    }
    ```

    Example:

    ```diff
    let package = Package(
        name: "Copenssl",
        pkgConfig: "openssl",
        providers: [
    -        .Brew("openssl"),
    +        .brew(["openssl"]),

    -        .Apt("openssl-dev"),
    +        .apt(["openssl", "libssl-dev"]),
        ]
    )
    ```
    </p></details>


* Remove implicit target dependency rule for test targets.
* 移除测试 targets 的隐式 target 依赖规则。

    There is an implicit test target dependency rule: a test target "FooTests"
    implicity depends on a target "Foo", if "Foo" exists and "FooTests" doesn't
    explicitly declare any dependency. We propose to remove this rule because:
    
    目前有一个隐式测试 target 依赖规则：如果 "Foo" 存在并且 "FooTests" 没有明确声明任何依赖关系，则测试 target "FooTests" 隐式地依赖于目标 "Foo"。我们建议移除这个规则，因为：

    1. It is a non obvious "magic" rule that has to be learned. 这是一条不明显“魔法”规则，需要学习。
    2. It is not possible for "FooTests" to remove dependency on "Foo" while
       having no other (target) dependency. 当 "FooTests" 没有其他（target）依赖时，无法移除对 "Foo" 的依赖。
    3. It makes real dependencies less discoverable. 它使得真正的依赖关系较难被发现。
    4. It may cause issues when we get support for mechanically editing target
       dependencies. 当我们支持手动编辑依赖关系时，可能导致问题。

* Use factory methods for creating objects.
* 使用工厂方法来创建对象。

    We propose to always use factory methods to create objects except for the
    main `Package` object. This gives a lot of flexibility and extensibility to
    the APIs because Swift's type system can infer the top level type in a
    context and allow using the shorthand dot syntax.
    
    我们建议始终使用工厂方法来创建对象，处了主 `Package` 对象。这给了 API 很大的灵活性和可扩展性，因为 Swift 的类型系统可以在上下文中推断顶级类型，并允许使用简写的点语法。

    Concretely, we will make these changes:
    
    具体来说，我们将做如下更改：

    * Add a factory method `target` to `Target` class and change the current
      initializer to private.
    * 为 `Target` 类添加一个 `target` 工厂方法，并将目前的初始化方法更改为私有的。

        <details>
          <summary>View example and diff</summary>
          <p>

        Example:

        ```diff
        let package = Package(
            name: "Foo",
            target: [
        -        Target(name: "Foo", dependencies: ["Utility"]),
        +        .target(name: "Foo", dependencies: ["Utility"]),
            ]
        )
        ```
        </p></details>

    * Introduce a `Product` class with two subclasses: `Executable` and
      `Library`.  These subclasses will be nested inside `Product` class
      instead of being a top level declaration in the module. Nesting will give
      us a namespace for products and it is easy to find all the supported
      products when the product types grows to a large number. We will add two
      factory methods to `Product` class: `library` and `executable` to create
      respective products.
      
    * 引入一个 `Product` 类及其两个子类：`Executable` 和 `Library`。这些子类将嵌套在 `Product` 类中，而不是在模块中的顶级声明。嵌套将为我们提供一个产品的命名空间，当产品类型增长到很大量时，可以很容易找到所有支持的产品。我们将为 `Product` 类添加两个工厂方法：`library` 和 `executable` 来分别创建相应的产品。

        ```swift
        /// Represents a product.
        class Product {
        
            /// The name of the product.
            let name: String

            private init(name: String) {
                self.name = name
            }
        
            /// Represents an executable product.
            final class Executable: Product {

                /// The names of the targets in this product.
                let targets: [String]

                private init(name: String, targets: [String])
            }
        
            /// Represents a library product.
            final class Library: Product {
                /// The type of library product.
                enum LibraryType: String {
                    case `static`
                    case `dynamic`
                }

                /// The names of the targets in this product.
                let targets: [String]
        
                /// The type of the library.
                ///
                /// If the type is unspecified, package manager will automatically choose a type.
                let type: LibraryType?
        
                private init(name: String, type: LibraryType? = nil, targets: [String])
            }

            /// Create a library product.
            static func library(name: String, type: LibraryType? = nil, targets: [String]) -> Library

            /// Create an executable product.
            static func executable(name: String, targets: [String]) -> Library
        }
        ```

        <details>
          <summary>View example</summary>
          <p>

        Example:

        ```swift
        let package = Package(
            name: "Foo",
            target: [
                .target(name: "Foo", dependencies: ["Utility"]),
                .target(name: "tool", dependencies: ["Foo"]),
            ],
            products: [
                .executable(name: "tool", targets: ["tool"]), 
                .library(name: "Foo", targets: ["Foo"]), 
                .library(name: "FooDy", type: .dynamic, targets: ["Foo"]), 
            ]
        )
        ```
        </p></details>


* Special syntax for version initializers.
* 版本初始化方法的特殊语法。

    A simplified summary of what is commonly supported in other package managers:
    
    在其他包管理器中通常支持的简要总结：

    | Package Manager | x-ranges      | tilde (`~` or `~>`)     | caret (`^`)   |
    |-----------------|---------------|-------------------------|---------------|
    | npm             | Supported     | Allows patch-level changes if a minor version is specified on the comparator. Allows minor-level changes if not.  | patch and minor updates |
    | Cargo           | Supported     | Same as above           | Same as above |
    | CocoaPods       | Not supported | Same as above           | Not supported |
    | Carthage        | Not supported | patch and minor updates | Not supported |

    Some general notes:
    
    一些一般说明：

    Every package manager we looked at supports the tilde `~` operator in some form, and it's generally
    recommended as "the right thing", because package maintainers often fail to increment their major package
    version when they should, incrementing their minor version instead. See e.g. how Google created a
    [6-minute instructional video](https://www.youtube.com/watch?v=x4ARXyovvPc) about this operator for CocoaPods.
    This is a form of version overconstraint; your package should be compatible with everything with the same
    major version, but people don't trust that enough to rely on it. But version overconstraint is harmful,
    because it leads to "dependency hell" (unresolvable dependencies due to conflicting requirements for a package
    in the dependency graph).
    
    我们所看到的每一个包管理器都以某种形式支持波浪号 `~` 操作符，通常推荐使用它是“正确的事情”，因为包的维护者往往不应该增加他们包的主要版本，而是增加他们的次要版本。例如谷歌如何为 CocoaPods 的这个操作符创建一个 `[6-minute instructional video](https://www.youtube.com/watch?v=x4ARXyovvPc)`。这是一种版本过渡约束的形式；你的包应该与具有相同主要版本的所有软件兼容，但是人们很难相信它足以依赖。但版本过渡约束是有害的，因为它导致了 "dependency hell"（由于依赖关系图中的包的需求冲突导致无法解析的依赖关系）。
    
    We'd like to encourage a better standard of behavior in the Swift Package Manager. In the future, we'd like
    to add tooling to let the package manager automatically help you use Semantic Versioning correctly,
    so that your clients can trust your major version. If we can get package maintainers to use SemVer correctly,
    through automatic enforcement in the future or community norms for now, then caret `^` becomes
    the best operator to use most of the time. That is, you should be able to specify a minimum version,
    and you should be willing to let your package use anything after that up to the next major version. This
    means you'll get safe updates automatically, and you'll avoid overconstraining and introducing dependency hell.
    
    我们希望在 Swift 包管理器中鼓励良好的行为标准。将来，我们希望添加工具来让包管理器自动帮助您正确使用 Semantic Versioning，以便您的客户端可以信任您的主要版本。如果我们可以让包的维护人员正确地使用 SemVer，通过今后的自动执行或者目前的社区规范，那么大部分时间下，插入符号 `^` 就成了最好的运算符。也就是说，你应该能够指定一个最低版本，并且你应该愿意让你的包使用任何东西，直到更新下一个主要版本。这意味着你将自动获得安全更新，你将避免 overconstraining 和引入 dependency hell。
    
    Caret `^` and tilde `~` syntax is somewhat standard, but is syntactically non-obvious; we'd prefer a syntax
    that doesn't require reading a manual for novices to understand, even if that means we break with the
    syntactic convention established by the other package managers which support caret `^` and tilde `~`.
    We'd like to make it possible to follow the tilde `~` use case (with different syntax), but caret `^`
    should be the most convenient, to encourage its use.
    
    插入符号 `^` 和波浪号 `~` 语法在某种程度上是标准的，但在语法结构上不够明显，我们更期望一个不需要新手通过阅读手册来理解的语法，即使这意味着我们打破了其他支持插入符号 `^` 和波浪号 `~` 的包管理器所建立的约定。我们希望能够沿用波浪号 `~` 的用法（使用不同的语法），但插入符号 `^` 应该是最方便的，以鼓励它的使用。

    What we propose:
    
    我们建议：

    * We will introduce a factory method which takes a lower bound version and
      forms a range that goes upto the next major version (i.e. caret).
    * 我们将引入一个工厂方法，该方法采用下限版本，并形成一个上升至下一个主要版本的范围（即插入符号）。

      ```swift
      // 1.0.0 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.0.0"),

      // 1.2.0 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.2.0"),

      // 1.5.8 ..< 2.0.0
      .package(url: "/SwiftyJSON", from: "1.5.8"),
      ```

    * We will introduce a factory method which takes `Requirement`, to
      conveniently specify common ranges.
    * 我们将引入一个接收 `Requirement` 的工厂方法，以方便地指定常用范围。

      `Requirement` is an enum defined as follows:
      
      `Requirement` 是一个枚举，定义如下：

      ```swift
      enum Requirement {
          /// The requirement is specified by an exact version.
          /// requirement 由一个确切的版本指定。
          case exact(Version)

          /// The requirement is specified by a version range.
          /// requirement 由一个版本范围指定。
          case range(Range<Version>)

          /// The requirement is specified by a source control revision.
          /// requirement 由源码控制修订指定。
          case revision(String)

          /// The requirement is specified by a source control branch.
          /// requirement 由源码控制分支指定。
          case branch(String)

          /// Creates a specified for a range starting at the given lower bound
          /// and going upto next major version.
          /// 创建一个指定范围，从给定的下限开始，到下一个主要版本。
          static func upToNextMajor(from version: Version) -> Requirement

          /// Creates a specified for a range starting at the given lower bound
          /// and going upto next minor version.
          /// 创建一个指定范围，从给定的下限开始，到下一个次要版本。
          static func upToNextMinor(from version: Version) -> Requirement
      }
      ```

      Examples:

      ```swift
      // 1.5.8 ..< 2.0.0
      .package(url: "/SwiftyJSON", .upToNextMajor(from: "1.5.8")),

      // 1.5.8 ..< 1.6.0
      .package(url: "/SwiftyJSON", .upToNextMinor(from: "1.5.8")),

      // 1.5.8
      .package(url: "/SwiftyJSON", .exact("1.5.8")),
      ```

    * This will also give us ability to add more complex features in future:
    * 这将使我们有能力在将来添加更复杂的功能：

      Examples:
      例如：
      > Note that we're not actually proposing these as part of this proposal.
      > 请注意，我们实际上并没有建议把这些作为本提案的一部分。

      ```swift
      .package(url: "/SwiftyJSON", .upToNextMajor(from: "1.5.8").excluding("1.6.4")),

      .package(url: "/SwiftyJSON", .exact("1.5.8", "1.6.3")),
      ```

    * We will introduce a factory method which takes `Range<Version>`, to specify
      arbitrary open range.
    * 我们将引入一个接收 `Range<Version>` 的工厂方法，以便指定任意公开的范围。

      ```swift
      // Constraint to an arbitrary open range.
      // 对任意公开范围的约束。
      .package(url: "/SwiftyJSON", "1.2.3"..<"1.2.6"),
      ```

    * We will introduce a factory method which takes `ClosedRange<Version>`, to specify arbitrary closed range.
    * 我们将引入一个接收 `ClosedRange<Version>` 的工厂方法，以便指定任意封闭的范围。

      ```swift
      // Constraint to an arbitrary closed range.
      // 对任意封闭范围的约束。
      .package(url: "/SwiftyJSON", "1.2.3"..."1.2.8"),
      ```
      
    * As a slight modification to the
      [branch proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0150-package-manager-branch-support.md),
      we will add cases for specifying a branch or revision, rather than
      adding factory methods for them:
    * 作为 [branch proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0150-package-manager-branch-support.md) 的一个细微修改，我们将添加用于指定分支或修改的枚举 cases，而不是为它们添加工厂方法。

      ```swift
      .package(url: "/SwiftyJSON", .branch("develop")),
      .package(url: "/SwiftyJSON", .revision("e74b07278b926c9ec6f9643455ea00d1ce04a021"))
      ```

    * We will remove all of the current factory methods:
    * 我们将移除当前所有的工厂方法：

      ```swift
      // Constraint to a major version.
      // 对主要版本（大版本）的约束。
      .Package(url: "/SwiftyJSON", majorVersion: 1),

      // Constraint to a major and minor version.
      // 对主要版本（大版本）和次要版本（小版本）的约束。
      .Package(url: "/SwiftyJSON", majorVersion: 1, minor: 2),

      // Constraint to an exact version.
      // 对指定版本的约束。
      .Package(url: "/SwiftyJSON", "1.2.3"),

      // Constraint to an arbitrary range.
      // 对任意范围的约束
      .Package(url: "/SwiftyJSON", versions: "1.2.3"..<"1.2.6"),

      // Constraint to an arbitrary closed range.
      // 对任意封闭范围的约束。
      .Package(url: "/SwiftyJSON", versions: "1.2.3"..."1.2.8"),
      ```

* Adjust order of parameters on `Package` class:
* 调整 `Package` 类的参数顺序：

    We propose to reorder the parameters of `Package` class to: `name`,
    `pkgConfig`, `products`, `dependencies`, `targets`, `swiftLanguageVersions`.
    
    我们建议将 `Package` 类的参数重新排序为：`name`,
    `pkgConfig`, `products`, `dependencies`, `targets`, `swiftLanguageVersions`。

    The rationale behind this reorder is that the most interesting parts of a
    package are its product and dependencies, so they should be at the top.
    Targets are usually important during development of the package.  Placing
    them at the end keeps it easier for the developer to jump to end of the
    file to access them. Note that the `swiftLanguageVersions` property will likely
    be removed once we support Build Settings, but that will be discussed in a separate proposal.
    
    这种重新排序的基本原理是，一个包中最引人关注的部分是它的产品和依赖关系，所以它们应该位于最前面。Targets 通常在开发包的期间中很重要。将它们放在最后可以让开发者更容易跳到文件的结尾来访问它们。请注意，一旦我们支持编译设置（Build Settings），`swiftLanguageVersions` 属性可能会被移除，但这将在单独的提案中讨论。

    <details>
      <summary>View example</summary>
      <p>

    Example:

    ```swift
    let package = Package(
        name: "Paper",
        products: [
            .executable(name: "tool", targets: ["tool"]),
            .library(name: "Paper", type: .static, targets: ["Paper"]),
            .library(name: "PaperDy", type: .dynamic, targets: ["Paper"]),
        ],
        dependencies: [
            .package(url: "http://github.com/SwiftyJSON/SwiftyJSON", from: "1.2.3"),
            .package(url: "../CHTTPParser", .upToNextMinor(from: "2.2.0")),
            .package(url: "http://some/other/lib", .exact("1.2.3")),
        ]
        targets: [
            .target(
                name: "tool",
                dependencies: [
                    "Paper",
                    "SwiftyJSON"
                ]),
            .target(
                name: "Paper",
                dependencies: [
                    "Basic",
                    .target(name: "Utility"),
                    .product(name: "CHTTPParser"),
                ])
        ]
    )
    ```
    </p></details>

* Eliminate exclude in future (via custom layouts feature).
* 在将来移除 exclude（通过自定义布局特性）。

    We expect to remove the `exclude` property after we get support for custom
    layouts. The exact details will be in the proposal of that feature.
    
    我们希望在获得对自定义布局（custom layouts）的支持后移除 `exclude` 属性。确切的细节将在该特性所在的提案中叙述。

## Example manifests 示例清单

* A regular manifest. 一个常规的清单。

```swift
let package = Package(
    name: "Paper",
    products: [
        .executable(name: "tool", targets: ["tool"]),
        .library(name: "Paper", targets: ["Paper"]),
        .library(name: "PaperStatic", type: .static, targets: ["Paper"]),
        .library(name: "PaperDynamic", type: .dynamic, targets: ["Paper"]),
    ],
    dependencies: [
        .package(url: "http://github.com/SwiftyJSON/SwiftyJSON", from: "1.2.3"),
        .package(url: "../CHTTPParser", .upToNextMinor(from: "2.2.0")),
        .package(url: "http://some/other/lib", .exact("1.2.3")),
    ]
    targets: [
        .target(
            name: "tool",
            dependencies: [
                "Paper",
                "SwiftyJSON"
            ]),
        .target(
            name: "Paper",
            dependencies: [
                "Basic",
                .target(name: "Utility"),
                .product(name: "CHTTPParser"),
            ])
    ]
)
```

* A system package manifest. 一个系统包的清单。

```swift
let package = Package(
    name: "Copenssl",
    pkgConfig: "openssl",
    providers: [
        .brew(["openssl"]),
        .apt(["openssl", "libssl-dev"]),
    ]
)
```

## Impact on existing code 对现有代码的影响

The above changes will be implemented only in the new Package Description v4
library. The v4 runtime library will release with Swift 4 and packages will be
able to opt-in into it as described by
[SE-0152](https://github.com/apple/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md).

上述更改将仅在新的 Package Description v4 库中执行。v4 运行时库将随 Swift 4 一起发布，包将能够根据 [SE-0152](https://github.com/apple/swift-evolution/blob/master/proposals/0152-package-manager-tools-version.md) 的描述选择性地加入。

There will be no automatic migration feature for updating the manifests from v3
to v4. To indicate the replacements of old APIs, we will annotate them using
the `@unavailable` attribute where possible. Unfortunately, this will not cover
all the changes for e.g. rename of the target dependency enum cases.

没有自动迁移的功能将包清单从 v3 更新到 v4。为了提示替换旧的 API，我们将尽可能地使用 `@unavailable` 属性对其进行注释。不幸的是，它无法覆盖所有的变化，例如 target 依赖关系枚举 cases 的重命名。

All new packages created with `swift package init` command in Swift 4 tools
will by default to use the v4 manifest. It will be possible to switch to v3
manifest version by changing the tools version using `swift package
tools-version --set 3.1`.  However, the manifest will needed to be adjusted to
use the older APIs manually.

所有使用 Swift 4 工具中的 `swift package init` 命令创建的新包将默认使用 v4 清单。通过使用 `swift package tools-version --set 3.1` 更改工具版本，可以把清单切换到 v3 版本。然而，清单将需要手动调整以使用旧的 API。

Unless declared in the manifest, existing packages automatically default
to the Swift 3 minimum tools version; since the Swift 4 tools will also include
the v3 manifest API, they will build as expected.

除非在清单中声明，否则现有的包将自动默认使用 Swift 3 最小的工具版本；由于 Swfit 4 工具也将包含 v3 清单的 API，所以它们将按预期进行编译。

A package which needs to support both Swift 3 and Swift 4 tools will need to
stay on the v3 manifest API and support the Swift 3 language version for its
sources, using the API described in the proposal
[SE-0151](https://github.com/apple/swift-evolution/blob/master/proposals/0151-package-manager-swift-language-compatibility-version.md).

如果一个包需要同时支持 Swift 3 和 Swift 4 工具，需要将其清单 API 保留在 v3，并让其源码支持 Swift 3 语言版本，使用提案 [SE-0151](https://github.com/apple/swift-evolution/blob/master/proposals/0151-package-manager-swift-language-compatibility-version.md) 中描述的 API。

An existing package which wants to use the new v4 manifest APIs will need to bump its
minimum tools version to 4.0 or later using the command `$ swift package tools-version
--set-current`, and then modify the manifest file with the changes described in
this proposal.

一个现有的包如果想要使用新的 v4 版本的清单 API，需要使用命令 `$ swift package tools-version --set-current` 将其最低工具版本提升到 4.0 或更高版本，然后根据本提案中所述的更改修改清单文件。

## Alternatives considered 考虑过的替代方案

* Add variadic overloads.
* 添加可变重载

    Adding variadic overload allows omitting parenthesis which leads to less
    cognitive load on eyes, especially when there is only one value which needs
    to be specified. For e.g.:
    
    添加可变重载允许省略括号，从而减少视觉上的认知负担，尤其是当只有一个值需要被指定时。例如：

        Target(name: "Foo", dependencies: "Bar")

    might looked better than:
    
    也许看起来比下面的更好：

        Target(name: "Foo", dependencies: ["Bar"])

    However, plurals words like `dependencies` and `targets` imply a collection
    which implies brackets. It also makes the grammar wrong. Therefore, we
    reject this option.
    
    然而，像 `dependencies` 和 `targets` 这样的复数词意味着是一个隐含括号的集合。它也会带来语法上的错误。因此，我们拒绝这一方案。
    
* Version exclusion.
* 版本排除
    
    It is not uncommon to have a specific package version break something, and
    it is undesirable to "fix" this by adjusting the range to exclude it
    because this overly constrains the graph and can prevent picking up the
    version with the fix.
    
    一个特定的包版本破坏某些规则的情况并不少见，通过调整范围来排除它以“修复”这种情况是不可取的，因为这会过度地限制 graph，并且可能会阻止通过修复来选取版本。

    This is desirable but it should be proposed separately.
    
    这是可取的，但应在单独的提案中提出。

* Inline package declaration.
* 内联包声明

    We should probably support declaring a package dependency anywhere we
    support spelling a package name. It is very common to only have one target
    require a dependency, and annoying to have to specify the name twice.
    
    我们或许应该支持在我们支持拼写一个包名称的任何地方声明一个包依赖。只有一个 target 需要一个依赖项，这是很常见的，但烦人的是必须指定这个名词两次。

    This is desirable but it should be proposed separately.
    
    这是可取的，但应在单独的提案中提出。

* Introduce an "identity rule" to determine if an API should use an initializer
  or a factory method:
* 引入一个 "identity rule" 来确定一个 API 是否使用初始化方法（initializer）或工厂方法（factory method）：

    Under this rule, an entity having an identity, will use a type initializer
    and everything else will use factory methods. `Package`, `Target` and
    `Product` are identities. However, a product referenced in a target
    dependency is not an identity.
    
    在这个规则下，具有 identity 的实体将使用类型初始化方法，其他所有的都将使用工厂方法。`Package`，`Target` 和 `Product` 都是 identities。然而，target 依赖项中引用的产品并不是 identity。

    We rejected this because it may become a source of confusion for users.
    Another downside is that the product initializers will have to used with
    the dot notation (e.g.: `.Executable(name: "tool", targets: ["tool"])`)
    which is a little awkward because we expect factory methods and enum cases
    to use the dot syntax. This can be solved by moving these products outside
    of `Product` class but we think having a namespace for product provides a
    lot of value.
    
    我们拒绝这个方案，因为它可能成为用户混淆的来源。另一个缺点是，产品初始化方法将必须使用点符号，这有点尴尬的是因为我们只有工厂方法和枚举 cases 才使用点语法。这虽然可以通过将这些产品移到 `Product` 类之外来解决，但是我们认为为产品提供名称空间会提供很大的价值。

* Upgrade `SystemPackageProvider` enum to a struct.
* 将 `SystemPackageProvider` 枚举升级成一个结构体。

    We thought about upgrading `SystemPackageProvider` to a struct when we had
    the "identity" rule but since we're dropping that, there is no need for
    this change.
    
    当我们提出 "identity rule" 时，考虑过将 `SystemPackageProvider` 枚举升级成一个结构体，但由于我们放弃了那个方案，所以这个修改也没有必要了。

    ```swift
    public struct SystemPackageProvider {
        enum PackageManager {
            case apt
            case brew
        }

        /// The system package manager.
        let packageManager: PackageManager

        /// The array of system packages.
        let packages: [String]

        init(_ packageManager: PackageManager, packages: [String])
    }
    ```


