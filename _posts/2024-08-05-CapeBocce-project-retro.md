---
title: 'A tvOS Project Retro: Introducing (and Open-Sourcing) the Cape Bocce Bracket App'
date: 2024-08-05 09:00:00
excerpt: Detailing my experience building a bocce tournament tvOS app for a family vacation.
---
During my childhood, my immediate family and some members of my extended family began a tradition to go vacationing on Cape Cod once per year. This trip is an incredibly fond memory for myself and everyone involved, so it was very exciting when we decided to go back this year for the first time in a decade.

My whole family enjoys games - just about anything for which we can keep score. One routine formed where we’d draw up a tournament bracket on a piece of paper, and everyone would play bocce in the driveway. We would often make this trip during the Olympics, so participants would pick a country to represent as a fun extra touch.

With the return of the vacation this summer, I thought I would upgrade that piece of paper to an Apple TV app. I had a pretty clear scope in mind based on previous years, and about a month or so before the trip, so I felt it would be a good chance to learn a new platform with a concrete goal. The app would need to:
* Allow inputting 8-12 participants with a name and a chosen country.
* Generate a double elimination bracket with the input list of participants (everybody gets to play at least twice).
* Allow the user to choose a winner for each match from the opening round to the championship.
* Have some sort of “congratulations” screen for the champion, for a bit of surprise and delight.
* Show a list of past brackets.
* Support just our family - an App Store release and other production-izing would be a non-goal.

The vacation wrapped up last week, and the Cape Bocce tvOS app produced 7 very fun tournaments. Here is a preview of the app in action:

<div style="text-align: center;">
  <video width="320" height="570" controls="controls">
    <source src="https://media.githubusercontent.com/media/nevillco/nevillco.github.io/main/images/CapeBocce/01-live-preview.mov" type="video/mp4">
  </video>
</div>
And here is a full walkthrough of the app’s user experience:

<div style="text-align: center;">
  <video width="640" height="320" controls="controls">
    <source src="https://github.com/nevillco/nevillco.github.io/blob/main/images/CapeBocce/02-full-walkthrough.mp4?raw=true" type="video/mp4">
  </video>
</div>

