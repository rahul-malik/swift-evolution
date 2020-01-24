# Inline package dependency import syntax

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Rahul Malik](https://github.com/rahul-malik), [Ankit Aggarwal](https://github.com/aciidb0mb3r), [David Hart](https://github.com/hartbit)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

Developers today are using Swift as a scripting language to utilize its language features and portability across macOS and Linux. Compared to other common scripting languages (Python, Ruby, etc.), Swift maintains a "batteries-not-included" approach and relies on external packages to provide functionality like command-line argument parsing that are common in scripts. This proposal focuses on a new syntax for importing package dependencies directly within scripts written in Swift with the same level of specificity offered by the current Package manifest schema.

Swift-evolution thread: [Discussion thread topic for that
proposal](https://forums.swift.org/)

## Motivation

Writing scripts in Swift currently requires more work than other languages. The community has filled many of the gaps in terms of functionality like command-line argument parsing but even utilizing these packages can add significantly more friction.

The community has also tried to solve this problem themselves through projects like [Marathon](http://github.com/johnsundell/marathon) and [swift-sh](http://github.com/mxcl/swift-sh) to provide functionality like resolving and fetching dependencies which are available within SwiftPM today.
There are drawbacks to this solution
- Additional tooling is required for any host machine that runs a Swift script
- Package resolution and fetching is performed outside of SwiftPM and can't leverage it's capabilities

This proposal would dramatically improve the developer experience for writing scripts in Swift. Dependencies for common use-cases would be easy to integrate and scripts would be portable across machines without additional tools installed on the host machine. 

## Proposed solution


Improving dependency management and portabilty of scripts requires declaring dependencies within the script itself. We would implement a new import syntax for external packages. These external packages can be referenced using the `@package(url: ....)` annotation to specify the package location and version requirements.

```swift
// MyScript.swift
import Foundation
import UIKit
@package(url: "http://github.com/author/example.git") import Example
```

This script can be executable by using `swift run MyScript.swift` which will utilize Swift package managers dependency fetching and resolution logic.

The `@package(...)` syntax will contain the same level of control and specificity as `PackageDependencyDescription.Requirement` which should feel like a natural to developers familiar with Swift Package Manager manifest syntax.  


## Detailed design


#### Basic usage
The most common case is expected to be importing a dependency where the product name is equivalent to the imported module name.
```swift
import Foundation
import UIKit
@package(url: "http://github.com/author/example.git") import Example
```

#### Dependency version requirements

The `@package(..)` annotation would also allow a developer to specify requirements to ensure your scripts are reproducible. The syntax will contain the same level of control and specificity as `PackageDependencyDescription.Requirement` which should feel like a natural to developers familiar with Swift Package Manager manifest syntax.  

*Revision*
```swift
@package(url: "http://github.com/author/example.git", .revision( "065675b3d1364a6f63b94a9c89be2e9ed0a4c3a1")) import Example
```

*Branch*
```swift
@package(url: "http://github.com/author/example.git", .branch("releases/1.0")) import Example
```

*Exact Version*
```swift
@package(url: "http://github.com/author/example.git", .exact("1.0")) import Example
```

*Range of version*
```swift
@package(url: "http://github.com/author/example.git", .from("1.0")) import Example
```

*Local dependency*
Absolute path
```swift
@package(path: "/Users/author/example") import Example
```

Relative path
```swift
@package(path: "my_project/internal_package") import Example
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
import UIKit
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


## Security

TBD

## Impact on exisiting packages

This change will not affect existing packages


## Alternatives considered

TBD
