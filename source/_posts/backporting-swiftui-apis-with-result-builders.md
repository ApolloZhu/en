---
title: Backporting SwiftUI APIs with Result Builders
tags:
  - kde
  - KDE Connect
categories:
  - Programming
date: 2023-04-17 12:42:46
---

When Apple announced SwiftUI back in 2019, there's a new language feature called "function builder" that didn't go through the Swift Evolution process but was shipped with Apple's Swift toolchain to make SwiftUI a reality. I'm glad that the community decided to adopt this feature as result builders, allowing us to mimic SwiftUI's API ourselves.

<!-- more -->

{% note danger %}

## Disclaimer

This code is written without knowledge of how Apple's SwiftUI framework actually works.
{% endnote %}

## Motivation

When [KDE Connect iOS](https://invent.kde.org/network/kdeconnect-ios) tried to lower deployment target to iOS 14, we ran into the limitation of [not being able to show multiple alerts in the same View](https://sarunw.com/posts/how-to-show-multiple-alerts-on-the-same-view-in-swiftui/), and had to find a very hacky [workaround](https://invent.kde.org/network/kdeconnect-ios/-/merge_requests/16) for iOS 14 only by adding many hidden views to the view hierarchy, and duplicating the buttons/texts:

```swift
if #available(iOS 15.0, *) {
  content
    .alert(title1, isPresented: $isPresented1) {
      primaryButton1
      secondaryButton1
    } message: {
      subtitle1
    }
    .alert(...)
} else {
  content

  hiddenView1
    .alert($isPresented1) {
      Alert(
        title: title1,
        message: subtitle1,
        primaryButton: primaryButton1,
        secondaryButton: secondaryButton1
      )
    }

  hiddenView2
    .alert(...)
}
```

## Proposed Solution

By mimicking SwiftUI's new `alert` API but making it available on iOS 14, we could hide the complexity away from the call site allowing only writing code as in the iOS 15 branch, while doing all the dirty work inside the actual implementation:

```swift
extension View {
  func alert(
    _ titleKey: LocalizedStringKey,
    isPresented: Binding<Bool>,
    @AlertActionBuilder actions: () -> AlertActionBuilder.Buttons?,
    @ViewBuilder message: () -> Text?
  ) -> some View
}
```

But wait, what is `AlertActionBuilder`? To understand why we need to introduce this new thingy, let's first look at what's `ViewBuilder`, and why we can't use `ViewBuilder` for our purpose.

## Detailed Design

### `ViewBuilder`

The new iOS 15 SwiftUI alert API is defined as follows:

```swift
extension View {
  public func alert<A, M>(
    _ titleKey: LocalizedStringKey,
    isPresented: Binding<Bool>,
    @ViewBuilder actions: () -> A,
    @ViewBuilder message: () -> M
  ) -> some View where A : View, M : View
}
```

Note the `@ViewBuilder` attribute in front of `actions` and `message`: this is what enables us to write code like:

```swift
var body: some View {
  Button(...)
  Button(...)
}
```

Normally, when the Swift compilers sees values that are not used as part of another expression, assigned to a variable, or returned, it will complain about "Result of ... is unused," as you can see by writing the same code but for a computed property that's not a `View`'s [`body`](https://developer.apple.com/documentation/swiftui/view/body-swift.property):

```swift
var buttons: some View {  // Function declares an opaque return type,
                          // but has no return statements in its body
                          // from which to infer an underlying type
  Button(...) // Result of 'Button<Label>' initializer is unused
  Button(...) // Did you mean to return the last expression?
}
```

However, by annotating it with SwiftUI's custom `ViewBuilder` attribute, the code now compiles by building a combined result after getting transformed using [rules specified by `ViewBuilder`](https://developer.apple.com/documentation/swiftui/viewbuilder#building-content), which the final result made using its `buildBlock` function is then [implicitly returned](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Shorthand-Getter-Declaration):

```swift
@ViewBuilder
var buttons: some View {
  /* let v1 = */ Button(...)
  /* let v2 = */ Button(...)
  // return buildBlock(v1, v2)
}
```

Because code annotated with the `ViewBuilder` attribute doesn't follow how Swift code normally gets compiled, it's kind of like a miniature language within Swift specialized for constructing views from closures (i.e. things within braces). We call this kind of language a **Domain Specific Language**, or **DSL**.

### Differences Between iOS 14 and iOS 15 API

While the iOS 15 API allows arbitrary views as the list of `actions`, the iOS 14 API requires either no buttons, a single dismiss button, or a primary and a secondary button. This is fine as `ViewBuilder`'s `buildBlock` function returns a `TupleView`, from which we can extract the buttons to pass to iOS 14's API. However, the bigger problem is that `ButtonRole` is only available on iOS 15 and we can't peek inside SwiftUI's `Button` `struct` to figure out what role it has -- nor can we do so on iOS 14 where the `ButtonRole` type doesn't exist. Thus, we'll need our own type to store relevant information then later convert it to `Button` on iOS 15 and `Alert.Button` on iOS 14:

```swift
struct _Button {
  enum Role {
    case cancel, destructive
  }

  let titleKey: LocalizedStringKey
  let role: Role?
  let action: () -> Void

  @available(iOS, introduced: 15)
  var iOS15Button: some View { ... }

  @available(iOS, deprecated: 15)
  var iOS14Button: Alert.Button { ... }
}
```

### Making `AlertActionBuilder` - a DSL for Building Alert Actions

Making a DSL in Swift using result builder is very simple: declaring a new type and annotate it with the `resultBuilder` attribute, then provide at least one static `buildBlock`/`buildPartialBlock` method:

```swift
@resultBuilder
enum AlertActionBuilder {
  static func buildBlock(/* TODO */) -> /* TODO */ { ... }
}
```

To allow building the 3 types of alert buttons mentioned above, we can represent the result as an `Optional<AlertActionBuilder.Buttons>`:

```swift
extension AlertActionBuilder {
  enum Buttons {
    case dismiss(_Button)
    case primary(_Button, secondary: _Button)
  }
  // AlertActionBuilder.Buttons? has an additional case
  // `nil` to represent no buttons.
```

The Swift compiler will try to match contents inside an `@AlertActionBuilder` closure with the `buildBlock` functions defined. For example, if there's nothing inside the curly braces, it will choose:

```swift
  static func buildBlock() -> Buttons? {
    return nil
  }
```

Similarly, it will do so for the one button and two buttons cases:

```swift
  static func buildBlock(_ button: _Button) -> Buttons? {
    return .dismiss(button)
  }

  static func buildBlock(_ button1: _Button, _ button2: _Button) -> Buttons? {
    // Maybe switch the order depending what roles each of these buttons has
    return .primary(button1, secondary: button2)
  }
}
```

That's it! We can now write code like:

```swift
content
  .alert(title, isPresented: $isPresented) {
    _Button("Unpair", role: .destructive) { ... }

    _Button("Cancel", role: .dismiss) { ... }
  }
  .alert(...)
```

and switch on `AlertActionBuilder.Buttons?` inside our `alert` API implementation to call appropriate SwiftUI APIs on both iOS 14 and iOS 15. The need to use `_Button` instead of `Button` is annoying though, so let's use the tricked mentioned in {% post_link shimming-swiftui-apis-by-hacking-overload-resolution %} to fix that:

```swift
func Button(
  _ titleKey: LocalizedStringKey,
  role: _Button.Role? = nil,
  action: @escaping () -> Void
) -> _Button {
  _Button(titleKey, role: role, action: action)
}
```

Way to go!

## Further Readings

The full implementation can be found at [[Refactor] Reduce code duplication for iOS 14 support](https://invent.kde.org/network/kdeconnect-ios/-/merge_requests/34). To learn more about building DSLs and using result builders, you can checkout:

- [SE-0289: Result builders](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md)
- [SE-0348: `buildPartialBlock` for result builders](https://github.com/apple/swift-evolution/blob/main/proposals/0348-buildpartialblock.md)
- [WWDC21-10253: Write a DSL in Swift using result builders](https://developer.apple.com/videos/play/wwdc2021/10253/)
- [Improved Result Builder Implementation in Swift 5.8](https://forums.swift.org/t/improved-result-builder-implementation-in-swift-5-8/63192)
- [`ApolloZhu/BoolBuilder`: `@resultBuilder` for building a `Bool`](https://github.com/ApolloZhu/BoolBuilder)

On a side note, I'm not sure what [macros](https://github.com/apple/swift-evolution/blob/main/visions/macros.md) -- which is different from result builders and [property wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md) though all of them begin with an `@` -- will impact how people approach implementing DSLs in Swift in the future, but they are certainly interesting for library authors to explore as well.