In this post I’m going to write about the aspects of the project I found interesting, and walk through parts of the [code, which I’ve open-sourced](https://github.com/CleanFoundry/CapeBocce-tvOS). 

## Architecture
### Tuist
[Tuist](https://tuist.io/) is a toolchain and CLI meant to help scale Xcode projects. It provides a variety of benefits, but at the core, it generates a `xcodeproj`/`xcworkspace` from a Swift manifest.

Bringing in any 3rd party dependency should require evaluation of tradeoffs. Tuist is an opinionated tool, and it comes with a learning curve, but given its benefits, I opt to use it on any project I can.
* Packages can be [cached](https://docs.tuist.io/guides/develop/projects/dependencies#tuist-s-xcodeproj-based-integration) as static frameworks (or other output formats), meaning you will never see “Resolving packages…” in Xcode - a large productivity gain.
* Since your `xcodeproj`/`xcworkspace` is generated based on your Swift manifest, they can be added to your `.gitignore`, eliminating painful merge conflicts.
* Modularizing a project, in a much more uniform fashion, is easy. Rather than manually tweaking Xcode settings, you declare your workspace, project, targets, test targets and schemes as [Swift values](https://docs.tuist.io/guides/develop/projects/manifests#manifests). You can declare any abstractions around these values you want. Adding a new target, with a well-defined dependency graph, is just a [~20 line Swift representation of the target](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/main/Tuist/ProjectDescriptionHelpers/Targets/CompleteMatchKit.swift) and [1 line to add it to the project](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Tuist/ProjectDescriptionHelpers/CFProject%2BCapeBocce.swift#L18). 
  * I took the code reuse a step further and broke out a layer of my preferred Tuist abstractions as a separate repo, so it may be reused in future projects using Tuist’s [plugin system](https://docs.tuist.io/guides/develop/projects/plugins#project-description-helper-plugin). I open sourced [that repo](https://github.com/CleanFoundry/cf-tuist-helpers) for reference too.

![An image of the CapeBocce directory structure, where each folder is a separate module.](/images/CapeBocce/03-tuist-directory-structure.png)

I should caveat that while I’ve used Tuist in a handful of projects, I haven’t migrated an existing project, so I can’t be sure of the difficulty there, but there seem to be [detailed docs](https://docs.tuist.io/guides/start/migrate/xcode-project) and some tooling to support that process.

Tuist supports tvOS out of the box, so I went forward with using it here. Another perk of Tuist is, if someone chose to stop using it, that would simply require de-listing my `xcodeproj` and `xcworkspace` from the `.gitignore`, so users aren’t locked into the tool.
### The Composable Architecture (and Friends)
Another tool I often like to adopt is [The Composable Architecture](https://github.com/pointfreeco/swift-composable-architecture) (TCA), a library by [PointFree](https://www.pointfree.co/). TCA can be thought of as a design pattern akin to MVVM or RIBs, as well as the package that supports the design pattern. It prioritizes testability, developer ergonomics, and composition. 

Tradeoffs: like Tuist, TCA is an investment at the architectural level with a significant learning curve, but all of PointFree’s libraries have top-notch documentation, sample code, and an active Slack channel for support. TCA achieves a good developer experience and a low level of boilerplate partly through the use of [Swift Macros](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/macros/), which do come with a significant [hit to clean build time](https://forums.swift.org/t/compilation-extremely-slow-since-macros-adoption/67921), but the benefits provided by the library, to me, outweigh the cons for most Apple platform projects:
* Domain logic is strictly separated from the UI. UI being strictly a product of state means you can easily construct any state your app can be in. The structure provided by TCA may seem rigid in this case, but I find it makes it very difficult to write bugs or get your UI into an invalid state.
* It has a nice [dependency system](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/dependencymanagement/) that makes it easy to control external dependencies, which in turn make it easy to test your app, via automated or manual tests.
* Units of domain logic, or “features,” are highly composable. It’s easy to throw multiple features in a `NavigationStack` or a `TabView`, or show multiple features within a screen.

The Cape Bocce app has very little test coverage, but were it to be scaled to production, I would add a unit test suite per feature reducer, testing sequences of actions as [recommended in the PointFree docs](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/testing), and add a unit test suite per UI module which tests the UI layer via [snapshot testing](https://github.com/pointfreeco/swift-snapshot-testing).
### SwiftUI
The choice of SwiftUI versus UIKit was easier as I wanted to lean into system UI for most of the app, without fiddling too much. There were definitely a few tvOS-specific learnings, outlined below, but on the whole SwiftUI enabled a sufficient UI with very little code, with some nice animations and customization in select parts. Simply adding a `.tint(.indigo)` [at the root level](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/CapeBocceApp/CapeBocceApp.swift#L14) added a certain personality to the app.

## Learnings
### tvOS Storage Limits
For an app that would be run on a single device, I definitely wanted to avoid any kind of server component to this project. There are 2 data models that require persistence: completed brackets, and a list of recent participants. I figured this persistence would be a good opportunity to leverage a new feature within TCA, the [@Shared macro](https://www.pointfree.co/blog/posts/134-sharing-state-in-the-composable-architecture). Not only does this macro enable data to be easily shared across multiple features of the app (automatically listening for updates and reflecting them in the UI), but the data can be backed by a persistence strategy such as UserDefaults or the file system. So I went about storing my `Bracket` model as JSON in the user’s documents directory, the sensible spot for complex data types like this.

Surprise, a consistent [crash on write](https://stackoverflow.com/questions/58977501/xcode-tvos-error-you-don-t-have-permission-to-save-the-file-filename-txt-in)! Looking up the crash led to [this helpful documentation](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppleTV_PG/index.html#//apple_ref/doc/uid/TP40015241) on tvOS app development oddities:
> The maximum size for a tvOS app bundle 4 GB. Moreover, your app can only access 500 KB of persistent storage that is local to the device (using the [NSUserDefaults](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Classes/NSUserDefaults/Description.html#//apple_ref/occ/cl/NSUserDefaults) class). Outside of this limited local storage, all other data must be purgeable by the operating system when space is low.

A bummer, but in the interest of finding a quick workaround, I checked the average size of my `Bracket` model, and it turns out that 500 KB limit would be plenty for this use case (enough to store ~40 brackets). So I persisted the models as `Data` in UserDefaults via [TCA’s AppStorageKey](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/main/Sources/AppStoragePersistenceKit/PersistenceReaderKey%2BDefaults.swift), which worked like a charm. Were this app to be generalized and put in the App Store, the right answer would be to use iCloud storage for this data.
### Extending the tvOS Focus System
SwiftUI tvOS apps do a nice job of managing user focus by default. With no work at all, if you lay out user-interactive controls on a screen, tvOS will seek out the appropriate control to highlight when the user swipes up, down, left or right. However, the primary screen of the Cape Bocce app, which renders a bracket, required some custom behavior:
* I wanted custom styling for the currently-highlighted match; when focused, a match should highlight the match number in indigo and show a couple extra labels with the participant’s country.
* Matches are not always directly left or right from each other because of how they are laid out - without any manual work, swiping would not always work, and some matches would even be unreachable! We need to make the focus detection system smarter about this view.

I think the end result looks pretty good!
<div style="text-align: center;">
  <video width="640" height="320" controls="controls">
    <source src="https://media.githubusercontent.com/media/nevillco/nevillco.github.io/main/images/CapeBocce/04-focus-state.mp4?raw=true" type="video/mp4">
  </video>
</div>

SwiftUI’s mechanism for managing focus is the [@FocusState](https://developer.apple.com/documentation/swiftui/focusstate) property wrapper. It’s generic over a Hashable type, so you can define your own enum/struct to represent the current focus value (or, omit the type to default to a `Bool`, representing whether `self` is focused). Like other aspects of SwiftUI, it has the shortcoming of coupling logic to UI, as it needs to be declared inside a `View`. So, if you are implementing complex logic around focus - like moving the user focus around programmatically - you can [bind the UI value to a TCA value](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/CreateBracketFormFeatureUI/CreateBracketFormFeatureView.swift#L37), enabling you to even write unit tests around your focus logic.

For the custom styling of the highlighted match, all that was required was [adding a boolean `isFocused`](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/BracketFeatureUI/MatchButtonView.swift#L11) to the match button using `@FocusState` and using it in the view `body`.

Regarding making the focus system “smarter” about finding matches to the left or the right: matches are all contained in `VStack`s, but occasionally with vertical spacing above and below, so what we want is for the `VStack`s themselves to catch focus, and pass it to an eligible child. That’s exactly what SwiftUI’s [`.focusSection()` modifier](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/BracketFeatureUI/MatchButtonView.swift#L80) is for. By adding this 1-liner to a couple different layers of wrapping container views, focus switching starts working as expected, and all matches are reachable, even if there is nothing _directly_ to the left or right.
### Generating a Bracket
As mentioned above, my whole family and I love playing games - enough so that this is not my first software project that modeled a bracket. A bracket contains an [array of matches](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/DataModel/Bracket.swift#L16), each of which has [two participants](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/DataModel/Match.swift#L9-L10). However, each participant could be the winner of a previous match, or the loser of a previous match ([link](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/DataModel/Match.swift#L41C1-L47C2)):
```
public enum MatchParticipant: Codable, Equatable {

    case participant(Participant)
    case awaitingWinner(MatchNumber)
    case awaitingLoser(MatchNumber)

}
```
The tricky part is to stitch them together correctly. The base case, where the number of participants is a power of 2, and the bracket is single-elimination, is pretty easy: every 2 participants play against each other, then you pit the winners against each other recursively:

![An example of an 8-person, single-elimination bracket.](/images/CapeBocce/05-8-person-bracket.png)

However, it might not be obvious how to generate a bracket with 11 participants (during our vacation, we would have somewhere between 8-13 people participating each day), and it’s probably not obvious to most how to construct the “loser’s bracket” for a double elimination tournament. Even worse, there doesn’t seem to be any literature on this topic that I could find on the internet! I would have to reverse-engineer this process. The website challonge.com was the perfect reference: I could create double elimination brackets for each number of participants, screenshot them, observe patterns and begin trying to replicate the bracket structure.

![A, 11-person, double-elimination bracket generated by Challonge.](/images/CapeBocce/06-challonge.png)

The resulting source code can be found [here](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/main/Sources/CreateBracketKit/Bracket%2BGenerate.swift). There are some rigid assumptions, including some force unwraps and a static list of powers of 2, which were always valid for the Cape Bocce app use cases but should be cleaned up. There is also some invented terminology: when the number of participants is not a power of 2, the general idea is to play enough matches to reduce the remaining participants to a power of 2 - for example, with 11 participants, if 3 matches are played, we would be left with 8 participants (4 matches), reducing the problem to a simpler case. This first round of 3 matches is called a “filling round,” and then the following round of 4 matches is called the “first filled round.” If this topic is interesting to you, we have that in common! Some future directions will be discussed at the conclusion of this post.
### Layered tvOS App Icons
I’m no designer, but I wanted to toss on an app icon for this project. I generated an image of some bocce balls on the beach using [Midjourney](https://www.midjourney.com), and went to drop it in the asset catalog, when I found the tvOS app icon asset catalog is structured quite differently than iOS!
![The app icon asset catalog structure for a tvOS app.](/images/CapeBocce/07-asset-catalog-structure.png)

tvOS app icons are composed of multiple _layers_, which allows for this shimmering effect when you focus on the app. I cropped out the bocce balls and put them in a separate layer:
<div style="text-align: center;">
  <video width="640" height="320" controls="controls">
    <source src="https://media.githubusercontent.com/media/nevillco/nevillco.github.io/main/images/CapeBocce/08-app-icon-shimmer.mov?raw=true" type="video/mp4">
  </video>
</div>
I would have ideally content-aware-replaced the bocce balls in the background layer, but didn’t get to it in time.
### The Champion Screen (ChatGPT and Confetti)
With a few days left before heading on vacation, the project was functionally complete, and I had time to add a bit of surprise and delight. Here’s what I ended up with, when a participant wins a tournament:
<div style="text-align: center;">
  <video width="640" height="320" controls="controls">
    <source src="https://media.githubusercontent.com/media/nevillco/nevillco.github.io/main/images/CapeBocce/09-champion-screen.mp4?raw=true" type="video/mp4">
  </video>
</div>

I found this [open source Swift package](https://github.com/simibac/ConfettiSwiftUI) for the confetti animation. The binding-based API was [a little clunky](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/BracketFeatureUI/BracketFeatureView.swift#L48-L57), but it was easy enough to drop in. 

The async-loaded fun fact is a query to ChatGPT whose prompt looks like ([link](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/FunFactDependency/OpenAIQuery.swift)):
> system: You will be provided names of countries. When given the name of a country, look up a fun fact about that country. The fun fact should be interesting, family friendly, and at most 1 short paragraph. Do not include any other text besides the fun fact.

> user: Provide a fun fact for [country name].

This was a fun and surprisingly easy touch: modern `URLSession` with async/await makes this [integration](https://github.com/CleanFoundry/CapeBocce-tvOS/blob/8a27b76135bbaca1ad7f7d00a5e8e945bef1fd6b/Sources/FunFactDependency/FunFactDependency.swift#L23-L46) just 23 lines of code, and wrapping it in TCA’s dependency system means it’s just a function of the shape:
```
(_ countryName: String) async throws -> String
```
One could substitute a mock value, a thrown error, or an infinite-loading function in its place during development. Of course, our usage of the `gpt-4o-mini` model was less than a hundredth of a cent for the week.

## Conclusion
I had a lot of fun working on, and using, this Cape Bocce tvOS app. Architecture investments in Tuist and The Composable Architecture worked out well for velocity, and I learned a few nuances to tvOS development. I don’t aim to put it in the App Store, but if I did, I’d probably fix the following first:
* Rename and rebrand the app as a general “bracket app.”
* Move persistence from UserDefaults to iCloud.
* Make the “choose a country” feature optional, including the fun fact generation, as it’s pretty niche.
* Unit test the bracket generation process, including validating against wider ranges of numbers of participants.
* Add error handling UI for when bracket generation could fail, iCloud operations could fail, and the fun fact could fail.

I do plan on enhancing the `CreateBracketKit` module in a few ways, followed by open sourcing it as a Swift package:
* Add validation with custom error messaging.
* Make `Bracket` and `Participant` protocols so that callers could supply their own implementations.
* Support N-elimination brackets with N participants per match, rather than specifically double-elimination brackets with 2 participants per match.

Thanks for reading! I’ve also left [discussions open](https://github.com/CleanFoundry/CapeBocce-tvOS/discussions) on the GitHub repo for any questions about the code.