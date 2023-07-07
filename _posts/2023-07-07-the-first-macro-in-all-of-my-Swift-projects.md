---
title: 'The First macro in all of my Swift Projects'
date: 2023-07-07 09:00:00
excerpt: One of the highlight features of Swift 5.9 is the macro system. There’s one macro in particular that I am excited about adopting in a variety of projects.
---

Last month, WWDC 2023 came packed with exciting new developments for the iOS community, as it always does. Every year, one of the most anticipated talks for me is [What’s new in Swift](https://developer.apple.com/videos/play/wwdc2023/10164/), because beyond writing iOS apps, I like to [leverage Swift for server-side projects too](https://vapor.codes).

With this WWDC comes Swift 5.9, and I'd describe the set of changes overall as somewhat low-level and nontrivial to grasp. I don't anticipate using the ownership APIs or C++ interoperability, and while I'm sure I'll discover a use case for type parameter packs, there is a lot of new syntax to process. Macros are definitely nontrivial to grasp as well, but the value proposition to developers is pretty clear - **extend the compiler in useful ways that previously required custom tooling for code generation**. Historically, when iOS developers have found repetitive boilerplate that existing tools can’t help with automating, [Sourcery](https://github.com/krzysztofzablocki/Sourcery) had been the best tool for the job. Both Sourcery and the new Macro system fundamentally work by allowing you to examine your code as an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) and output some new Swift code to compile alongside your manually-written code. Even as the Swift compiler improves, there are still a bunch of common use cases for such a tool:
* Custom Equatable, Hashable and Codable implementations
* Generating mocks for type declarations
* Mapping user preferences in User Defaults to a UIKit/SwiftUI form

If you want to learn more about macros, their benefits, and now they work, you should watch the WWDC session [Expand on Swift macros](https://developer.apple.com/videos/play/wwdc2023/10167) first, and then [Write Swift macros](https://developer.apple.com/videos/play/wwdc2023/10166/).

There’s one mainstay use case in most of my projects, and to explain why I find it so common, I need to back up and talk about a stylistic discussion that also occurs a lot in Swift projects.

# You Might Not Want an Enum

The situation I want to examine is when a Swift project has a **data type with a bunch of preset values**. That’s (intentionally) a pretty vague description, and it can include:
* The set of all HTTP methods: GET, PUT, POST, DELETE, and so on.
* Possible roles for users on a platform: Admin, Moderator, User, Guest, etc.
* Models of non-player characters in a video game.
* A client-side list of supported countries and their flags.
* A set of themes for users to customize their UI.

The list of scenarios fitting this criteria is quite long, and the related style discussion that comes up is: **should the data type be a struct or an enum**? Both of the following definitions have the same callsite and usage semantics.
```swift
enum Country {
  // ...
  case unitedStates
  // ...

  var name: String {
    switch self {
    // ...
    case .unitedStates: return "United States"
    // ...
    }
  }
}
```
and:
```swift
struct Country {
  let name: String
  init(name: String) {
    self.name = name
  }

  // ...
  static let unitedStates = Country(name: "United States")
  // ...
}
```
can both be called as (for example):
```swift
print(Country.unitedStates.name)
```
Enums have many valid use cases and are a critical part of the Swift language, but in the above described situations, I would say the *struct is preferable here*. I'm going to keep the "why" brief, because there already exist some good resources on this topic, but:
* It’s easier to extend with new values. When adding a new case to the enum, you need to fill in the `case` inside of `var name: String` - which sounds like a small deal, but at scale, these value types will often have _many_ more than 1 property. With a struct, all of those properties are localized together.
* You can lock down the initialization (if you want): with a struct, I can choose to make the `init` private, or I can customize which properties are directly instantiable. With enums, you can add a case if you’re inside the owning module, and not otherwise.
* At scale, it’s easy for convoluted logic or accidental mismatches to occur inside the computed `var`s. With structs, all you can do is declare the type.

For more on this choice and why I advocate for structs, see [Matt Diephouse’s post](https://matt.diephouse.com/2017/12/when-not-to-use-an-enum/).

## You Probably Want to Iterate

While I've established that I think structs are the right tool for this job, regardless of implementation, it will often be valuable to **iterate over this set of static values**. Maybe you want to show a modal UI with all of the supported `Country` values, or all of the themes in your app, or you want to snapshot test the JSON representation of all of the user roles. Admittedly, this is the one area where `enum`s have a leg up - Apple supports this out of the box by just conforming to [CaseIterable](https://developer.apple.com/documentation/swift/caseiterable):
```swift
extension Country: CaseIterable {} // works for free if using an enum, not for a struct!
print(Country.allCases.map(\.name))
```
So, how do we patch this gap in functionality with structs? This is a perfect use case for (historically) Sourcery and (now) Swift macros! If the above examples don’t sound valuable, I’ll be clear that I find this a _very_ valuable piece of functionality because 1) every app always has these data types with a bunch of preset values and 2) I find they very often end up being representable as JSON, some UI, or both. Being able to iterate over the set means you can get new JSON snapshot tests, UI snapshot tests, SwiftUI previews and more, without any additional work when you add a new value.

So, that's why I have a `StaticMemberIterable` in most of my projects.

## `StaticMemberIterable`

As mentioned above, this functionality is a good example of a gap in current compiler functionality, which we can patch up via code generation - either via Sourcery, or, starting with Xcode 15, Swift macros.

You can see my Sourcery implementation of `StaticMemberIterable` [here](https://gist.github.com/nevillco/aec0c67a7457a99fb220336614bc8184) as a gist. You’ll see that it works with Sourcery’s annotations system, which means you just need to add a comment above the struct to opt-in to the `StaticMemberIterable` code generation. The template could easily be tweaked to make it work by conforming your struct type to a protocol, however: the template is a `.swifttemplate` file, which, while [documented](https://github.com/krzysztofzablocki/Sourcery/blob/master/guides/Writing%20templates.md#swift-templates), is not as easy to work with or diagnose errors as regular Swift - that’s one of the handful of reasons why macros are a big improvement to this implementation.

As far as the Macro-version of the implementation, I’m using Ian Keen’s [MacroKit](https://github.com/IanKeen/MacroKit), and the outputted code is functionally identical as before.

So, why is the Macro implementation so much better?
* They are written as pure Swift, with compile-time checking and the ability to unit test your `StaticMemberIterable` implementation (not that I’ve gotten around to that part!). 
* They are installed via a plugin system, which has its own overhead, but with Sourcery at some level you will need to invoke it via shell, passing arguments (or providing a .yml file) and making sure it’s being executed as (usually) a build phase. The implementation can often require its own code in your CI system. At scale, macros can be iterated on more effectively and safely, because their entire integration is integrated and visible with Xcode.
* Macros are predictable and additive - every declaration or expression that is affected by a Macro is given a clear annotation, and macros can only add and extend code, not modify or delete it. Xcode has first class support for viewing the diff produced by the Macro, inline. Sourcery is also additive, but as mentioned earlier, its tooling stack often goes outside of Xcode and can have a higher learning curve.

## Future Directions

`StaticMemberIterable` is a good addition to a lot of projects, and I'm excited to see developers push other brand new compiler capabilities with macros. A few things on this topic I’m still thinking about:
* A separate `@ViewFromStaticMemberIterable` Macro that must be attached to a SwiftUI View, provides a type that conforms to `StaticMemberIterable`, and produces snapshots for each instance. I could write a single `XCTestCase` that iterates over them in a `for` loop, but test failure reporting is subpar that way - all failures point to the same line and it's not clear which instances had their tests fail. Better to generate a whole `XCTestCase` with a test function per instance.
    * I’m a big proponent of snapshot testing. There is enough to be said on snapshot testing to merit a separate post, but in short, I think it’s a low-effort, high-reward way of adding test coverage to iOS apps. I am a fan of PointFree’s snapshot testing [framework](https://github.com/pointfreeco/swift-snapshot-testing). If you already use snapshot tests in your project, I would say the likelihood increases that `StaticMemberIterable` is useful to you in some capacity.
* Extend `StaticMemberIterable` to include values produced by static _functions_, if their only argument is also `CaseIterable` or `StaticMemberIterable`:

```swift
@StaticMemberIterable(recursive: true)
struct AppTheme {
  static let skyBlue: AppTheme = //...
  static let metal: AppTheme = // ...

  // Improve `StaticMemberIterable` to produce cases like
  // `.promotionTheme(.holiday), .promotionTheme(.anniversary), etc.
  // for this function if `PromotionKind` is `CaseIterable` or `StaticMemberIterable`,
  // and if `recursive` is true.
  static func promotionTheme(promotionKind: PromotionKind) -> AppTheme {
    // ...
  }

}
```

This would be a substantial test of the SwiftSyntax macros API, but I believe should be possible.

That’s all for now on this topic. In addition to the resources linked above, I also recommend checking out the [swift-macro-examples Repo](https://github.com/DougGregor/swift-macro-examples) if you’re just getting started with this new language feature.
