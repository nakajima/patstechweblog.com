---
title: `SomeDecodable`
author: Pat Nakajima
publishedAt: 05/04/2024
excerpt: Sometimes you want to parse JSON that doesn't really agree with Codable.
tags: codable, json, good or bad idea?, swift
---

Sometimes you want to parse JSON that doesn't really agree with Codable’s synthesized init. Like if a key wants to be a single thing **or** an array of that thing. What am I talking about? Ok let's say you want to parse some JSON into this:

```swift
struct BlogPost: Codable {
  let author: String
}
```

So the JSON you get can look like this:

```json
{
  "author": "pat"
}
```

But _TWIST_, sometimes it gives you this:

```json
{
  "author": ["pat", "evil pat"]
}
```

With `Codable` that usually means you have to write your own `init(from decoder: any Decoder)`. And for me that means I usually have to google how to do that again.

Well now in this very specific case, I don't anymore because I'm using this:

```swift
public enum SomeDecodable<T: Decodable>: Decodable {
	case none, one(T), many([T])

	public init(from decoder: any Decoder) throws {
		let container = try! decoder.singleValueContainer()

		if let one = try? container.decode(T.self) {
			self = .one(one)
		} else if let many = try? container.decode([T].self) {
			self = .many(many)
		} else {
			self = .none
		}
	}
}

// SomeDecodable can have some Equatable, as a treat.
extension SomeDecodable: Equatable where T: Equatable { }
```

Now, in our model, we can use it:

```swift
struct BlogPost {
  let author: SomeDecodable<String>
}

let post = try JSONDecoder().decode(BlogPost.self, from: json)
switch post.author {
case let .one(author):
  print("The author is named \(author)")
case let .many(authors):
  print("The authors are named \(authors.joined(separator: ", "))")
default:
  print("I have no idea who the authors are")
}
```

With [parameter pack iteration](https://www.swift.org/blog/pack-iteration/) in Swift 6.0, we might be able to get rid of the enum and support more generic cases, but I haven't figured out how to do it with Swift 5.10 yet.
