---
title: Swift_Server_Data
author: Pat Nakajima
publishedAt: 05/15/2024
excerpt: I mean why not?
tags: swift, swift server, probably a bad idea
---

<small style="text-align: center; display: block;"><em>This is the first of what I'm sure will be a few posts detailing <a href="https://github.com/nakajima/ServerData.swift">a fun lil thing I'm trying to do</a>.</em></small>

I was working on an iOS thing for fun and decided it probably wanted a server component. My first thought was oops. Then it was, "hmm what do I write this in?" I landed on Swift because I like Swift[^1]. I landed on MySQL because I know it.

I spun up a little [Hummingbird](https://github.com/hummingbird-project/hummingbird) server then started looking at persistence options. I found a bunch of great options but none really seemed to work the way I wanted and I thought I could just put something together quickly (oops). Cut to me writing my own.

After using SwiftData in the iOS side, I liked a lot about how it felt to use. I didn't super know how it worked besides something something macros something something predicates. So I thought it'd be fun to make a version of it for the server.

Servers are different than clients though. So here are my sort of high level goals:

#### Be Safe

I want to be Sendable. When I go, I want everyone to talk about me like "wow pat was so sendable." So I wanted to use structs because they make that easier.

Another thing I wanted was type-safe queries. I know generating queries is _fraught_ but so is me writing SQL. As long as there's an escape hatch for writing raw queries, I think we should be good.

#### Not worry about observation

A lot of what makes SwiftData nice on the client is that updates to the store can update views easily[-ish](/posts/2-live-model/). On the server I don't care about that, which is nice because a _downside_ of SwiftData is that all your model objects are classes. Reference semantics can make sense for models but after using stuff like [GRDB](https://github.com/groue/GRDB.swift) and playing with [Blackbird](https://github.com/marcoarment/Blackbird), I think using structs is nicer.

#### Automigrations

While I'm prototyping it's nice to be able to teardown the DB and make it from scratch, handling changes to the data model in the schema. SwiftData on the client lets me do this and I wanted to be able to do it in cloud[^2].

And the final goal is:

#### Learn how stuff works

I watched [the WWDC](https://developer.apple.com/videos/play/wwdc2023/10166/) talks [about macros](https://developer.apple.com/videos/play/wwdc2023/10167/) but I had never tried to implement one. Was I scared off by talk of how they increased compilation times? Sure, but as is evident by my embarking on this project, I've got plenty of time apparently.

### `INSERT INTO blog_posts (title) VALUES ("hello world");`

This whole thing started with [SQLKit](https://github.com/vapor/sql-kit), which seemed to be at a nice level of abstraction for me without pulling too much else in. It's a nice wrapper around general SQL databases that has adapters for MySQL, Postgres and SQLite that doesn't try to do too much but handles stuff like:

- DB connections
- Basic querying
- Being able to encode/decode stuff into SQL

At first I was just doing stuff with SQLKit and tbh, I probably didn't need to write a wrapper. But where's the fun in that?

### `@Column var usePropertyWrapper = false`

My next step was to figure out how to model models. I wanted to be able to annotate my model's properties with DB specific stuff like constraints. At first I tried using a [`propertyWrapper`](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers) for this, but I had to bail for a few reasons.

First, property wrappers don't know the name of the property they're wrapping. You can _sort of_ get around this with `Mirror` trickery, but it felt clunky.

I want model translation to the DB to be done via `Codable`. But making a wrapped property `Codable` means making the property wrapper `Codable`. That means implementing `init(from: any Decoder)` and you don't have access to the normal property wrapper `init` from there. What am I talking about? Let's say you've got a property wrapper:

```swift
@propertyWrapper struct ColumnWrapper<Value: ColumnValue> {
	// We can just store stuff here since we don't actually need to do
	// anything with getting/setting. We just want a place to stash some
	// additional data about a property.
	var wrappedValue: Value

	// This is where we'd store stuff like the raw SQL type, constraints,
	// or a custom column name.
	var constraints: [String]

	init(constraints: [String] = []) {
		// Let's just assume we can set a default value so we can have a pretty
		// init that lets us just call @ColumnWrapper(constraints: [...])
		// instead of having to pass a wrappedValue every time.
		//
		// This isn't actually how anything works but just go with me here.
		self.wrappedValue = Value.defaultValue
		self.constraints = constraints
	}
}
```

And we want to use it like so:

```swift
struct SomeModel {
	@ColumnWrapper(constraints: ["primaryKey"]) var id: Int?
}
```

Our `ColumnWrapper`'s constraints would then be `["primaryKey"]`. Awesome. Except when want to make `SomeModel` decodable, which means that `ColumnWrapper` needs to be decodable as well. We need to add this to our wrapper to be conformant:

```swift
init(from decoder: any Decoder) throws {
	self.wrappedValue = try decoder.singleValueContainer().decode(Value.self)

	// oh no we can't set constraints here.......
}
```

We could possibly store metadata about properties in some sort of global storage, but that didn't feel great to me. Neither did switching the model to a class to be able to [access its enclosing instance](https://www.swiftbysundell.com/articles/accessing-a-swift-property-wrappers-enclosing-instance/). And even if it did, I don't think it'd be great to have every instance have to carry around a definition of its own columns.

So anyway, property wrappers were out. What next?

## `public macro Model() = ?`

I wrote a `@Model` macro that takes a table name that represents where the model's records are stored in the DB[^3]. Then it grabs their properties and stores the following:

- The property's name
- The property's Swift type (so we can try to figure out how it wants to be stored in the DB)
- Whether or not the property is optional (so we can add `NOT NULL` constraints in the right places)

To annotate properties with additional info, I added a `@Column` wrapper that actually doesn't expand to anything, it's just there so that `@Model` has a place to look for more information like:

- An explicit SQL type that the property wants to be stored as
- Any additional [constraints supported by SQLKit](https://github.com/vapor/sql-kit/blob/main/Sources/SQLKit/Expressions/Clauses/SQLColumnConstraintAlgorithm.swift) that should be applied to the property.

I thought about using a property wrapper instead of a macro that does nothing, but felt like it's nice that with the macro, the properties are just normal Swift properties, there's no indirection or `Codable` trickery required.

---

Ok, so now we've got a model that looks like this:

```swift !image!
@Model(table: "people") struct Person {
	// We assume this is the primary key since it's named `id`
	// so it gets a PRIMARY KEY AUTO_INCREMENT. Could maybe this
	// configurable at some point…
	var id: Int?

	// Adds a `NOT NULL` to the `age` column
	var age: Int

	// Adds a unique index (courtesy of SQLKit)
	@Column(.unique) var name: String

	// We can store this string as a blob for some reason
	@Column(type: .blob) var about: String?
}
```

The macro expands into an extension that looks like this:

```swift
extension Person: StorableModel {
	static let _$table = "people"
	static var _$columnsByKeyPath: [AnyHashable: ColumnDefinition] {
		[
			\Person.id: ColumnDefinition(name: "id", sqlType: nil, swiftType: Int.self, isOptional: true, constraints: []),
			\Person.age: ColumnDefinition(name: "age", sqlType: nil, swiftType: Int.self, isOptional: false, constraints: []),
			\Person.name: ColumnDefinition(name: "name", sqlType: nil, swiftType: String.self, isOptional: false, constraints: [.unique]),
			\Person.about: ColumnDefinition(name: "about", sqlType: .blob, swiftType: String.self, isOptional: true, constraints: [])
		]
	}
}
```

It's a dictionary where the keys are the key path for the property (this makes it easy to look up information about a column from places we might need it) and the values are `ColumnDefinition` objects that contain everything that was defined in `@Column`.

You might notice that the `id` column doesn't have the primary key constraint specified. That's because I wanted to keep the macro as small as possible, just pulling stuff out of the source. The logic about what SQL types get inferred from Swift types happens elsewhere.

--- SECTION

Now we've got enough information to automatically create database tables. How does that work? Well I feel like this post is already pretty long so I think I'm gonna do that in another one.

Stay tuned for some of the nuts and bolts of talking to SQLKit and my journey into the center of the predicate.

---

[^1]: I was a Rails developer for like fifteen years so it probably made sense to use Rails. But I really dig Swift and this project isn't really **for** anything so I thought I'd give Swift a shot. Swift on the server has a bunch of great stuff these days. <br/>It's just fun you know?
[^2]: A little beelink server I have running in my office.
[^3]: I probably could have derived the table name from the name but I like plural table names and I didn't want to handle inflection.
