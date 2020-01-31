# SwiftPM support for Swift scripts

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

We propose to add a new attribute `@package` to import statements for referencing Swift packages. This attribute would be only valid when running in the Swift interpreter (via `swift MyScript.swift` or `swift run MyScript.swift`) and will be rejected if used during compilation of a regular Swift module. Swift Package Manager will then fetch and build the declared packages and make them available to the script when executing it.

```swift
@package(url: "https://github.com/jpsim/Yams.git", from: "2.0.0")
import Yams
```

This script can be executable by either `swift MyScript.swift` or `swift run MyScript.swift` which will utilize SwiftPM's dependency fetching and resolution logic.

The `@package(...)`syntax will allow the same parameters as SwiftPM's [Package Dependency Description](https://docs.swift.org/package-manager/PackageDescription/PackageDescription.html#package-dependency) which should feel natural to developers familiar with Swift Package Manager manifest syntax.  

## Detailed design


#### Basic usage
The most common case is expected to be importing a dependency where the product name is equivalent to the imported module name.

Specifying the version of the package is optional. If omitted, SwiftPM will resolve the latest version of the dependency.
```swift
@package(url: "https://github.com/jpsim/Yams.git")
import Yams
```

When a dependency product and target names do not match, the product must be explicitly specified.

For an example, lets look at the Package description below for `swift-tools-support-core`. 
```swift
let package = Package(
    name: "swift-tools-support-core",
    products: [
        .library(
            name: "SwiftToolsSupport",
            type: .dynamic,
            targets: ["TSCBasic", "TSCUtility"]),
    ],
    // Rest of Package.swift truncated for brevity .....
}
```

With the above package description, the `SwiftToolsSupport` product does not match a target within the package description so it must be specified explicitly.

```swift
@package(url: "https://github.com/apple/swift-tools-support-core.git", .exact("0.0.1"), products: ["SwiftToolsSupport"])
import TSCBasic 

```

#### Dependency version requirements

The `@package(..)` annotation would also allow a developer to specify requirements to ensure your scripts are reproducible. The syntax will contain the same level of control and specificity as `PackageDependencyDescription.Requirement` which should feel like a natural to developers familiar with Swift Package Manager manifest syntax.  

*Revision*
```swift
@package(url: "https://github.com/jpsim/Yams.git", .revision("5fa313eae1ca127ad3c706e14c564399989cb1b1")) 
import Yams 
```

*Branch*
```swift
@package(url: "https://github.com/jpsim/Yams.git", .branch("releases/1.0")) 
import Yams 
```

*Exact Version*
```swift
@package(url: "https://github.com/jpsim/Yams.git", .exact("2.0.0")) 
import Yams 
```

*Range of version*
```swift
@package(url: "https://github.com/jpsim/Yams.git", from: "1.0.0") 
import Yams 
```

*Local dependency*
Absolute path
```swift
@package(path: "/Users/username/Yams") 
import Yams 
```

Relative path
```swift
@package(path: "dependencies/Yams") 
import Yams 
```

### Integration with Swift Compiler 

The `@package()` syntax will need to be added to the AST definition and compiler front-end will require changes to parse this syntax.

### Integration with SwiftPM

#### Execution
Scripts that utilize the `@package()` attribute can be executed by `swift run` and managed by SwiftPM.

SwiftPM will utilize the changes to the compiler front-end to access packages referenced using the `@package` attribute.

#### Package resolution 

Once package dependencies are known, SwiftPM will use it's existing logic for resolving those dependencies and outputting a ".resolved" file to ensure future invocations are reproducible.

The name of the resolved file will be `script_name.resolved`. By default, the resolve file will be located under `~/.swiftpm/scripts` but can be specified to another path if desired.

#### Updating package dependencies

Dependencies of scripts can be updated to their latest version by executing `swift package update <script-name>`. 


#### Converting a script to a Package

SwiftPM will have new support for transforming a script to a package by extracting the information specified in `@package(...)` declarations.  

#### Built products

Products built from scripts will be located in common per-user location `~/.swiftpm/scripts/...`

#### Shared local dependency cache

Dependencies will be stored and managed by a local dependency cache which will allow scripts to reuse any dependencies that have already been fetched. 


## Alternatives considered

### Global package dependencies

One alternative could be adding a feature to globally install packages that can be referenced by any script. This is usually a bad idea because it negatively affects the portability of our scripts.
