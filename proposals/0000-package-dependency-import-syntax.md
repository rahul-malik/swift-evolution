# Inline package dependency import syntax

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Rahul Malik](https://github.com/rahul-malik), [Ankit Aggarwal](https://github.com/aciidb0mb3r), [David Hart](https://github.com/hartbit)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Swift is a general-purpose language that aims expand it's availability and impact on various domains and platforms. We believe great scripting support and experience is an important part of improving the impact of Swift as a language. Swift already includes basic support for scripting via the Swift command-line tool. This is a proposal for greatly improving the script support by providing a deeper integration with the Swift Package Manager.

Swift-evolution thread: [Discussion thread topic for that
proposal](https://forums.swift.org/)

## Motivation

The Swift standard library maintains a "batteries-not-included" approach and relies on external packages to provide more functionality. However, this introduces friction in writing Swift scripts as there's no easy way to reference external packages from a single file. For example, there's no way to easily include basic scripting utilities like a command-line argument parsing library.

The Swift community has also recognized and tried to provide better scripting support through projects like [Marathon](http://github.com/johnsundell/marathon) and [swift-sh](http://github.com/mxcl/swift-sh). These projects leverage the Swift Package Manager to fetch and build external packages referenced in a Swift script. We believe we can take this approach further and provide a much better scripting experience by improving the integration between the Swift language and the Swift Package Manager. This will also allow Swift users to easily share their scripts without requiring to install additional tooling.

This proposal would dramatically improve the developer experience for writing scripts in Swift. Dependencies for common use-cases would be easy to integrate and scripts would be portable across machines without additional tools installed on the host machine. 

## Proposed solution

We propose to add a new attribute @package to the import statement for referencing Swift packages. This attribute would be only valid when running in the Swift interpreter (via `swift MyScript.swift` or `swift run MyScript.swift`) and will be rejected if used during compilation of a regular Swift module. Swift Package Manager will then fetch and build the declared packages and make them available to the script when executing it.

```
import Foundation

@package(url: "https://github.com/jpsim/Yams.git", from: "2.0.0")
import Yams
```

This script can be executable by either `swift MyScript.swift` or `swift run MyScript.swift` which will utilize Swift package managers dependency fetching and resolution logic.

The `@package(...)`syntax will allow the same parameters as SwiftPM's [Package Dependency Description](https://docs.swift.org/package-manager/PackageDescription/PackageDescription.html#package-dependency) which should feel natural to developers familiar with Swift Package Manager manifest syntax.  

## Detailed design


#### Basic usage
The most common case is expected to be importing a dependency where the product name is equivalent to the imported module name.
```swift
import Foundation
@package(url: "http://github.com/author/example.git") import Example
```

#### Dependency version requirements

The `@package(..)` annotation would also allow a developer to specify requirements to ensure your scripts are reproducible. The syntax will contain the same level of control and specificity as `PackageDependencyDescription.Requirement` which should feel like a natural to developers familiar with Swift Package Manager manifest syntax.  

*Revision*
```swift
@package(url: "http://github.com/author/example.git", .revision( "065675b3d1364a6f63b94a9c89be2e9ed0a4c3a1")) 
import Example
```

*Branch*
```swift
@package(url: "http://github.com/author/example.git", .branch("releases/1.0")) 
import Example
```

*Exact Version*
```swift
@package(url: "http://github.com/author/example.git", .exact("1.0")) 
import Example
```

*Range of version*
```swift
@package(url: "http://github.com/author/example.git", .from("1.0")) 
import Example
```

*Local dependency*
Absolute path
```swift
@package(path: "/Users/author/example") 
import Example
```

Relative path
```swift
@package(path: "my_project/internal_package") 
import Example
```

#### Transitive dependencies 

Package dependencies automatically make their transitive package dependencies available for use.

For an example, lets look at the Package description below for our example dependency and transitive dependency.

Direct dependency
```swift
let package = Package(
    name: "Example",
    products: [
        .library(name: "example", targets: ["Example"]),
    ],
    dependencies: [
        .package(url: "https://github.com/another_author/TransitiveDependency.git", from: "1.0.0"),
    ],
)
```

Transitive dependency
```swift
let package = Package(
    name: "TransitiveDependency",
    products: [
        .library(name: "TransitiveDependency", targets: ["TransitiveDependency"]),
    ],
    dependencies: [],
)
```

With the above Package descriptions, we can import transitive dependencies by explicitly exporting them in the package attribute definition.
```swift
import Foundation
@package(url: "http://github.com/author/example", products: ["TransitiveDependency"]) import Example 
import TransitiveDependency 
```

### Integration with Swift Compiler 

The `@package()` syntax will need to be added to the AST definition and compiler front-end will require changes to parse this syntax.

### Integration with SwiftPM

#### Execution
Scripts that utilize the `@package()` attribute will be executed by `swift run` and managed by Swift package manager.

Swift package manager will utilize the changes to the compiler front-end to access packages referenced using the `@package` attribute.

#### Package resolution 
Once package dependencies are known, SwiftPM will use it's existing logic for resolving those dependencies and outputting a ".resolved" file to ensure future invocations are reproducible.

The name of the resolved file will be `script_name.resolved`.

### Integration with SourceKit-LSP 

TBD


## Impact on exisiting packages

This change will not affect existing packages


## Alternatives considered

### Global package dependencies

One alternative could be adding a feature to globally install packages that can be referenced by any script. This is usually a bad idea because it negatively affects the portability of our scripts.
