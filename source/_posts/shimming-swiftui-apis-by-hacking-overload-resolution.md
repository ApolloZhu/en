---
title: Shimming SwiftUI APIs by Hacking Overload Resolution
tags:
  - kde
  - KDE Connect
  - Swift
categories:
  - Programming
date: 2023-04-13 23:52:50
---

The best way to build an app is with Swift and SwiftUI -- if you don't have to support older iOS versions. <!-- more -->

## Motivation

Because Swift doesn't support back deploying types and only recently implemented [SE-0376 Function Back Deployment](https://github.com/apple/swift-evolution/blob/main/proposals/0376-function-back-deployment.md), SwiftUI as a system framework can only introduce new features for the latest iOS version. As [KDE Connect iOS](https://invent.kde.org/network/kdeconnect-ios) needs to support iOS 14, if we want to use some of the newer SwiftUI APIs, we have to check what's the iOS version of the current device, then execute different branches of code depending on what API is available:

```swift
if #available(iOS 15, *) {
  theSameActualViewContent
    .alert(title, isPresented: $isPresented) {
      primaryButton
      secondaryButton
    } message: {
      someText
    }
} else {
  theSameActualViewContent
    .alert(isPresented: isPresented) {
      Alert(
        title: title,
        message: someText,
        primaryButton: primaryButton,
        secondaryButton: secondaryButton
      )
    }
}
```

Not only do we have to add the `if #available` checks, to reduce code duplication, we need to introduce variables like `theSameActualViewContent`, `title`, and `primaryButton`. If we have multiple alerts, we have to repeat this process for every single one of them. The friction makes writing SwiftUI not fun anymore.

## Proposed Solution

One way to solve this is to "bring the new APIs to an older environment, using only the means of that environment," or implement what people call a "**shim**."

{% note info %}
Other common terms that describes this are "backport" and "polyfill."
{% endnote %}

While libraries like [SwiftUI Backports](https://github.com/shaps80/SwiftUIBackports) exists, there are still times that we need to do this ourselves if the library hasn't gotten to the thing we want, such as support for `@FocusState` on iOS 14. In addition, to make it easier to later drop support for older iOS versions, it's possible -- and the easiest -- for in house wrapper/implementation to **"abuse" overload resolution** to keep the shim API exactly the same as SwiftUI at call sites. This means, instead of needing to add some disambiguator like in [this blog post](https://davedelong.com/blog/2021/10/09/simplifying-backwards-compatibility-in-swift/):

```swift
@Backport.FocusState var isFocused
...
view.backport.refreshable { ... }
```

or using a different name such as in [this YouTube video](https://www.youtube.com/watch?v=2IB4CuSRea4) and many other online tutorials:

```swift
@MyFocusState var isFocused
...
view.myRefreshable { ... }
```

the goal is directly write:

```swift
@FocusState var isFocused
...
view.refreshable { ... }
```

as if the shim doesn't exist.

## Overload Resolution

When multiple types, functions, and/or variables from different modules share the same name `X`, the compiler needs to figure out which one exactly are you referring to when you write `X` in source code. According to Swift's documentation on [resolving name lookup ambiguities](https://github.com/apple/swift/blob/main/docs/Modules.rst#ambiguity):

> 1. Declarations in the current source file are best.
> 2. Declarations from other files in the same module are better than declarations from imports.
> 3. Declarations from selective imports are better than declarations from non-selective imports. (This may be used to give priority to a particular module for a given name.)
> 4. Every source file implicitly imports the core standard library as a non-selective import.
> 5. If the name refers to a function, normal overload resolution may resolve ambiguities.

That means we could provide shims by declaring:

1. a type alias called `FocusState` to shadow SwiftUI's definition of `FocusState` type

```swift
typealias FocusState = State
```

2. an "overload" global function that's similar to SwiftUI's definition of `Button.init` initializer but with custom types available on iOS 14

```swift
func Button(
  _ titleKey: LocalizedStringKey,
  role: _ButtonRole? = nil,
  action: @escaping () -> Void
) -> _Button {
  ...
}
```

3. an overload function called `refreshable` with a slightly different signature but indistinguishable from call site if using trailing closures

```swift
extension View {
  // This has signature refreshable(_:) while the SwiftUI one is refreshable(action:)
  func refreshable(_ action: @escaping @Sendable () async -> Void) -> some View {
    ...
  }
}
```

Since these are "Declarations from other files in the same module," they are "better" than declarations from imported SwiftUI framework. As they are available on iOS 14, the compiler will happy take these shims over the actual SwiftUI APIs.

## Migration

What needs to happen when we drop support for iOS 14? Just delete the files implementing the shims. To make sure we remember doing this, we can mark the shim APIs to be obsolete by the iOS version they become available at:

```swift
@available(iOS, obsoleted: 15,
           message: "Delete this file and use SwiftUI.FocusState instead.")
typealias FocusState = State
```

Since the backport implementation is different from SwiftUI, always test the app to make sure other parts of the code base are not relying on shim-specific behaviors.
