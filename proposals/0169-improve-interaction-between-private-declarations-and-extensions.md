# Improve Interaction Between `private` Declarations and Extensions

* Proposal: [SE-0169](0169-improve-interaction-between-private-declarations-and-extensions.md)
* Authors: [David Hart](http://github.com/hartbit), [Chris Lattner](https://github.com/lattner)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Active review (April 6...11, 2017)**

## Introduction

In Swift 3, a declaration marked `private` may be accessed by anything nested in the scope of the private declaration, for example a private property or method defined on a struct may be accessed by methods defined within that struct.  

This model was introduced by [SE-0025](0025-scoped-access-level.md) and with nearly a year of experience using this model, it has worked well in almost all cases. The primary case it falls down is when the implementation of a type is split into a base definition and a set of extensions. Because of the SE-0025 model, extensions to the type are not allowed to access private members defined on that type.

This proposal recommends extending `private` access control so that members defined in an extension of a type have the same access as members defined on the type itself, so long as the type and extension are in the same source file.  We expect this to dramatically reduce the number of uses of `fileprivate` in practice.

## Motivation

[SE-0025](0025-scoped-access-level.md) defined the `private` access control level to be used for scoped access, and introduced `fileprivate` for the case when a declaration needs to be visible across declarations, but not only within a file.  The goal of the proposal was for `fileprivate` to be used rarely. However, that goal of the proposal has not been realized: Swift's reliance on extensions as a grouping and conformance mechanism has made `fileprivate` more necessary than expected.

The prevalence of `fileprivate` in practice has caused mixed reactions from Swift developers, culminating in proposal [SE-0159](0159-fix-private-access-levels.md) which suggested reverting the access control model to Swift 2’s design.  That proposal was rejected for two reasons: scoped access is something that many developers use and value, and because it was seen as too large of a change given Swift 4’s source compatibility goals.

In contrast to SE-0159, this proposal is an extremely narrow change (which is almost completely additive) to the SE-0025 model, which embraces the extension oriented design of Swift. The authors believe that this change will not preclude introduction of submodules in the future.


## Detailed Design

The only change this proposal introduces is for extensions of a type defined in the same source file as the type itself.  Simply, the access control scope of the extension is considered to be the access control scope of the extended type.

This has two ramifications: 

* Declarations in one of these extensions get access to the `private` members of the type.

* If the declaration in the extension itself is defined as `private`, then they are accessible to declarations in the type, and other extensions of that type (in the same file).

Here is a simple code example that demonstrates this:

```swift
struct S {
    private var p: Int

    func f() { 
        use(g())    // ok, g() is accessible within S
    }
}

extension S {
    private func g() {
        use(p)      // ok, g() has access to p, since it is in an extension on S.
    }
}

extension S {
    func h() {
        use(g())    // Ok, h() has access to g() since it defined in the access control scope for S.
    }
}
```

Please note:

* This visibility does **not** extend to subclasses of a class in the same file, it only affects extensions of the type itself.

* Constrained extensions are extensions, so this visibility **does** extend to them as well. For example, the body of `extension Optional where Wrapped == String { }` would have access to `private` members of Optional, assuming the extension is defined in the same file as `Optional`.

* This proposal does not change the behavior of extensions that are not in the same file as the type - i.e., extensions in a seperate file to the type do not share access between themselves:

```swift
// FileA.swift
struct A {
}

// FileB.swift
extension A {
    private func foo() {
        bar()    // not ok, foo() does not have access to bar()
    }
}

extension A {
    private func bar() {
    }
}
```

* This proposal does not change access control behavior for types nested within each other. As in Swift 3, inner types have access to the private members of outer types, but outer types cannot refer to private members of inner types.  For example:

```swift
struct Outer {
    private var outerValue = 42
    
    struct Inner {
        private var innerValue = 57

        func innerTest() {
            print(outerValue)    // still ok.
        }
    }

    func test(_ i: Inner) {
        print(i.innerValue)      // still an error
    }
}
```

## Source Compatibility

In Swift 3 compatibility mode, the compiler will continue to treat `private` as before. In Swift 4 mode, the compiler will modify the semantics of `private` to follow the rules of this proposal. No migration will be necessary as this proposal merely broadens the visibility of `private`.

Cases where a type had `private` declarations with the same signature in the same type/extension but in different scopes will produce a compiler error in Swift 4. For example, the following piece of code compiles in Swift 3 compatibilty mode but generates a `Invalid redeclaration of 'bar()'` error in Swift 4 mode:

```swift
struct Foo {
    private func bar() {}
}

extension Foo {
    private func bar() {}
}
```

## Alternatives Considered

Access control has been hotly debated on the swift-evolution mailing list, both in the Swift 3 cycle (leading to [SE-0025](0025-scoped-access-level.md) and most recently in Swift 4 which led to [SE-0159](0159-fix-private-access-levels.md). There have been too many proposals to summarize here, including the introduction of a `scoped` keyword.

The core team has made it clear that most of those proposals are not in scope for discussion in Swift 4 (or any later release), given the significant impact on source compatibility. This is the primary reason for this narrow scope proposal.

A future direction that may be interesting to consider and debate (as a separate proposal, potentially after Swift 4) is whether extensions within the same file as a type should be treated as parts of the extended type **in general**.  This would allow idioms like this, for example:


```swift
struct Foo {
    var x: Int
}
// ...
extension Foo {
    var y: Int
}
```

However, this is specifically **not** part of this proposal at this time. It is merely a possible avenue to consider in the future.