# Protect mutable state with Swift actors

Presenters:
- Dario Rexin, Swift
- Doug Gregor, Swift

## Data races make concurrency hard

Data races occur when:
- Two threads concurrently access the same data
- One of them is a write

They're notoriously hard to avoid and debug because:
- They require non-local reasoning because the data accesses causing the races
  might be in different parts of the program
- They are non-deterministic, because the operating system scheduler might
  interleave the concurrent tasks in different ways each time you run the program

Data races are caused by shared mutable state. If your data doesn't change, or it
isn't shared across multiple concurrent tasks, you can't have a data race on it.

One way to avoid data races is to eliminate shared mutable state by using value
semantics. With a variable of a value type, all mutation is local. Let properties
of value-semantic types are immutable, so it's safe to access them from different
concurrent tasks.

Sometimes there are still cases where shared mutable state is required.

## Shared mutable state in concurrent programs

Shared mutable state requires synchronization.

Various synchronization primitives exist:
- Atomics
- Locks
- Serial dispatch queues

## Actors

Actors provide synchronization for shared mutable state. Actors _isolate_ their
state from the rest of the program.
- All access to that state goes through the actor
- The actor ensures mutually-exclusive access to its state

### Actor types

- Similar capabilities to structs, enums, and classes
- Reference type
- Synchronization and data isolation set actors apart

For example:

```
actor Counter {
  var value = 0

  func increment() -> Int {
    value = value + 1
    return value
  }
}
```

Making the Counter an actor type eliminates concurrent access. The increment
method will run to completion without any other code executing on the actor.
This guarantee eliminates data races on the actor's state.

When you interact with the actor from outside, do it async. This enables the code
to "wait" until the actor is available.

### Actor reentrancy

Example:

```
// Image download cache
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    // Check the cache
    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        // Download the image
        let image = try await downloadImage(from: url)


        // Store the downloaded image to the cache before returning
        cache[url] = image
        return image
    }
}
```

Because we're in an actor, the code is free from low-level data races. Any
number of images can be downloaded concurrently. Actor synchronization ensures
that only 1 task can execute code that accesses the cache instance property at
a time, so there's no way the cache can be corrupted.

Whenever an `await` occurs, it means the function can be suspended. At the point
when the function resumes, the overall program state will have changed.

Check assumptions after an `await`. Subtle bugs could creep in when you have
suspended functions.

To design well for actor reentrancy:
- Perform mutation in synchronous code
- Expect that the actor state could change during suspension
- Check your assumptions after an `await`

### Actor isolation

#### Protocol conformance

Conforming to various protocols must respect actor isolation

Example of Equatable (Works):

```
actor LibraryAccount {
  let idNumber: Int
  var booksOnLoan: [Book] = []
}

extension LibraryAccount: Equatable {
  static func ==(lhs: LibraryAccount, rhs: LibraryAccount) -> Bool {
    lhs.idNumber == rhs.idNumber
  }
}
```

This static equality method has no self instance because it's static, so it's
not isolated to the actor. Instead, we have two parameters of actor type, and
this static method is outside of both of them. But that's ok, because this method
is only accessing immutable state on the actor.

Example of Hashable (Causes an error):

```
actor LibraryAccount {
  let idNumber: Int
  var booksOnLoan: [Book] = []
}

extension LibraryAccount: Hashable {
  func hash(into hasher: inout Hasher) {
    hasher.combine(idNumber)
  }
}
```

This Hashable implementation causes a compiler error. This function could be
called from outside the actor, but hash into is not async, so there's no way to
maintain actor isolation.

Instead, make the method non-isolated:

```
actor LibraryAccount {
  let idNumber: Int
  var booksOnLoan: [Book] = []
}

extension LibraryAccount: Hashable {
  nonisolated func hash(into hasher: inout Hasher) {
    hasher.combine(idNumber)
  }
}
```

Non-isolated means that this method is treated as being outside the actor, even
though it is syntactically described on the actor. This means it can satisfy the
synchronous requirement from the Hashable protocol.

Because non-isolated methods are treated as being outside the actor, they cannot
reference mutable state on the actor. This method is fine, because it's referring
to the immutable `idNumber`. If we tried to hash based on something else, such as
the array of `booksOnLoan`, we'd get an error, because access to mutable state
from the outside would permit data races.

#### Closures

"Closures are little functions that are defined within one function, that can
then be passed to another function to be called sometime later."

Like functions, a closure might be actor-isolated, or it might be non-isolated.

In this example, we'll read some from each book we have on loan, and return the
total number of pages we've read. The call to reduce involves a closure that
performs the reading:

```
extension LibraryAccount {
  func readSome(_ book: Book) -> Int { ... }

  func read() -> Int {
    booksOnLoan.reduce(0) { book in
      readSome(book)
    }
  }
}
```

Note there's no `await` in this call to `readSome()`. That's
because this closure, which is formed within the actor-isolated function `read()`,
is itself actor-isolated. We know this is safe because the reduce operation is
going to execute synchronously, and can't escape the closure out to some other
thread where it could cause concurrent access.

In an example where you want to read some books later:

```
extension LibraryAccount {
  func read() -> Int { ... }

  func readLater() {
    asyncDetached {
      await read()
    }
  }
}
```

