---
title: Swift_Server_Data: Talking to the database
author: Pat Nakajima
publishedAt: 05/17/2024
excerpt: Wrapping SQLKit for fun and absolutely no profit
tags: swift, swift server, probably a bad idea
---

<small style="text-align: center; display: block;"><em>This is part 2 of a series of posts about writing a SwiftData-esque server side framework for fun.<br/>You can <a href="https://patstechweblog.com/posts/3-swift-server-data">read part 1 here</a> or <a href="https://github.com/nakajima/ServerData.swift">check it out on GitHub</a>.</em></small>

Previously on patstechweblog: I was flailing my way through Swift macros in order inspect my model's properties. These [macros let me create column definitions](https://github.com/nakajima/ServerData.swift/blob/main/Sources/ServerDataMacros/ModelMacro.swift) that look like this:

```swift
// Contains information about a column. This gets populated from the
// @Model macro, along with some metadata from the @Column macro which
// doesn't do anything besides sit there waiting to be parsed by swift-syntax.
public struct ColumnDefinition: Sendable {
	public var name: String
	public var sqlType: SQLDataType?
	public var swiftType: Any.Type
	public var isOptional: Bool
	public var constraints: [SQLColumnConstraintAlgorithm]
}
```

The macro stores these on the model as a static property called `_$columnsByKeyPath`. So how do we turn those into a database table? With SQLKit:

```swift
// The @Model macro adds conformance to StorableModel
public extension StorableModel {
	// This is defined to get around macro expansion ordering issues. We should never
	// see this actually happen.
	static var _$table: String { fatalError("should have been expanded by the macro") }

	static func create(in database: any SQLDatabase) async throws {
		// Create a SQLKit `SQLCreateTableBuilder` that we'll use to create the table.
		var creator = database.create(table: _$table)

  	// Go through each of the columns that we got from the @Model macro
		for column in _$columnsByKeyPath.values {
			// Copy all the constraints that were defined with the @Column macro
			var constraints = column.constraints

			// If the property isn't optional, add a not null constraint into the DB as well
			if !column.isOptional {
				constraints.append(.notNull)
			}

			// Just assume that the primary key is called `id`. Works well enough in Rails.
			if column.name == "id" {
				constraints.append(.primaryKey(autoIncrement: true))
			}

			// If we specified an explicit SQL type with @Column, use that for the DB.
      // Otherwise, check to see what the column looks like it could be stored as.
			let type: SQLDataType = if let sqlType = column.sqlType {
				sqlType
			} else {
				switch column.swiftType {
				case is any StorableAsInt.Type: .bigint
				case is any StorableAsDouble.Type: .real
				case is any StorableAsText.Type: .text
				case is any StorableAsData.Type: .blob
				case is any StorableAsDate.Type: .custom(SQLRaw("DATETIME"))
				default:
					fatalError("cannot represent: \(column.swiftType)")
				}
			}

			// Add the column to the builder.
			creator = creator.column(column.name, type: type, constraints)
		}

		// When we're done, create the table!
		try! await creator.run()
	}
}
```

SQLKit's [`SQLCreateTableBuilder`](https://github.com/vapor/sql-kit/blob/main/Sources/SQLKit/Builders/Implementations/SQLCreateTableBuilder.swift) is doing the heavy lifting here. It abstracts the database-specific statements away into [dialects](https://github.com/vapor/sql-kit/blob/main/Sources/SQLKit/Database/SQLDialect.swift), which can be MySQL, Postgres or SQLite. How do we tell ServeData about database? That's the job of the Container. The Container wraps our specific database adapter and makes it available to ServerData. 

Here's how we set one up for MySQL:


```swift
let databaseName = "cool_development_database"
var configuration = MySQLConfiguration(
	url: ProcessInfo.processInfo.environment["MYSQL_URL"]!
)!

configuration.database = databaseName

let source = MySQLConnectionSource(configuration: configuration)
// Ostensibly you've have another way of getting an event loop group in an actual app,
// but this is how I do it in tests.
let pool = EventLoopGroupConnectionPool(
	source: source,
	on: MultiThreadedEventLoopGroup(numberOfThreads: 2)
)

let mysql = pool.database(logger: Logger(label: "test"))

let container = try Container(
	name: databaseName,
	database: mysql.sql(),
	shutdown: { pool.shutdown() } // Called on Container.deinit
)
```

So what can we do with the Container? All sorts of stuff really. Like create a PersistentStore for a model:

```swift
let store = PersistentStore(for: Person.self, container: container)

// Create the database table if it doesn't exist already
await store.setup()
```

Now that we have a PersistentStore, we can start saving records:

```swift
// Insert a person into the people table
let person = Person(age: 40, name: "Pat")
try await store.save(person)

// Insert a person into the people table and set its ID to
// the new record's ID
var person = Person(age: 40, name: "Pat")
try await store.save(&person)

// Insert a bunch of people into the people table
var people = [Person(age: 40, name: "Pat"),	Person(age: 50, name: "Not Pat")]
try await store.save(people)
```

Believe it or not, we can also list records:

```swift
let people = try await store.list()
```

We can use a Swift `Predicate` that generates SQL to add conditions as well (the following examples actually escape the values but I've removed that for simplicity here):

```swift
// SELECT * FROM people WHERE `name` = "Pat"
let people = try await store.list(where: #Predicate {
	$0.name == "Pat"
})

// SELECT * FROM people WHERE COALESCE(`id`, -1) > 0
let people = try await store.list(where: #Predicate {
	($0.id ?? -1) > 0
})

// SELECT * FROM people WHERE `age` > 30 AND `name` = "Pat" OR `name` = "Pat"
let people = try await store.list(where: #Predicate {
	$0.age > 30 && ($0.name == "Pat" || $0.name == "Not Pat")
})
```

Obviously it doesn't make sense to use a Predicate for all cases, but it can be nice while prototyping to have type-safe, SQL-escaped queries. We can also just use SQLKit directly if we want to query the DB with our own SQL.

---

So that's the basic in and outs of getting things in and out of the database. Next time we'll talk about how the Predicate stuff works. The `#Predicate` macro in SwiftData is neat and weird and magical and it was fun to learn how it works.