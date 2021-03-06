# Cuckoo
## Mock your Swift objects!

[![CI Status](http://img.shields.io/travis/Brightify/Cuckoo.svg?style=flat)](https://travis-ci.org/Brightify/Cuckoo)
[![Version](https://img.shields.io/cocoapods/v/Cuckoo.svg?style=flat)](http://cocoapods.org/pods/Cuckoo)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![License](https://img.shields.io/cocoapods/l/Cuckoo.svg?style=flat)](http://cocoapods.org/pods/Cuckoo)
[![Platform](https://img.shields.io/cocoapods/p/Cuckoo.svg?style=flat)](http://cocoapods.org/pods/Cuckoo)
[![Slack Status](http://swiftkit.brightify.org/badge.svg)](http://swiftkit.brightify.org)

## Introduction
Cuckoo was created due to lack of a proper Swift mocking framework. We built the DSL to be very similar to [Mockito](http://mockito.org/), so anyone using it in Java/Android can immediately pick it up and use it.

To have a chat, [join our Slack team](http://swiftkit.brightify.org)!

## How does it work
Cuckoo has two parts. One is the [runtime](https://github.com/Brightify/Cuckoo) and the other one is an OS X command-line tool simply called [CuckooGenerator](https://github.com/SwiftKit/CuckooGenerator).

Unfortunately Swift does not have a proper reflection, so we decided to use a compile-time generator to go through files you specify and generate supporting structs/classes that will be used by the runtime in your test target.

The generated files contain enough information to give you the right amount of power. They work based on inheritance and protocol adoption. This means that only overridable things can be mocked. We currently support all features which fulfill this rule except for things listed in TODO. Due to the complexity of Swift it is not easy to check for all edge cases so if you find some unexpected behavior please file an issue.

## Changelog
List of all changes and new features can be found [here](CHANGELOG.md).

## TODO
We are still missing support for some important features like:  

- [x] inheritance (grandparent methods)
- [x] generics
- [ ] type inference for instance variables (you need to write it explicitly, otherwise it will be replaced with `__UnknownType`)  

## What will not be supported
Due to the limitations mentioned above, none of the things that can't be overridden can't be supported. This includes:
- `struct` - workaround is to use a common protocol
- everything with `final` or `private` modifier
- global constants and functions
- static properties and methods

## Requirements
Cuckoo works on the following platforms:

- **iOS 8+**
- **Mac OSX 10.9+**
- **tvOS 9+**

We plan to add a **watchOS 2+** support soon.

## Cuckoo
### 1. Installation
#### CocoaPods
Cuckoo runtime is available through [CocoaPods](http://cocoapods.org). To install
it, simply add the following line to your test target in your Podfile:

```Ruby
pod "Cuckoo"
```

And add the following `Run script` build phase to your test target's `Build Phases`:

```Bash
# Define output file. Change "${PROJECT_DIR}/${PROJECT_NAME}Tests" to your test's root source folder, if it's not the default name.
OUTPUT_FILE="${PROJECT_DIR}/${PROJECT_NAME}Tests/GeneratedMocks.swift"
echo "Generated Mocks File = ${OUTPUT_FILE}"

# Define input directory. Change "${PROJECT_DIR}/${PROJECT_NAME}" to your project's root source folder, if it's not the default name.
INPUT_DIR="${PROJECT_DIR}/${PROJECT_NAME}"
echo "Mocks Input Directory = ${INPUT_DIR}"

# Generate mock files, include as many input files as you'd like to create mocks for.
"${PODS_ROOT}/Cuckoo/run" generate --testable "${PROJECT_NAME}" \
--output "${OUTPUT_FILE}" \
"${INPUT_DIR}/FileName1.swift" \
"${INPUT_DIR}/FileName2.swift" \
"${INPUT_DIR}/FileName3.swift"
# ... and so forth, the last line should never end with a backslash

# After running once, locate `GeneratedMocks.swift` and drag it into your Xcode test target group.
```

Input files can be also specified directly in `Run script` in `Input Files` form.

Notes: All paths in the Run script must be absolute. Variable `PROJECT_DIR` automatically points to your project directory.  
Keep in mind to include paths to inherited Classes and Protocols for mocking/stubbing parent and grandparents.

#### Carthage
To use Cuckoo with [Carthage](https://github.com/Carthage/Carthage) add in your Cartfile this line:
```
github "Brightify/Cuckoo"
```

Then use the `Run script` from above and replace
```Bash
"${PODS_ROOT}/Cuckoo/run"
```
with
```Bash
"Carthage/Checkouts/Cuckoo/run"
```

Also don't forget to add the Framework into your project's test target.

### 2. Usage
Usage of Cuckoo is similar to [Mockito](http://mockito.org/) and [Hamcrest](http://hamcrest.org/). However, there are some differences and limitations caused by generating the mocks and Swift language itself. List of all the supported features can be found below. You can find complete examples in [tests](Tests).

#### Mock initialization
Mocks can be created with the same constructors as the mocked type. Name of mock class always corresponds to name of the mocked class/protocol with `Mock` prefix (e.g. mock of protocol `Greeter` is called `MockGreeter`).

```Swift
let mock = MockGreeter()
```

#### Spy
Spies are a special case of Mocks where each call is forwarded to the victim by default. From Cuckoo version `0.11.0` we changed the way spies work. When you need a spy, give Cuckoo a class to mock instead of a protocol. You'll then be able to call `enableSuperclassSpy()` (or `withEnabledSuperclassSpy()`) on a mock instance and it will behave like a spy for the parent class.

```Swift
let spy = MockGreeter().withEnabledSuperclassSpy()
```

NOTE: The behavior was changed due to a limitation of Swift. Since we can't create a real proxy for the spy, calls inside the spy were not caught by the Mock and it was confusing. If you rely on the old behavior (i.e. you use spies with final classes), let us know on Slack or in the issues.

#### Stubbing
Stubbing can be done by calling methods as a parameter of the `when` function. The stub call must be done on special stubbing object. You can get a reference to it with the `stub` function. This function takes an instance of the mock that you want to stub and a closure in which you can do the stubbing. The parameter of this closure is the stubbing object.

Note: It is currently possible for the subbing object to escape from the closure. You can still use it to stub calls but it is not recommended in practice as the behavior of this may change in the future.

After calling the `when` function you can specify what to do next with following methods:

```Swift
/// Invokes `implementation` when invoked.
then(_ implementation: IN throws -> OUT)

/// Returns `output` when invoked.
thenReturn(_ output: OUT, _ outputs: OUT...)

/// Throws `error` when invoked.
thenThrow(_ error: ErrorType, _ errors: Error...)

/// Invokes real implementation when invoked.
thenCallRealImplementation()

/// Does nothing when invoked.
thenDoNothing()
```

Which methods can be used depends on the stubbed method. For example you cannot use the `thenThrow` method with a method that isn't throwing or rethrowing.

The stubbing of a method can look like this:

```Swift
stub(mock) { stub in
  when(stub.greetWithMessage("Hello world")).then { message in
      print(message)
  }
}
```

As for a property:

```Swift
stub(mock) { stub in
  when(stub.readWriteProperty.get).thenReturn(10)
  when(stub.readWriteProperty.set(anyInt())).then {
      print($0)
  }
}
```

Notice the `get` and `set`, these will be used in verification later.

##### Enabling default implementation
In addition to stubbing, you can enable default implementation using an instance of the original class that's being mocked. Every method/property that is not stubbed will behave according to the original implementation.

Enabling the default implementation is achieved by simply calling the provided method:

```Swift
let original = OriginalClass<Int>(value: 12)
mock.enableDefaultImplementation(original)
```

For passing classes into the method, nothing changes whether you're mocking a class or a protocol. However, there is a difference if you're using a `struct` to conform to the original protocol we are mocking:

```Swift
let original = ConformingStruct<String>(value: "Hello, Cuckoo!")
mock.enableDefaultImplementation(original)
// or if you need to track changes:
mock.enableDefaultImplementation(mutating: &original)
```

Note that this only concerns `struct`s. `enableDefaultImplementation(_:)` and `enableDefaultImplementation(mutating:)` are different in state tracking.

The standard non-mutating method `enableDefaultImplementation(_:)` creates a copy of the `struct` for default implementation and works with that. However, the mutating method `enableDefaultImplementation(mutating:)` takes a reference to the struct and the changes of the `original` are reflected in the default implementation calls even after enabling default implementation.

We recommend using the non-mutating method for enabling default implementation unless you need to track the changes for consistency within your code.

##### Chain stubbing
It is possible to chain stubbing. This is useful for when you need to define different behavior for multiple calls in order. The last behavior will be used for all calls after that. The syntax goes like this:

```Swift
when(stub.readWriteProperty.get).thenReturn(10).thenReturn(20)
```

which is equivalent to:

```Swift
when(stub.readWriteProperty.get).thenReturn(10, 20)
```

The first call to `readWriteProperty` will return `10` and all calls after that will return `20`.

You can combine the stubbing methods as you like.

##### Overriding of stubbing
When looking for stub match Cuckoo gives the highest priority to the last call of `when`. This means that calling `when` multiple times with the same function and matchers effectively overrides the previous call. Also more general parameter matchers have to be used before specific ones.

```Swift
when(stub.countCharacters(anyString())).thenReturn(10)
when(stub.countCharacters("a")).thenReturn(1)
```

In this example calling `countCharacters` with `a` will return `1`. If you reversed the order of stubbing then the output would always be `10`.

#### Usage in real code
After previous steps the stubbed method can be called. It is up to you to inject this mock into your production code.

Note: Call on mock which wasn't stubbed will cause an error. In case of a spy, the real code will execute.

#### Verification
For verifying calls there is function `verify`. Its first parameter is the mocked object, optional second parameter is the call matcher. Then the call with its parameters follows.

```Swift
verify(mock).greetWithMessage("Hello world")
```

Verification of properties is similar to their stubbing.

You can check if there are no more interactions on mock with function `verifyNoMoreInteractions`.

##### Argument capture
You can use `ArgumentCaptor` to capture arguments in verification of calls (doing that in stubbing is not recommended). Here is an example code:

```Swift
mock.readWriteProperty = 10
mock.readWriteProperty = 20
mock.readWriteProperty = 30

let argumentCaptor = ArgumentCaptor<Int>()
verify(mock, times(3)).readWriteProperty.set(argumentCaptor.capture())
argumentCaptor.value // Returns 30
argumentCaptor.allValues // Returns [10, 20, 30]
```

As you can see, method `capture()` is used to create matcher for the call and then you can get the arguments via properties `value` and `allValues`. `value` returns last captured argument or nil if none. `allValues` returns array with all captured values.

### 3. Matchers
Cuckoo makes use of *matchers* to connect your mocks to your code under test.

#### A) Automatic matchers for known types
You can mock any object that conforms to the `Matchable` protocol.
These basic values are extended to conform to `Matchable`:

- `Bool`
- `String`
- `Float`
- `Double`
- `Character`
- `Int`
- `Int8`
- `Int16`
- `Int32`
- `Int64`
- `UInt`
- `UInt8`
- `UInt16`
- `UInt32`
- `UInt64`

#### B) Custom matchers
If Cuckoo doesn't know the type you are trying to compare, you have to write your own method `equal(to:)` using a `ParameterMatcher`. Add this method to your test file:

```swift
func equal(to value: YourCustomType) -> ParameterMatcher<YourCustomType> {
    return ParameterMatcher { tested in
        // Implementation of your equality test.
        // ie: (try? tested.method()) == (try? value.method())
    }
}
```

⚠️ If you try to match an object with an unknown or non-`Matchable` type, it could lead to:

```
Command failed due to signal: Segmentation fault: 11
```

For details and implementation example (with Alamofire), see [this issue](https://github.com/Brightify/Cuckoo/issues/124).

#### Parameter matchers
`ParameterMatcher` also conforms to `Matchable`. You can create your own `ParameterMatcher` instances or if you want to directly use your custom types there is the `Matchable` protocol. Standard instances of `ParameterMatcher` can be obtained via these functions:

```Swift
/// Returns an equality matcher.
equal<T: Equatable>(to value: T)

/// Returns an identity matcher.
equal<T: AnyObject>(to value: T)

/// Returns a matcher using the supplied function.
equal<T>(to value: T, equalWhen equalityFunction: (T, T) -> Bool)

/// Returns a matcher matching any Int value.
anyInt()

/// Returns a matcher matching any String value.
anyString()

/// Returns a matcher matching any T value or nil.
any<T>(type: T.Type = T.self)

/// Returns a matcher matching any closure.
anyClosure()

/// Returns a matcher matching any throwing closure.
anyThrowingClosure()

/// Returns a matcher matching any non nil value.
notNil()
```

`Matchable` can be chained with methods `or` and `and` like so:

```Swift
verify(mock).greetWithMessage("Hello world".or("Hallo Welt"))
```

#### Call matchers
As a second parameter of the `verify` function you can use instances of `CallMatcher`. Its primary function is to assert how many times was the call made. But the `matches` function has a parameter of type `[StubCall]` which means you can use a custom `CallMatcher` to inspect the stub calls or for some side effect.

Note: Call matchers are applied after the parameter matchers. So you get only stub calls of wanted method with correct arguments.

Standard call matchers are:

```Swift
/// Returns a matcher ensuring a call was made `count` times.
times(_ count: Int)

/// Returns a matcher ensuring no call was made.
never()

/// Returns a matcher ensuring at least one call was made.
atLeastOnce()

/// Returns a matcher ensuring call was made at least `count` times.
atLeast(_ count: Int)

/// Returns a matcher ensuring call was made at most `count` times.
atMost(_ count: Int)
```

As with `Matchable` you can chain `CallMatcher` with methods `or` and `and`. But you cannot mix `Matchable` and `CallMatcher` together.

#### Resetting mocks
Following functions are used to reset stubbing and/or invocations on mocks.

```Swift
/// Clears all invocations and stubs of mocks.
reset<M: Mock>(_ mocks: M...)

/// Clears all stubs of mocks.
clearStubs<M: Mock>(_ mocks: M...)

/// Clears all invocations of mocks.
clearInvocations<M: Mock>(_ mocks: M...)
```

#### Stub objects
Stubs are used when you want to suppress real code. Stubs are different from Mocks in that they don't support stubbing and verification. They can be created with the same constructors as the mocked type. Name of stub class always corresponds to name of the mocked class/protocol with `Stub` suffix (e.g. stub of protocol `Greeter` is called `GreeterStub`).

```Swift
let stub = GreeterStub()
```

When method or property is called on stub nothing happens. If some type has to be returned then `DefaultValueRegistry` will provide default value. Stubs can be used to set implicit (no) behavior to mocks without the need to use `thenDoNothing()` like this: `MockGreeter().spy(on: GreeterStub())`.

##### DefaultValueRegistry
`DefaultValueRegistry` is used in Stubs to get default values for return types. It knows only default Swift types, sets, arrays, dictionaries, optionals and tuples (up to 6 values). Tuples for more values can be added with extensions. Custom types must be registered before use with `DefaultValueRegistry.register<T>(value: T, forType: T.Type)`. Default values can be changed with the same method. Sets, arrays, etc. do not have to be registered if their generic type is already registered.

Method `DefaultValueRegistry.reset()` can be used to delete all values registered by the user.

## Cuckoo generator
### Installation
For normal use you can skip this because the [run script](run) downloads and builds the correct version of the generator automatically.

#### Custom
This is a more complicated path. You can clone this repository and build it yourself. You can look at the [run script](run) for more inspiration.

### Usage
Generator can be called manually through the terminal. Each call consists of build options, a command, generator options, and arguments. Options and arguments depends on used command. Options can have additional parameters. Names of all of them are case sensitive. The order goes like this:

```
cuckoo build_options command generator_options arguments
```

#### Build Options
These options are only used for downloading or building the generator and don't interfere with the result of the generated mocks.

When the [run script](run) is executed without any build options (they are only valid when specified **BEFORE** the `command`), it simply searches for the `cuckoo_generator` file and builds it from source code if it's missing.

To download generator from GitHub instead of building it if it's missing, use the `--download [version]` option as the first argument (i.e. `run --download generate ...` or `run --download 0.12 generate ...` if you need a specific version). If you're having issues with rather long build time (especially in CI), this might be the way to fix it.

**NOTE**: If you encounter Github API rate limit using the `--download` option, the [run script](run) refers to the environment variable `GITHUB_ACCESS_TOKEN`.
Add this line (replacing the Xs with your [GitHub token](https://github.com/settings/tokens), no additional permissions are needed) to the script build phase before the `run` call:
```
export GITHUB_ACCESS_TOKEN="XXXXXXX"
```

The build option `--clean` forces either build or download of the version specified even if the generator is present. At the moment the [run script](run) doesn't enforce the generator version to be the same as the Cuckoo version. This might cause problems with outdated generator being used and the features you expect are not present. If you're having compile issues with something that should be supported, please try to use this option first to see if your generator isn't just outdated.

We recommend only using `--clean` when you're trying to fix a compile problem as it forces the build every time which makes the testing way longer than it needs to be.

#### Generator commands
##### `generate` command
Generates mock files.

This accepts options that can be used to adjust the behavior, these are listed below.

After the options come arguments, in this case a list (separated by spaces) of files for which you want to generate mocks.

###### `--output` (string)
Where to put the generated mocks.

If a path to a directory is supplied, each input file will have a respective output file with mocks.

If a path to a Swift file is supplied, all mocks will be in a single file.

Default value is `GeneratedMocks.swift`.

###### `--testable` (string)[,(string)...]
A comma separated list of frameworks that should be imported as @testable in the mock files.

###### `--exclude` (string)[,(string)...]
A comma separated list of classes and protocols that should be skipped during mock generation.  

###### `--no-header`
Do not generate file headers.

###### `--no-timestamp`
Do not generate timestamp.

###### `--no-inheritance`
Do not mock/stub parents and grandparents.

###### `--file-prefix` (string)
Names of generated files in directory will start with this prefix. Only works when output path is directory.

###### `--no-class-mocking`
Do not generate mocks for classes.

###### `--regex` (string)
A regular expression pattern that is used to match Classes and Protocols. All that do not match are excluded. Can be used alongside `--exclude` in which case the `--exclude` has higher priority.

###### `-g` or `--glob`
Activate [glob](https://en.wikipedia.org/wiki/Glob_(programming)) parsing for specified input paths.

###### `-d` or `--debug`
Run generator in debug mode. There is more info output as well as included in the generated mocks (e.g. method parameter info).

#### `version` command
Prints the version of this generator.

#### `help` command
Display general or command-specific help.

After the `help` command you can specify the name of another command for displaying command-specific information.

## Contribute
Cuckoo is open for everyone and we'd like you to help us make the best Swift mocking library. For Cuckoo development, follow these steps:
1. Make sure you have Xcode 9.1 installed
2. Clone the **Cuckoo** repository
3. In Terminal, run: `make dev` from inside the **Cuckoo** directory
4. Open `Cuckoo.xcodeproj` and peek around

The project consists of two parts - runtime and code generator. When you open the `Cuckoo.xcodeproj` in Xcode, you'll see these directories:
    - `Source` - runtime sources
    - `Tests` - tests for the runtime part
    - `CuckoGenerator.xcodeproj` - project generated by `swift package generate-xcodeproj` for the Generator sources

Thank you for your help!

## Inspiration
- [Mockito](http://mockito.org/) - Mocking DSL
- [Hamcrest](http://hamcrest.org/) - Matcher API

## Used libraries
- [Commandant](https://github.com/Carthage/Commandant)
- [FileKit](https://github.com/nvzqz/FileKit)
- [SourceKitten](https://github.com/jpsim/SourceKitten)

## License
Cuckoo is available under the [MIT License](LICENSE).