This detached task executes the closure concurrently with other work that the
actor is doing. Therefore, the closure can't be on the actor, or we'd introduce
data races. This closure is not isolated to the actor. When it wants to call the
`read()` method, it must do so asynchronously, as indicated by the `await`.

#### Actor isolation and data

In these library examples, we haven't examined what the `Book` type actually is.

```
actor LibraryAccount {
  let idNumber: Int
  var booksOnLoan: [Book] = []
}

struct Book {
  var title: String
  var authors: [Author]
}
```

A value type, like a struct, is a good choice, because it means all the state for
an instance of the `LibraryAccount` actor is self-contained.

If we call this method to get a random book to read:

```
func visit(_ account: LibraryAccount) async {
  guard var book = await account.selectRandomBook() else {
    return
  }
  book.title = "\(book.title)!!!"
}
```

We get a copy of the book that we can read. Changes that we make to our copy of
the book won't affect the actor, and vice versa.

However, if we turn the book into a class, things are a bit different:

```
actor LibraryAccount {
  let idNumber: Int
  var booksOnLoan: [Book] = []
  func selectRandomBook() -> Book? { ... }
}

class Book {
  var title: String
  var authors: [Author]
}
```

Our `LibraryAccount` actor now references instances of the Book class. That's
not a problem in itself. But what happens when we call the method to select a
random book? Now we have a reference into the mutable state of the actor, which
has been shared outside the actor. We've created the potential for data races.

Value types and actors are both safe to use concurrently, but classes can still
pose problems.

The name for types that are safe to use concurrently is `Sendable`.

A `Sendable` type is safe to share concurrently. If you copy a value from one
place to another, and both places can safely modify their own copies of that
value without interfering with each other, the type can be `Sendable`.

Many different kinds of types are `Sendable`:
- Value types
- Actor types
- Immutable classes
- Internally-synchronized class
- `@Sendable` function types

Most classes are neither immutable nor internally-synchronized, so they're not
`Sendable`.

Functions aren't necessarily `Sendable`. There's a new kind of function type
for functions that are safe to pass across actors: `@Sendable`

#### Checking Sendable

`Sendable` describes a common, but not universal, property of types. Swift
will eventually prevent non-`Sendable` types from being shared across actors.

Check `Sendable` by adding a protocol conformance:

```
struct Book: Sendable {
  var title: String
  var authors: [Author]
}
```

In the above example, if `Author` was a class, you'd get an error:
`error: Sendable type 'Book' has non-Sendable stored property 'authors' of type '[Author]'`

You can propogate `Sendable` by adding a conditional conformance:

```
struct Pair<T, U> {
  var first: T
  var second: U
}

extension Pair: Sendable where T: Sendable, U: Sendable {
}
```

In this example, the struct is only `Sendable` when both of its generic arguments
are `Sendable`.

The same approach can be used to conclude that an array of `Sendable` types is,
itself, `Sendable`.

#### Sendable functions

`@Sendable` function types conform to the `Sendable` protocol

`@Sendable` places restrictions on closures:
- No mutable captures
- Captures must be of `Sendable` type
- Cannot be both synchronous and actor-isolated

We've actually been relying on the idea of `@Sendable` closures in this talk.
The operation that creates detached tasks takes a Sendable function, written
here with `@Sendable` in the function type:

```
func asyncDetached<T>(_ operation: @Sendable () async -> T) -> Task.Handle<T, Never>
```

### Main actor

The main thread is important for apps:
- UI rendering
- Main run loop to processing events

But you don't want to do all of your work on the main thread. If you do too much
work on the main thread, for example because you have a slow input/output
interaction, or a blocking interaction with the server, your UI will freeze. So
you need to do work on the main thread when it interacts with the UI, but get off
the main thread quickly for computationally-expensive or long-waiting operations.

Do work off the main thread when you can, and then call `DispatchQueue.main.async`
to perform updates on the main thread.

For example:

```
func checkedOut(_ booksOnLoan: [Book]) {
  booksView.checkedOutBooks = booksOnLoan
}

DispatchQueue.main.async {
  checkedOut(booksOnLoan)
}
```

Interacting with the main thread is a lot like working with an actor. If you're
already running on the main thread, you can safely access and update your UI state.
If you aren't running on the main thread, you need to interact with it async. This
is exactly how actors work.

There's a special actor to describe the main thread which we call the main actor.

- Actor that represents the main thread
- Performs all of its synchronization through the main DispatchQueue. This means
  that from a runtime perspective, the main actor is interchangeable with using
  `DispatchQueue.main`

With Swift concurrency, you can mark a declaration with the `@MainActor` attribute
to say that it must be executed on the main actor. For example:

```
@MainActor func checkedOut(_ booksOnLoan: [Book]) {
  booksView.checkedOutBooks = booksOnLoan
}

await checkedOut(booksOnLoan)
```

If you call this from outside the main actor, you need to `await` so the call can
be performed asynchronously.

Types can be placed on the main actor:
- Implies that all methods and properties of the type are `MainActor`
- Opt out individual members with nonisolated

## Summary

- Use actors to synchronize access to mutable state
- Design for actor reentrancy
- Prefer value types and actors to eliminate data races
- Leverage the main actor to protect UI interactions

See also:
- Swift concurrency: Update a sample app
- Swift concurrency: Behind the scenes
