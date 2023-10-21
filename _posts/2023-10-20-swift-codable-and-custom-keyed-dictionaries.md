---
title: 'Codable and Custom Keyed Dictionaries'
date: 2023-10-20 22:00:00
excerpt: Swift’s JSON parsing solution, Codable, has its occasional rigid edges. Custom dictionary key types are one of them.
---

[`Codable`](https://developer.apple.com/documentation/swift/codable) is a protocol from the Swift Standard Library that finds its way onto Swift projects of virtually every kind - from iOS apps, to CLIs, to Swift on the Server - because it offers a first-party means of translating between JSON and Swift models. It supports formats other than JSON, but JSON is the primary one: engaging with a web API, using Codable value types as the request and response models, and translating them with `JSONEncoder`+`JSONDecoder`, continues to be a standard pattern across domains.

Codable has been around for a while now (iOS 8+), and like many APIs, it has its areas of high utility and flexibility, and areas of rigidity that can take developers down a rabbit hole. I’ll be highlighting one of those latter areas around dictionaries - specifically, **when a dictionary’s key is a type other than `Int` or `String`**. But first, a quick detour to explain why this situation may come up.

## Codable and Type Safety
Suppose we are working on a social media application. Those aren’t a whole lot of fun without an online component, so we will probably want to communicate with some web API - perhaps we are building our own backend, or communicating with a preexisting, popular social media API. Either way, we can leverage Codable here to send requests and read responses using native Swift values. 

### A Codable Example
Here is an example model from the API, representing a group of users:
```swift
import Foundation
struct SocialGroup: Codable {
    let id: String
    let userIDs: [String]
}
```
Thanks to the Codable conformance, with no added code, we can easily go back and forth between our new struct and its JSON representation:
```swift
// A little wrapper for printing Encodable values
// In production, you’d want to handle the `try!` and
// force unwrap (`!`) more gracefully.
func printJSON<T: Encodable>(_ value: T) {
    let jsonData = try! JSONEncoder().encode(value)
    print(String(data: jsonData, encoding: .utf8)!)
}
let example = SocialGroup(
    id: "GroupID1",
    userIDs: ["UserID1", "UserID2"]
)
printJSON(example)
// ✅ Prints:
// {"userIDs":["UserID1","UserID2"],"id":"GroupID1"}
```

### A Bug Waiting to Happen
But we have one slight problem with our new Codable type. Despite the fact that user IDs and group IDs are semantically completely different things, they are both declared as `String`s, so there is nothing stopping us from writing code like:
```swift
func fetchGroupUser(id: String) -> User {
    // Fetch the user by its ID.
}
// 🚨 Bug - example.id is the ID of a group, not a user!
fetchGroupUser(id: example.id) 
```
Swift has a strong type system with lots of tools, and we can indeed leverage one of them - generics - to prevent this class of bug from occurring at all. 
> In the same way that it is a good practice to write unit tests for bugs, so that developers or users don’t have to find them manually, it is also a good practice to replace those unit tests with compile-time errors where possible, so that developers never have to write or maintain that unit test in the first place.

### The Solution: Phantom Types
The solution to catching this bug at compile time is called a _phantom type_ (I’ve also heard _tagged type_ or _generic wrapper type_). It’s a thin wrapper type around some raw type (`String` in our case, though the concept can be generalized) that uses Swift generics to disambiguate values with the same raw type, but different semantics (`UserID` and `SocialGroupID` in our case). It only takes a few lines of code[^1]:
```swift
struct ID<Tag>: Codable, ExpressibleByStringLiteral {
    public var rawValue: String

    public init(_ rawValue: String) {
        self.rawValue = rawValue
    }
    init(stringLiteral value: String) {
        self.rawValue = value
    }
}
enum UserIDTag { }
enum GroupIDTag { }

typealias UserID = ID<UserIDTag>
typealias GroupID = ID<GroupIDTag>
```
Now, even though they both just wrap a String value, `UserID` and `GroupID` are different types within the type system, and the compiler won’t allow our above bug. Great! We also gave our tagged type `ExpressibleByStringLiteral` conformance, so that a developer manually initializing an ID can do so as `”UserID1”` instead of `ID(rawValue: “UserID1”)` as a convenience. However, we have changed our model’s JSON representation, so unless we can also change the backend, we have likely broken our integration!
```swift
struct SocialGroup: Codable {
    let id: GroupID
    let userIDs: [UserID]
}
let example = SocialGroup(
    id: "GroupID1",
    userIDs: ["UserID1", "UserID2"]
)
printJSON(example)
// 🚨 Bug - this now prints:
// {"userIDs":[{"rawValue":"UserID1"},{"rawValue":"UserID2"}],"id":{"rawValue":"GroupID1"}}
```
Even if we could change the backend, we probably wouldn’t want to agree on this JSON representation - it has a bunch of superfluous, low-value complexity to it. We should fix it by improving `Tag` to have the same JSON representation as its raw String - and luckily, this is a straightforward enough fix by overriding a couple of Codable’s default method implementations:
```swift
extension ID {
    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(self.rawValue)
    }

    init(from decoder: Decoder) throws {
        self.init(try decoder.singleValueContainer().decode(String.self))
    }
}
```
Now our ID type just encodes and decodes its raw value directly, and we’ve added a compile-time check to a tricky class of bugs. There is more to explore on this topic - like other conformances added to our ID type, and supporting raw values other than String - which is why I often end up bringing in [PointFree’s swift-tagged library](https://github.com/pointfreeco/swift-tagged), which provides a mature implementation.

## The Rigid Edge: Dictionary Keys
With this new ID type, and a couple of overridden methods, we can basically use our new ID types anywhere we would previously use a String:
```swift
struct WeaklyTypedExample: Codable {
    let value: String
    let optionalValue: String?
    let array: [String]
    let dictionaryValues: [Int: String]
}
// ✅ has the same JSON representation:
struct StronglyTypedExample: Codable {
    let value: UserID
    let optionalValue: UserID?
    let array: [UserID]
    let dictionaryValues: [Int: UserID]
}
```
With one important edge case, which is **when the phantom type is used as a dictionary key**.
```swift
struct WeaklyTypedExample: Codable {
    let dictionary: [String: Int]
}
let example2 = WeaklyTypedExample(dictionary: [
    "UserID1": 5,
    "UserID2": 10
])
printJSON(example2)
// ✅ Prints:
// {"dictionary":{"UserID2":10,"UserID1":5}}

// To use our phantom type as a dictionary key, we need to conform ID to Hashable. 
// The default implementation will hash our `rawValue`, which works fine here.
extension ID: Hashable { }
struct StronglyTypedExample: Codable {
    let dictionary: [UserID: Int]
}
let example3 = StronglyTypedExample(dictionary: [
    "UserID1": 5,
    "UserID2": 10
])
printJSON(example3)
// 🚨 What? It’s an array of keys and values?
// {"dictionary":["UserID1",5,"UserID2",10]}
```
It’s worth noting, while you can’t just create a heterogeneous array of Strings and Ints like this in Swift, the above print output _is_ valid JSON, and this type will succeed in a "round trip" of encoding/decoding. But why do we get this different representation when our keys are `UserID`s, rather than Strings?

### To the Source
You can find a handful of [threads asking about this behavior](https://forums.swift.org/t/bug-or-pebkac/33796) in various public forums, but the best explainer of this behavior (and why it’s intentional) is [the open source Swift code causing the behavior itself](https://github.com/apple/swift/blob/885dda1338898d9fd6da1c0d7bc569effae39666/stdlib/public/core/Codable.swift#L5352-L5361):
> Encodes the contents of this dictionary into the given encoder.
>   
> If the dictionary uses `String` or `Int` keys, the contents are encoded
> in a keyed container. Otherwise, the contents are encoded as alternating
> key-value pairs in an unkeyed container.

A [few lines down](https://github.com/apple/swift/blob/885dda1338898d9fd6da1c0d7bc569effae39666/stdlib/public/core/Codable.swift#L5378-L5380), our above type will hit this case.
> Keys are Encodable but not Strings or Ints, so we cannot arbitrarily
> convert to keys. We can encode as an array of alternating key-value
> pairs, though.

This makes sense - the implementation only knows our keys are `Encodable` - but arrays are Encodable too, for example, and definitely aren’t valid dictionary keys! Swift can’t know the difference without actually encoding your keys and checking, which would be significantly non-performant to do.

## The Fix: A Property Wrapper
In Swift, when you want to customize the read/write behavior of a particular property, a likely candidate for the job is a [property wrapper](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers). Property wrappers provide a layer of customization that affects how a given property is accessed, whether for `get` or `set` operations. By definition, that also makes them a pretty compatible fit for extending Codable values in a variety of ways[^2]. In our case, the desired behavior is straightforward enough: we want to wrap dictionaries keyed by an ID, and we want to encode/decode them exactly as if they were keyed by the ID’s raw values.
```swift
@propertyWrapper 
struct IDKeyed<Tag, Value: Codable>: Codable {

    var wrappedValue: [ID<Tag>: Value]

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        let untaggedValue = wrappedValue
            .reduce(into: [String: Value]()) { untaggedDict, next in
                untaggedDict[next.key.rawValue] = next.value
            }
        try container.encode(untaggedValue)
    }

    init(from decoder: Decoder) throws {
        let untaggedValue = try decoder.singleValueContainer()
            .decode([String: Value].self)

        self.wrappedValue = untaggedValue.reduce(into: [:]) { taggedDict, next in
            taggedDict[ID(next.key)] = next.value
        }
    }
}
```
Now, once used on our keyed dictionary, encoding and decoding works as expected:
```swift
struct StronglyTypedExample: Codable {
    @IDKeyed var dictionary: [UserID: Int]
}
let example4 = StronglyTypedExample(dictionary: [
    "UserID1": 5,
    "UserID2": 10
])
let example4JSONData = try! JSONEncoder().encode(example3)
print(String(data: example3JSONData, encoding: .utf8)!)
// ✅ prints:
// {"dictionary":{"UserID2":10,"UserID1":5}}
```

### Tradeoffs
While we’ve reached a solution that gives us the type safety and the JSON representation that we want, there are tradeoffs to consider. It’s worth highlighting that this property wrapper works by iterating over the dictionary’s keys in both directions via a `reduce` to go between String and phantom typed keys. **This is very likely 🚨 not performant 🚨 for large dictionaries**. Every codebase is different, and for some, the added type safety provided by using a phantom type is not worth the added complexity in this instance, and go back to using String keys. It’s always worth thinking critically about tradeoffs in your specific context.

One may also find themselves in the situation where they own the server or backend, and sticking with the default encoding behavior as a mixed key+value array works fine for you. Maybe you’re even reusing that default mixed array on both sides because you’re using Swift on the server. Again, tradeoffs are always worth evaluating, but a **word of caution on sticking with this approach**: 
> Dictionaries are un-sorted, which means the default array of mixed keys and values will be arbitrarily-ordered, which means reliably snapshot-testing this JSON becomes impossible.

For String-keyed or Int-keyed dictionaries, one can set `outputFormatting = .sortedKeys` on `JSONEncoder` to produce a stable ordering for snapshot testing, but as noted above, unless our keys are _specifically_ `String.self` or `Int.self`, that `outputFormatting` won’t matter, because our value will be encoded as an (arbitrarily-ordered) array. If we adopt the property wrapper, we encode a String-keyed dictionary under the hood, and our property will respect `outputFormatting = .sortedKeys` once more.

### A Better Fix with iOS 15: `CodingKeyRepresentable`

For deployment targets above iOS 15, rather than adding a property wrapper to each dictionary, you can conform your ID type to [`CodingKeyRepresentable`](https://developer.apple.com/documentation/swift/codingkeyrepresentable) instead. This is preferable because it doesn’t require the additional iteration through dictionary keys, and doesn’t require the developer to remember to apply a property wrapper.

This implementation adds an additional type which plays the role of the [`CodingKey`](https://developer.apple.com/documentation/swift/codingkey), which is the key used for encoding/decoding. We use it here to declare that an ID's `rawValue` is the coding key. Thanks to [Ian Keen](https://gist.github.com/IanKeen/3d226854c8c59a17e151a0022b71f6bb) for this piece.
```swift
struct AnyCodingKey: CodingKey {
    var stringValue: String
    var intValue: Int?

    init?(intValue: Int) {
        self.intValue = intValue
        self.stringValue = "\(intValue)"
    }
    init?(stringValue: String) {
        self.intValue = nil
        self.stringValue = stringValue
    }
    init<T: CodingKey>(_ key: T) {
        self.stringValue = key.stringValue
        self.intValue = key.intValue
    }
    init(_ int: Int) {
        self.init(intValue: int)!
    }
    init(_ string: String) {
        self.init(stringValue: string)!
    }
}
extension ID: CodingKeyRepresentable {
    var codingKey: CodingKey {
        AnyCodingKey(rawValue)
    }

    init?<T: CodingKey>(codingKey: T) {
        self.rawValue = codingKey.stringValue
    }
}

struct CodingKeyRepresentableExample: Codable {
    // No more property wrapper!
    let dictionary: [UserID: Int]
}
let example5 = CodingKeyRepresentableExample(dictionary: [
    "UserID1": 5,
    "UserID2": 10
])
printJSON(example5)
// ✅ prints:
// {"dictionary":{"UserID2":10,"UserID1":5}}
```

The various code snippets in this article can all be [viewed as a gist here](https://gist.github.com/nevillco/6a15b5829cbd104f67affc0dbf7fc2d9), which runs in an Xcode playground.

[^1]: Creating a new `ID` phantom type as seen here would require creating a new empty `enum` to use as the `Tag` namespace, and optionally a typealias if you want to avoid retyping the generic at every callsite, but the new Swift Macros should be able to eliminate this boilerplate. I've seen one such [implementation](https://github.com/FullQueueDeveloper/UniquelyTypedID).
[^2]: Other common use cases for property wrappers that interact with Codable include: parsing API responses that include integers wrapped as strings, providing default Swift values to properties that are optional in JSON, or trying multiple decoding paths before throwing an error.