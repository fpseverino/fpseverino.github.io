---
layout: post
title: "The Dangers of Protocol Conformances in Swift"
category: swift
---

A recent release of Swift Log (v1.8.0), which includes a PR I authored, has caused a ripple effect of unexpected breaking changes across the Swift Package ecosystem.
This post explores what happened, why it happened, and how we can avoid similar issues in the future.

## The Pull Request

In Swift server-side applications, logging is handled by [Swift Log](https://github.com/apple/swift-log), a logging API and implementation library maintained by Apple, widely used across the ecosystem.

As part of my contributions to Vapor 5, I moved some logging-related functionality from Vapor to ConsoleKit v5 and Swift Configuration.
One of the few things left in Vapor in that regard was a simple extension to make `Logger.Level` (a type from Swift Log) `@retroactive`ly conform to `CustomStringConvertible` and `LosslessStringConvertible`, two standard library protocols.

`@retroactive` is a Swift attribute that warns you when you conform types you don't own to protocols you don't own.
It exists to make you aware of the risks you're taking: if the type later adopts the same protocol, you end up in non-deterministic behavior, as the compiler won't know which conformance to use.
You can read more about `@retroactive` in [SE-0364](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md).

Going back to the `@retroactive` conformances in Vapor, since these were conformances to standard library protocols that I knew were fairly widely replicated across the ecosystem and that would be generally useful, I decided to propose moving them directly into Swift Log instead.
So, I opened [apple/swift-log#395](https://github.com/apple/swift-log/pull/395), motivating the reasons behind this change.

> Contents of the PR (minus tests):

```swift
extension Logger.Level: CustomStringConvertible, LosslessStringConvertible {
    /// A textual representation of the log level.
    public var description: String {
        self.rawValue
    }

    /// Creates a log level from its textual representation.
    /// - Parameter description: A textual representation of the log level, case insensitive.
    public init?(_ description: String) {
        self.init(rawValue: description.lowercased())
    }
}
```

The PR was reviewed, accepted and merged, and Swift Log v1.8.0 was released shortly after.
It was rightly released as a minor version bump, as conformances to standard library protocols and new APIs, such as computed variables and initializers, are not considered breaking changes by SemVer 2.0.

## Unexpected Consequences

Shortly after the release, discussions started about how SwiftPM would handle the `@retroactive` conformances in libraries.

> @fpseverino on Discord:
>
> It's gonna look great on my resume: "Destroyed the whole Swift package ecosystem"

Soon enough, conflicts started to appear: the first library to fall victim was Vapor itself.

### @retroactive Conformances

When trying to use `logLevel.description` or the `LosslessStringConvertible` initializer, users of Vapor would get a compiler error about ambiguous use of those members.

![Vapor error screenshot](/assets/protocol-conformances/vapor-error.png)

The issue was that both Vapor and Swift Log were providing conformances to those protocols for `Logger.Level`.
As previously mentioned, the conformances in Vapor were marked as `@retroactive`, which means that you, the author of said conformance, acknowledge that if the type being conformed to (which you don't own) later provides the same conformance you end up in non-deterministic behavior.
And that is exactly what happened here: Swift Log added the same conformances, and now the compiler couldn't decide which one to use.
The issue was quickly fixed by removing the conformances from Vapor, and a new release ([v4.120.0](https://github.com/vapor/vapor/releases/tag/4.120.0)) was cut.

### Combining Multiple Protocol Conformances

Next, a different issue appeared in the Hummingbird template.
The template uses Swift Argument Parser to get the log level from command line arguments, so it had an empty extension that `@retroactive`ly made `Logger.Level` conform to `ExpressibleByArgument`.

```swift
extension Logger.Level: @retroactive ExpressibleByArgument {}
```

That worked because Swift Argument Parser has two extensions to `RawRepresentable` and `LosslessStringConvertible` that provide a default initializer for `ExpressibleByArgument` for types conforming to one of those protocols.

```swift
extension RawRepresentable where Self: ExpressibleByArgument, RawValue: ExpressibleByArgument {
    public init?(argument: String) {
        if let value = RawValue(argument: argument) {
            self.init(rawValue: value)
        } else {
            return nil
        }
    }
}

extension LosslessStringConvertible where Self: ExpressibleByArgument {
    public init?(argument: String) {
        self.init(argument)
    }
}
```

Before Swift Log v1.8.0, `Logger.Level` only conformed to `RawRepresentable`, so there was no ambiguity.
After the release, however, `Logger.Level` conformed to both `RawRepresentable` **and** `LosslessStringConvertible`, so the compiler couldn't decide which initializer to use.
So the issue really lied in Swift Argument Parser, which didn't account for the possibility of a type conforming to both protocols.
The Hummingbird template _has_ to use a `@retroactive` conformance in this case, as Swift Argument Parser and Swift Log are (presumably) never going to depend on each other.
The Hummingbird template was quickly fixed too, by providing a custom implementation to `ExpressibleByArgument` instead.
The ambiguity issue was fixed in Swift Argument Parser as well, in [apple/swift-argument-parser#841](https://github.com/apple/swift-argument-parser/pull/841), by adding a more specific extension for types conforming to both protocols.

### Capturing Function Signatures without Spelling Out the Parameter Labels

The last library (so far) to be affected was the Swift AWS Lambda Runtime, and the issue was completely different from the previous two.
The library didn't provide any `@retroactive` conformance, neither did it expect `Logger.Level` to conform to any protocol, like in the Hummingbird/ArgumentParser case.
Instead, the issue was that Swift AWS Lambda Runtime used the `Logger.Level` initializer provided by `RawRepresentable` inside a mapping function, passing directly the initializer signature to `flatMap`, without spelling out the parameter label, like so:

```swift
log.logLevel = Lambda.env("LOG_LEVEL").flatMap(Logger.Level.init) ?? logger.logLevel
```

The compiler was able to resolve the call to `Logger.Level.init` before Swift Log v1.8.0, because there was only one initializer available that took a `String`: the one provided by `RawRepresentable`.
After the release, however, there was now a second initializer available, the one provided by `LosslessStringConvertible`, that also takes a `String`, so the compiler couldn't decide which one to use.
The issue has been fixed in [awslabs/swift-aws-lambda-runtime#619](https://github.com/awslabs/swift-aws-lambda-runtime/pull/619) by calling `flatMap` with a closure that explicitly calls the desired initializer, like so:

```swift
log.logLevel = Lambda.env("LOG_LEVEL").flatMap { .init(rawValue: $0) } ?? logger.logLevel
```

Another possible fix would have been to spell out the parameter label when passing the initializer to `flatMap`, like so:

```swift
log.logLevel = Lambda.env("LOG_LEVEL").flatMap(Logger.Level.init(rawValue:)) ?? logger.logLevel
```

## What We Learned

There are a few lessons that I think we can learn from this incident:

1. We shouldn't extend types we don't own with `@retroactive` conformances to protocols we don't own, especially `public`ly.
    This applies in particular to standard library protocols, which are more likely to be adopted by types in the future.
    `@retroactive` and its warning exist exactly for this reason, to make you aware of the risks you're taking.

2. While very similar, `RawRepresentable`, `CustomStringConvertible` and `LosslessStringConvertible` serve different purposes, and it's not unlikely that a type would want to conform to more than one of them.
    Libraries that provide default implementations based on those protocols (like Swift Argument Parser) should take this into account, and avoid ambiguities.

3. We should put extra thought when working with remote library packages, both as users and as contributors/maintainers.
    As users, we should be aware that we don't own the code, and even if it follows SemVer 2.0 it might change in unexpected ways, as we've seen.
    As contributors and maintainers, we should be aware that even additive changes, especially to widely used libraries like Swift Log, can have far-reaching and unpredictable consequences (see also [Hyrum's Law](https://www.hyrumslaw.com)).

4. And last but not least, the Swift open source community once again proved to be amazing, quickly identifying and fixing the issues caused by this change.
    Thanks to everyone involved in the discussions and fixes!
