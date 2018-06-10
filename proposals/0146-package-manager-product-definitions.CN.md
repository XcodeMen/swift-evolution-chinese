# 0146 - Package Manager Product Definitions

* Proposal: [SE-0146](0146-package-manager-product-definitions.md)
* Author: [Anders Bertelrud](https://github.com/abertelrud)
* Review manager: Daniel Dunbar
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution-announce/2016-November/000298.html)
* Bug: [SR-3606](https://bugs.swift.org/browse/SR-3606)
* 翻译: [KANGZUBIN](https://github.com/kangzubin)
* 校对: 

## Introduction 介绍

This proposal introduces the concept of *products* to the Swift Package Manager, and proposes enhancements to the `Package.swift` syntax to let packages define products that can be referenced by other packages.

本提案在 Swift 包管理器中引入 *产品* 的概念，以及建议对 `Package.swift` 的语法进行增强，让包定义可让其他包引用的产品。

## Motivation 动机

Currently, the Swift Package Manager has only a limited notion of the products that are built for a package.  It does have a set of rules by which it infers implicit products based on the contents of the targets in the package, and it also has a small amount of undocumented, unsupported package manifest syntax for explicitly declaring products, but it provides no supported way for package authors to declare what their packages produce.

目前，Swift 包管理器对为包所构建的产品只有有限的概念。它确实有一套规则，通过它可以根据包中 targets 的内容来推断隐式的产品，此外它还有少量的未记录的，不支持的包清单语法，用于显式声明产品，但是它没有提供支持包的作者声明他们的包能产生什么的方式。

Also, while the Swift Package Manager currently supports dependencies between packages, it has no support for declaring a dependency on anything more fine-grained than a package.

此外，虽然包管理器目前支持包之间的依赖关系，但它不支持给任何比包更细粒度的声明依赖。

Such fine-grained dependencies are often desired, and this desire has in the past been expressed as a request for the ability to declare a dependency on an arbitrary target in a package.  That, in turn, leads to requests to provide access control for targets, since a package author may want control over which targets can be independently accessed from outside the package.

这种细粒度的依赖通常是需要的，且这种需求在过去被表示为一种能够声明包中任意 target 的依赖关系的请求。这反过来又导致为 target 提供访问控制的请求，因为包的作者可能希望控制从包外部可以独立访问哪些 targets。

Even if visibility control for targets were to be provided (indeed, an early draft of the package manifest syntax had the notion of "publishing" a target to external clients), there would still be no way for the package author to declare anything about the kind of product that should be built.

即使要提供 targets 的可见性控制（事实上，包清单语法的早期草案也有“发布”的一个 target 到外部客户端的概念），包的作者仍然无法声明任何应该构建的产品类型。

One consequence of the lack of ability to define dependencies at subpackage granularity is that package authors have to break up packages into several smaller packages in order to achieve the layering they want.  Such package decomposition may be appropriate in some cases, but should be done based on what makes sense from a conceptual standpoint and not because the Swift Package Manager doesn't allow the author to express their intent.

缺少以子包（subpackage）的粒度来定义依赖关系的一个后果是，包的作者必须将包拆分成几个较小的包，以实现他们想要的分层。这种包的拆分在某些情况下可能是适当的，但应该从概念的角度来看这样做有何意义，而不是因为 Swift 包管理器不允许作者表明他们的意图。

For example, consider the package for a component that has both a library API and command line tools.  Such a package is also likely to be partitioned into a set of core libraries on which both the public library and the command line tools depend, but which should remain a private implementation detail as far as clients are concerned.

例如，考虑一个同时包含库 API 和命令行工具的组件的包。这样的包同样可能被划分成一组同时被公共库和命令行工具依赖的核心库，但就客户端而言，它应该保留私有的实现细节。

Such a package would currently need to be split up into three separate packages in order to provide the appropriate dependency granularity:  one for the public library, another for the command line tools, and a third, private package to provide the shared implementation used by the other two packages.  In the case of a single conceptual component that should have a single version number, this fracturing into multiple packages is directly contrary to the developer's preferred manner of packaging.

目前需要将这样的一个包拆分成三个独立的包，以提供适当的依赖关系粒度：一个用于公共库，另一个用于命令行工具，以及第三个私有包，用于提供其他两个包使用的共用实现。在单个概念组件应该具有单个版本号的情况下，这种分裂成多个包直接与开发人员的首选打包方式相违背。

What is needed is a way to allow package authors to define conceptually distinct products of a package, and to allow client packages to declare dependencies on individual products in a package.

我们需要的是一个允许包的作者从概念上定义一个包的不同产品的方法，并允许客户端程序包声明一包中的各个产品的依赖关系。

Furthermore, explicit product definitions would allow the package author to control the types of artifacts produced from the targets in the package.  This would include such things as whether a library is built as a static archive or a dynamic library.  We expect that additional product types will be added over time to let package authors build more kinds of artifacts.

此外，显式的产品定义将允许包的作者控制包中的 targets 产生的组件类型。这将包括诸如一个库是被编译为静态归档还是动态库。我们期望随着时间的推移会添加更多的产品类型，以便让包的作者编译更多类型的组件。

## Proposed solution 建议的解决方案

We will introduce a documented and supported concept of a package product, along with package manifest improvements to let package authors define products and to let package clients define dependencies on such products.  In defining a product, a package author will be able specify the type of product as well as its characteristics.

我们将引入一个文档化的和可支持的包产品（package product）的概念，以及包清单的改进，让包的作者定义产品，并让包的客户端定义对这些产品的依赖。在定义一个产品时，包的作者将可以指定产品的类型及其特性。

### Product definitions 产品定义

A package will be able to define an arbitrary number of products that are visible to all direct clients of the package.  A product definition consists of a product type, a name (which must be unique among the products in the package), and the root targets that comprise the implementation of the product.  There may also be additional properties depending on the type of product (for example, a library may be static or dynamic).

一个包将能够定义任意数量的产品，这些产品对包的所有直系客户端可见。产品定义由产品类型、名称（在包中的产品的名称必须是唯一的）以及构成产品实现的根 targets 组成。另外根据产品的类型，可能还会有其他属性（例如，一个库可能是静态库或动态库）。

Any target may be included in multiple products, though not all kinds of targets are usable in every kind of product; for example, a test target is not able to be included in a library product.

任何 target 都可以被包含在多个产品中，但并不是每种类型的产品都可以使用所有类型的 targets，例如，测试 target 就不能被包含在一个库产品中。 

The products represent the publicly vended package outputs on which client packages can depend.  Other artifacts might also be created as part of building the package, but what the products specifically define are the conceptual "outputs" of the package, i.e. those that make sense to think of as something a client package can depend on and use.  Examples of artifacts that are not necessarily products include built unit tests and helper tools that are used only during the build of the package.

产品代表客户端软件包可以依赖的公开发布的包输出。其他组件也可以作为包构建过程的一部分来创建，但产品所特别定义的是包的概念“输出”，即那些有意义地认为客户端软件包可以依赖和使用的东西。组件的示例是不需要产品引入构建单元测试以及只在构建包过程使用的辅助工具。

An example of a package that defines two library products and one executable product:

一个包定义两个库产品和一个可执行产品的示例：

```swift
let package = Package(
    name: "MyServer",
    targets: [
        Target(name: "Utils"),
        Target(name: "HTTP", dependencies: ["Utils"]),
        Target(name: "ClientAPI", dependencies: ["HTTP", "Utils"]),
        Target(name: "ServerAPI", dependencies: ["HTTP"]),
        Target(name: "ServerDaemon", dependencies: ["ServerAPI"]),
    ],
    products: [
        .Library(name: "ClientLib", type: .static, targets: ["ClientAPI"]),
        .Library(name: "ServerLib", type: .dynamic, targets: ["ServerAPI"]),
        .Executable(name: "myserver", targets: ["ServerDaemon"]),
    ]
)
```

The initial types of products that can be defined are executables and libraries.  Libraries can be declared as either static or dynamic, but when possible, the specific type of library should be left unspecified, letting the build system chooses a suitable default for the platform.

产品的初始类型可定义的有可执行文件和库。库可以被声明为静态的或动态的，但如果可以的话，库的指定类型应该保持未定义，让编译系统根据平台选择一个合适的默认值。

Note that tests are not considered to be products, and do not need to be explicitly defined.

请注意，测试不被视为产品，且不需要明确定义。

A product definition lists the root targets to include in the product; for product types that vend interfaces (e.g. libraries), the root targets are those whose modules will be available to clients.  Any dependencies of those targets will also be included in the product, but won't be made visible to clients.  The Swift compiler does not currently provide this granularity of module visibility control, but the set of root targets still constitutes a declaration of intent that can be used by IDEs and other tools.  We also hope that the compiler will one day support this level of visibility control.  See [SR-3205](https://bugs.swift.org/browse/SR-3205) for more details.

产品定义列出了要包含在产品中的根 targets；对于公布接口的产品类型（例如：库），根 targets 就是模块将可供客户端使用的那些 targets。这些 targets 的任何依赖项也将被包含在产品中，但对客户端不可见。Swift 编译器目前不提供这种粒度的模块可见性控制，但是根 targets 集合仍然构成 IDE 和其他工具可以使用的意向声明。我们也希望编译器有一天能支持这个级别的可见性控制。详见 [SR-3205](https://bugs.swift.org/browse/SR-3205)。

For example, in the package definition shown above, the library product `ClientLib` would only vend the interface of the `ClientAPI` module to clients.  Since `ClientAPI` depends on `HTTP` and `Utilities`, those two targets would also be compiled and linked into `ClientLib`, but their interfaces should not be visible to clients of `ClientLib`.

例如，上述显示的包定义中，库产品 `ClientLib` 只会将 `ClientAPI` 模块的接口公布给客户端。由于 `ClientAPI` 依赖于 `HTTP` 和 `Utilities`，因此这两个 targets 也会被编译并链接到 `ClientLib` 中，但它们的接口对 `ClientLib` 的客户端不可见。

### Implicit products 隐式产品

SwiftPM 3 applies a set of rules to infer products based on the targets in the package.  For backward compatibility, SwiftPM 4 will apply the same rules to packages that use the SwiftPM 3 PackageDescription API.  The `package describe` command will show implied product definitions.

SwiftPM 3 应用了一组规则根据包中的 targets 来推断产品。为了向后兼容，SwiftPM 4 将对使用SwiftPM 3 PackageDescription API 的包应用相同的规则。`package describe` 命令将显示隐含的产品定义。

When switching to the SwiftPM 4 PackageDescription API, the package author takes over responsibility for defining the products.  There will be tool support (probably in the form of a fix-it on a "package defines no products" warning) to make it easy for the author to add such definitions.  Also, the `package init` command will be extended to automatically add the appropriate product definitions to the manifest when it creates the package.

当切换到 SwiftPM 4 PackageDescription API 时，包的作者接管了定义产品的职责。这里将会有工具支持（可能是修复 "package defines no products" 警告的形式），是作者可以很容易添加这样的定义。同时，`package init` 命令将被扩展，以在创建包时自动将相应的产品定义添加到清单。

There was significant discussion about whether the implicit product rules should continue to be supported alongside the explicit product declarations.  The tradeoffs are described in the "Alternatives considered" section.

关于隐式产品规则是否应该继续被显式产品声明支持的问题进行了重要的讨论。它们的权衡在“考虑过的替代方案”部分进行描述。

### Product dependencies 产品依赖关系

A target will be able to declare its use of the products that are defined in any of the external package dependencies.  This is in addition to the existing ability to declare dependencies on other targets in the same package.

一个 target 将能够声明使用在任何外部包依赖关系中定义的产品。这是除了现有能力之外的，在同一包中声明对其他 targets 的依赖关系。

To support this, the `dependencies` parameter of the `Target()` initializer will be extended to also allow product references.

为了支持这个，`Target()` 初始化方法中的 `dependencies` 参数被扩展为也允许产品引用。

To see how this works, remember that each string listed for the `dependencies` parameter is just shorthand for `.target(name: "...")`, i.e. a dependency on a target within the same package.

为了看清它是如何工作的，请记住，为 `dependencies` 参数列出的每个字符串只是 `.target（name：“...”）` 的简写，即为同一个包中的 target 的依赖关系。
    
For example, the target definition:

例如，target 定义：
    
```swift
Target(name: "Foo", dependencies: ["Bar", "Baz"])
```
    
is shorthand for:

是以下的简写：
    
```swift
Target(name: "Foo", dependencies: [.target(name: "Bar"), .target(name: "Baz")])
```

This will be extended to support product dependencies in addition to target dependencies:

除了 target 依赖关系外，还将被扩展支持产品依赖关系：

```swift
let package = Package(
    name: "MyClientLib",
    dependencies: [
        .package(url: "https://github.com/Alamofire/Alamofire", majorVersion: 3),
    ],
    targets: [
        Target(name: "MyUtils"),
        Target(name: "MyClientLib", dependencies: [
            .target(name: "MyUtils"),
            .product(name: "Alamofire", package: "Alamofire")
        ])
    ]
)
```

The package name is the canonical name of the package, as defined in the manifest of the package that defines the product.  The product name is the name specified in the product definition of that same manifest.

包名称是包的规范名称，如定义产品的包清单中所定义的。产品名称是在同一个清单的产品定义中指定的名称。

The package name is optional, since the product name is almost always unambiguous (and is frequently the same as the package name).  The package name must be specified if there is more than one product with the same name in the package graph (this does not currently work from a technical perspective, since Swift module names must currently be unique within the package graph).

包名称是可选的，因为产品名称几乎总是明确的（并且通常与包名称相同）。如果 package graph 中有多于一个具有相同名字的产品，这必须指定该包的名称（当前从技术角度看这并不适用，因为 Swift 模块名称目前在 package graph 中必须是唯一的）。

In order to continue supporting the convenience of being able to use plain strings as shorthand, and in light of the fact that most of the time the names of packages and products are unique enough to avoid confusion, we will extend the short-hand notation so that a string can refer to either a target or a product.

为了继续支持使用简单的字符串作为简写的便利性，以及考虑到大多数情况下，包和产品的名称是足够唯一的，能够避免混淆，我们将扩展这样的简写符号，以便一个字符串可以指向 target 或产品。

The Package Manager will first try to resolve the name to a target in the same package; if there isn't one, it will instead to resolve it to a product in one of the packages specified in the `dependencies` parameter of the `Package()` initializer.

包管理器将首先尝试将该名称解析到同一个包中的 target；如果不存在，则将其解析为 `Package()` 初始化程方法的 `dependencies` 参数中指定的一个包中的产品。

For both the shorthand form and the complete form of product references, the only products that will be visible to the package are those in packages that are declared as direct dependencies -- the products of indirect dependencies are not visible to the package.

对于产品引用的简写形式和完整形式，包中唯一可见的产品是那些被声明为直接依赖包的产品 -- 而间接依赖关系的产品对于包是不可见的。

## Detailed design 详细设计

### Product definitions 产品定义

We will add a `products` parameter to the `Package()` initializer:

我们将在 `Package()` 初始化方法中添加一个 `products` 参数：

```swift
Package(
    name: String,
    pkgConfig: String? = nil,
    providers: [SystemPackageProvider]? = nil,
    targets: [Target] = [],
    dependencies: [Package.Dependency] = [],
    products: [Product] = [],
    exclude: [String] = []
)
```
    
The definition of the `Product` type will be:

`Product` 类型的定义为：
   
```swift
public enum Product {
    public class Executable {
        public init(name: String, targets: [String])
    }
    
    public class Library {
        public enum LibType {
            case .static
            case .dynamic
        }
        public init(name: String, type: LibType? = nil, targets: [String])
    }
}
```

The namespacing allows the names of the product types to be kept short.

该命名空间允许产品类型的名称保持简短。

As with targets, there is no semantic significance to the order of the products in the array.

与 targets 一样，对于数组中的产品的顺序没有语义意义。

### Implicit products 隐式产品

If the `products` array is omitted, and if the package uses version 3 of the PackageDescription API, the Swift Package Manager will infer a set of implicit products based on the targets in the package.

如果 `products` 数组被省略了，并且包使用了版本 3 的 PackageDescription API，则 Swift 包管理器将根据包中的 targets 推断一组隐式产品。

The rules for implicit products are the same as in SwiftPM 3:

隐式产品的规则与 SwiftPM 3 一致：

1. An executable product is implicitly defined for any target that produces an executable module (as defined here: https://github.com/apple/swift-package-manager/blob/master/Documentation/Reference.md).

   对于生成可执行模块的任何 target，都可以隐式定义一个可执行产品（定义如下：https://github.com/apple/swift-package-manager/blob/master/Documentation/Reference.md）。

2. If there are any library targets, a single library product is implicitly defined for all the library targets.  The name of this product is based on the name of the package.

   如果有任何库 targets，则为所有库 targets 隐式定义一个库产品。该产品的名称是基于包的名称。

### Product dependencies 产品依赖关系

We will add a new enum case to the `TargetDependency` enum to represent dependencies on products, and another enum case to represent an unbound by-name dependency.  The string-literal conversion will create a by-name dependency instead of a target dependency, and there will be logic to bind the by-name dependencies to either target or product dependencies once the package graph has been resolved:

我们将在 `TargetDependency` 枚举中添加一个新的枚举类型来表示对产品的依赖关系，和另一个枚举类型来表示一个未绑定的名称依赖关系。string-literal 转换将创建一个名称依赖关系而不是 target 依赖关系，并且一旦 package graph 发生转变，将会有逻辑把名称依赖关系绑定到 target 或产品依赖关系：

```swift
public final class Target {

    /// Represents a target's dependency on another entity.
    /// 表示 target 对另一个实体的依赖。
    public enum TargetDependency {
        /// A dependency on a target in the same project.
        /// 同一工程中的 tareget 的依赖。
        case Target(name: String)
        
        /// A dependency on a product from a package dependency.  The package name match the name of one of the packages named in a `.package()` directive.
        /// 包依赖中的产品的依赖。包名称与 `.package()` 中指定的包名称相匹配。
        case Product(name: String, package: String?)
        
        /// A by-name dependency that resolves to either a target or a product, as above, after the package graph has been loaded.
        /// 如上所述，在加载 package graph 之后解析为 target 或 product 的名称依赖。
        case ByName(name: String)
    }
    
    /// The name of the target.
    /// target 的名称。
    public let name: String
    
    /// Dependencies on other entities inside or outside the package.
    /// 包内部或外部的其他实体的依赖关系。
    public var dependencies: [TargetDependency]
    
    /// Construct a target.
    /// 构建一个 target。
    public init(name: String, dependencies: [TargetDependency] = [])
}
```

For compatibility reasons, packages using the Swift 3 version of the PackageDescription API will have implicit dependencies on the directly specified packages.  Packages that have adopted the Swift 4 version need to declare their product dependencies explicitly.

出于兼容性的原因，对于使用 Swift 3 版本的 PackageDescription API 的包将对直接指定的包有隐式的依赖关系。而采用 Swift 4 版本 API 的包需要明确声明其产品的依赖关系。

## Impact on existing code 对现有代码的影响

There will be no impact on existing packages that follow the documented Swift Package Manager 3.0 format of the package manifest.  Until the package is upgraded to the 4.0 format, the Swift Package Manager will continue to infer products based the existing rules.

对于遵循 Swift Package Manager 3.0 格式的包清单的现有包不会有任何影响。在包升级到 4.0 格式之前，Swift Package Manager 将继续根据现有的规则推断产品。

## Alternatives considered 考虑过的替代方案

Many alternatives were considered during the development of this proposal, and in many cases the choice was between two distinct approaches, each with clear advantages and disadvantages.  This section summarizes the major alternate approaches.

在提出这一提案的过程中，考虑了许多替代方案，在很多情况下是在两种截然不同的方法之间选择，每种方法都有明显的优点和缺点。本节总结一下主要的替代方法。

### Not adding product definitions
### 不添加产品定义

Instead of product definitions, fine-grained dependencies could be introduced by allowing targets to be marked as public or private to the package.

除了产品定义，可以通过允许将包中的 targets 被标记为公开或私有的来引入细粒度的依赖关系。

_Advantage_
_优点_

It would avoid the need to introduce new concepts.

这将避免引入新的概念。

_Disadvantage_
_缺点_

It would not provide any way for a package author to control the type and characteristics of the various artifacts.  Relying on the implied products that result from the inference rules is not likely to be scalable in the long term as we introduce new kinds of product types.

它不会提供任何方式让包的作者控制各种组件的类型和特性。在引入新的产品类型时，依赖于推断规则产生的隐式产品在长期内很难可扩展。

### Inferring implicit products
### 推断隐式产品

An obvious alternative to the proposal would be to keep the inference rules even in v4 of the PackageDescription API.

本提案的另外一个明显的替代方案是，在 PackageDescription API 的 v4 版本仍然保持推断规则。

_Advantage_
_优点_

It can be a lot more convenient to not have to declare products, and to be able to add new products just by adding or renaming files and directories.  This is particularly true for small, simple packages.

它可以带来更大的方便，不必声明产品，并且可以只通过添加或重命名文件和目录来添加新产品。对于小而简单的包来说尤其如此。

_Disadvantages_
_缺点_

The very fact that it is so easy to change the set of products without modifying the package manifest can lead to unexpected behavior due to seemingly unrelated changes (e.g. creating a file called `main.swift` in a module changes that module from a library to an executable).

事实上，由于在不修改包清单的情况下更改产品集是很容易的，因此一些看似不相关的更改会导致预料之外的行为（例如，在模块中创建一个名为 `main.swift` 的文件会将该模块从库更改为可执行文件）。

Also, as packages become more complex and new conceptual parts on which clients can depend are introduced, the interaction of implicit rules with the explicit product definitions can become very complicated.

另外，随着包变得越来越复杂，以及客户端可以依赖的新概念部分被引入，隐式规则与显示产品定义的相互作用可能变得非常复杂。

We plan to provide some of the convenience through tooling.  For example, an IDE (or `swift package` itself on the command line) can offer to add product definitions when it notices certain types of changes to the structure of the package.  We believe that having defined product types lets the tools present a lot better diagnostics and other forms of help to users, since it provides a clear statement of intent on the part of the package author.

我们计划通过工具来提供一些便利。比如，当 IDE（或者命令行中的 `swift package` 本身）在注意到包结构的某些类型发生变化时，它可以提供添加产品定义。我们相信，通过定义产品类型，可以让工具为用户提供更好的诊断和其他形式的帮助，因为它为包作者提供了一个清晰的意图声明。

### Distinguishing between target dependencies and product dependencies
### 区分 target 的依赖关系和产品的依赖关系

The proposal extends the `Target()` initializer's `dependencies` parameter to allow products as well as targets; another approach would have been to add a new parameter, e.g. `externalDependencies` or `productDependencies`.

在本提案中扩展了 `Target()` 初始化方法中的 `dependencies` 参数以支持产品和 targets；另一种方法是增加一个新的参数，例如 `externalDependencies` 或 `productDependencies`。

_Advantage_
_优点_

Targets and products are different types of entities, and it is possible for a target and a product to have the same name.  Having the `dependencies` parameter as a heterogeneous list can lead to ambiguity.

Targets 和产品是不同类型的实体，并且 target 和产品可能具有相同的名称。将 `dependencies` 参数作为多种类型的列表可能会带来混淆。
   
_Disadvantages_
_缺点_

Conceptually the *dependency* itself is the same concept, even if type of entity being depended on is technically different.  In both cases (target and product), it is up to the build system to determine the exact type of artifacts that should be produced.  In most cases, there is no actual semantic ambiguity, since the name of a target, product, and package are often the same uniquely identifiable "brand" name of the component.

从概念上讲，*依赖* 本身是相同的概念，即使被依赖的实体类型技术上是不同的。在这两种情况下（target 和产品），都要由编译系统来确定组件所应该生成的确切类型。在大多数情况下，不存在实际的语义歧义，因为一个 target，product 和 package 的名称通常是组件唯一可识别的 "brand" 名称。

Also, separating out each type of dependency into individual homogeneous lists doesn't scale.  If a third type of dependency needs to be introduced, a third parameter would also need to be introduced.  Keeping the list heterogeneous avoids this.

此外，将每种依赖关系的类型分解成单独的同类列表并不能完全解决问题。如果需要引入第三种类型的依赖关系，则还需要引入第三个参数。而保持多种类型的列表可以避免这一点。


