---
title: `@LiveModel` in SwiftData
author: Pat Nakajima
publishedAt: 05/06/2024
excerpt: Keep a `SwiftData` model up to date.
tags: swiftui, swiftdata, swift, good or bad idea?
---

Ok, so you've got a `SwiftUI` view that displays a `SwiftData` record. Maybe the model looks like this:

```swift
@Model final class Person {
	var name: String
	var friendCount: Int = 0

	init(name: String) {
		self.name = name
	}
}
```

The view maybe looks like this:

```swift image
struct PersonView: View {
	var person: Person

	var body: some View {
		HStack {
			Text(person.name)
			Text("\(person.friendCount) friends")
		}
	}
}
```

If updates happen in our container's `mainContext`, the view will be updated. But what if an update happens in a background context? Like this?

```swift
Task {
	// Use a background task to update the person's friendCount
	let context = ModelContext(container)

  // Assume we've got a `personID` from somewhere
	let person: Person = context.registeredModel(for: personID)!

  // Update the friend count
	person.friendCount += 1

  // Save the background context
	try! context.save()
}
```

The view won't be updated and we'll look like fools.

Unfortunately there's not really a way (afaik) to be updated when a **Swift**Data store changes, but there **is** a way to do it with **Core**Data. Read [this post for more details](https://fatbobman.com/en/posts/use-core-data-features-in-swiftdata-by-swiftdatakit/). Or check out [fatbobman/SwiftDataKit](https://github.com/fatbobman/SwiftDataKit) to be able to hook into what that blog post describes. Or keep reading to see how I'm using it. Or go have a cup of coffee, pet a cat, do whatever you want.

## The `@LiveModel` property wrapper

Let's use some stuff from SwiftDataKit to implement a property wrapper that keeps our model up to date, no matter where updates happen from. (This is something [marcoarment/Blackbird](https://github.com/marcoarment/Blackbird/blob/076827d5be06c3a1cf686b2012e8f3853cba7b38/Sources/Blackbird/BlackbirdSwiftUI.swift#L325) has and I thought was nice).

```swift
import Combine
import SwiftData
import SwiftDataKit
import NotificationCenter
import CoreData

// Keep a SwiftData record up to date
@propertyWrapper @Observable public final class LiveModel<T: PersistentModel> {
	var _model: T

	@MainActor public var wrappedValue: T {
		get { _model }
		set { _model = newValue }
	}

	var cancellable: AnyCancellable?

	@MainActor public init(wrappedValue: T) {
		self._model = wrappedValue

		if let context = wrappedValue.modelContext {
			self.cancellable = NotificationCenter.default.publisher(for: .NSManagedObjectContextDidSave).sink { [weak self] notification in
				guard let userInfo = notification.userInfo, let self else {
					return
				}

				if let updated = userInfo["updated"],
					 // Convert to an actual swift set
					 let set = (updated as? NSSet as? Set<NSManagedObject>),
					 // See if this update is for our model
					 let object = set.first(where: { $0.objectID.persistentIdentifier == self._model.id }),
					 // We know we have a persistent identifier because of the above check, so try to reload
					 // our model from its context.
					 let model: T = context.registeredModel(for: object.objectID.persistentIdentifier!)
				{
					// Update our model, so the Observation system can let the view know.
					self._model = model
				}
			}
		}
	}

	deinit {
		cancellable?.cancel()
	}
}
```

Now our model updates whenever it changes, even from a child context. I guess it'd be better if `SwiftData` did this automatically and maybe the next version will, but for now I've found this useful. Maybe you will too? Maybe it's a terrible idea? Is there some way to do this without reaching into CoreData? Lemme know.

**You can check out a demo of @LiveModel here: [https://github.com/nakajima/LiveModelDemo](https://github.com/nakajima/LiveModelDemo).**

```swift !image! !hidden!
@LiveModel var person: Person
```
