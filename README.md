# Spyable

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/Matejkob/swift-spyable/Test_SwiftPM.yml?label=CI&logo=GitHub)
[![Codecov](https://img.shields.io/codecov/c/github/Matejkob/swift-spyable?token=YRMM1BDQ85)](https://app.codecov.io/gh/Matejkob/swift-spyable)
[![](https://img.shields.io/endpoint?url=https%3A%2F%2Fswiftpackageindex.com%2Fapi%2Fpackages%2FMatejkob%2Fswift-spyable%2Fbadge%3Ftype%3Dswift-versions)](https://swiftpackageindex.com/Matejkob/swift-spyable)
[![](https://img.shields.io/endpoint?url=https%3A%2F%2Fswiftpackageindex.com%2Fapi%2Fpackages%2FMatejkob%2Fswift-spyable%2Fbadge%3Ftype%3Dplatforms)](https://swiftpackageindex.com/Matejkob/swift-spyable)

A powerful tool for Swift that simplifies and automates the process of creating spies
for testing. Using the `@Spyable` annotation on a protocol, the macro generates
a spy class that implements the same interface as the protocol and keeps track of 
interactions with its methods and properties.

## Overview

A "spy" is a specific type of test double that not only replaces a real component, but also 
records all interactions for later inspection. It's particularly useful in behavior verification,
where the interaction between objects, rather than the state, is the subject of the test.

The Spyable macro is designed to simplify and enhance the usage of spies in Swift testing. 
Traditionally, developers would need to manually create spies for each protocol in  their
codebase — a tedious and error-prone task. The Spyable macro revolutionizes this process
by automatically generating these spies.

When a protocol is annotated with `@Spyable`, the macro generates a corresponding spy class that 
implement this protocol. This spy class is capable of tracking all interactions with its methods
and properties. It records method invocations, their arguments, and returned values, providing 
a comprehensive log of interactions that occurred during the test. This data can then be used 
to make precise assertions about the behavior of the system under test.

**TL;DR**

The Spyable macro provides the following functionality: 
- **Automatic Spy Generation**: No need to manually create spy classes for each protocol.
  Just annotate the protocol with `@Spyable`, and let the macro do the rest.
- **Interaction Tracking**: The generated spy records method calls, arguments, and return
  values, making it easy to verify behavior in your tests.
- **Swift Syntax**: The macro uses Swift syntax, providing a seamless and familiar experience
  for Swift developers.

## Quick start

To get started, import Spyable: `import Spyable`, annotate your protocol with `@Spyable`:

```swift
@Spyable
protocol ServiceProtocol {
    var name: String { get }
    func fetchConfig(arg: UInt8) async throws -> [String: String]
}
```

This will generate a spy class named `ServiceProtocolSpy` that implements `ServiceProtocol`.
The generated class includes properties and methods for tracking the number of method calls,
the arguments passed, and whether the method was called.

```swift
class ServiceProtocolSpy: ServiceProtocol {
    var name: String {
        get { underlyingName }
        set { underlyingName = newValue }
    }
    var underlyingName: (String)!
    
    var fetchConfigArgCallsCount = 0
    var fetchConfigArgCalled: Bool {
        return fetchConfigArgCallsCount > 0
    }
    var fetchConfigArgReceivedArg: UInt8?
    var fetchConfigArgReceivedInvocations: [UInt8] = []
    var fetchConfigArgReturnValue: [String: String]!
    var fetchConfigArgClosure: ((UInt8) async throws -> [String: String])?

    func fetchConfig(arg: UInt8) async throws -> [String: String] {
        fetchConfigArgCallsCount += 1
        fetchConfigArgReceivedArg = (arg)
        fetchConfigArgReceivedInvocations.append((arg))
        if fetchConfigArgClosure != nil {
            return try await fetchConfigArgClosure!(arg)
        } else {
            return fetchConfigArgReturnValue
        }
    }
}
```

Then, in your tests, you can use the spy to verify that your code is interacting
with the `service` dependency of type `ServiceProtocol` correctly:

```swift
func testFetchConfig() async throws {
    let serviceSpy = ServiceProtocolSpy()
    let sut = ViewModel(service: serviceSpy)

    serviceSpy.fetchConfigArgReturnValue = ["key": "value"]

    try await sut.fetchConfig()

    XCTAssertEqual(serviceSpy.fetchConfigArgCallsCount, 1)
    XCTAssertEqual(serviceSpy.fetchConfigArgReceivedInvocations, [1])

    try await sut.saveConfig()

    XCTAssertEqual(serviceSpy.fetchConfigArgCallsCount, 2)
    XCTAssertEqual(serviceSpy.fetchConfigArgReceivedInvocations, [1, 1])
}
```

## Advanced Usage

### Restricting the Availability of Spies

If you wish, you can limit where Spyable's generated code can be used from. This can be useful if you want to prevent spies from being used in user-facing production code.

To enforce this restriction, supply the `behindPreprocessorFlag: String?` parameter of Spyable, like this:

```swift
@Spyable(behindPreprocessorFlag: "DEBUG")
protocol MyService {
    func fetchData() async
}
```
With `behindPreprocessorFlag` specified as DEBUG, the macro expansion will be wrapped in an #if DEBUG preprocessor macro, preventing its use anywhere that the DEBUG flag is not defined:

```swift
#if DEBUG
class MyServiceSpy: MyService {
    var fetchDataCallsCount = 0
    var fetchDataCalled: Bool {
        return fetchDataCallsCount > 0
    }
    var fetchDataClosure: (() async -> Void)?
        func fetchData() async {
        fetchDataCallsCount += 1
        await fetchDataClosure?()
    }
}
#endif
```
This example uses "DEBUG", but you can specify any flags you wish. For example, you could use "TESTS", and then make the macro available to your test targets by adding the "TESTS" flag to them under the "Active Compilation Conditions" custom build flags.

The default value is `SPYABLE`.

#### A Caveat Regarding Xcode Previews
Limiting spy availability may become restrictive if you intend to use spies in your Xcode Previews. If you intend to use spies with previews and you also want to prevent spies from being used in production code, the advised course of action is to split off previews into their own target where you can define a custom flag (Ex: "SPIES_ENABLED"), like this:

```
-- MyFeature (`SPYABLE = 0`)
---- MyFeatureTests (`SPYABLE = 1`)
---- MyFeaturePreviews (`SPYABLE = 1`)
```
Like the example before, you would specify this custom build flag under "Active Compilation Conditions" of both the `MyFeatureTests` and `MyFeaturePreviews` targets.

## Examples

This repo comes with an example of how to use Spyable. You can find it [here](./Examples).

## Documentation

The latest documentation for this library is available [here](https://swiftpackageindex.com/Matejkob/swift-spyable/0.1.2/documentation/spyable).

## Installation

> Warning: Xcode beta 15.x command line tools are required. 

**For Xcode project**

If you are using Xcode beta 15.x command line tools, you can add
 [swift-spyable](https://github.com/Matejkob/swift-spyable) macro to your project as a package.

> `https://github.com/Matejkob/swift-spyable`

**For Swift Package Manager**

In `Package.swift` add:

``` swift
dependencies: [
  .package(url: "https://github.com/Matejkob/swift-spyable", from: "0.1.0")
]
```

and then add the product to any target that needs access to the macro:

```swift
.product(name: "Spyable", package: "swift-spyable"),
```

## License

This library is released under the MIT license. See [LICENSE](LICENSE) for details.
