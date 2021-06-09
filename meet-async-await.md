# Meet Async/Await in Swift

Presenters:
- Nate Chandler, Swift Compiler Engineer
- Robert Widmann, Swift Compiler Engineer

Async/await in Swift makes it easier to write asynchronous code. Your code will
better reflect your ideas, and will be safer, too. The SDK has hundreds of
awaitable methods available to use.

Things that can be `async` in Swift's new
concurrency model:

- Functions
- Properties
- Initializers

## Functions: synchronous and asynchronous
Thread is blocked when you run a synchronous function. For example, when you
fetch a thumbnail and prepare a thumbnail image, the thread is blocked until
that is complete. When you use the completionHandler version of the func, you
can do other things on the thread while waiting for the results of the func. The
completion handler notifies you when the func is done.

Function before async/await using completion handlers:

Call the function with two arguments: a String, the input to the first operation,
and a completion handler, used to give the output back to the caller.

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void)
```

When `fetchThumbnail` is called, we first call `thumbnailURLRequest`:

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  // dataTask(with:completion:)

  // UIImage(data:)

  // prepareThumbnail(of:completionHandler:)


}
```

On the thread at this point is:
`fetchThumbnail` -> `thumbnailURLRequest` -> `fetchThumbnail`

Because it's synchronous, it doesn't need a completion handler.

Next, we call `dataTask` on the shared `URLSession` instance:

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in

    // UIImage(data:)

    // prepareThumbnail(of:completionHandler:)

  }
  task.resume()
}
```

On the thread at this point is:
`fetchThumbnail` -> `dataTask`

It synchronously produces a `URLSession` `dataTask` which must be resumed, to
kick off the asynchronous work.

`fetchThumbnail` then returns, and the thread is free to do other work.

Eventually, either the image finishes downloading, or something goes wrong. Either
way, the request completes. If something does go wrong, we need to invoke the
completion handler and pass the error along.

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in
      if let error = error {
          completion(nil, error)
      } else if (response as? HTTPURLResponse)?.statusCode != 200 {
          completion(nil, FetchError.badID)
      } else {
          // UIImage(data:)

          // prepareThumbnail(of:completionHandler:)
      }
  }
  task.resume()
}
```

If everything works out, we create an image from the from the data using UIImage's
init with data.

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in
      if let error = error {
          completion(nil, error)
      } else if (response as? HTTPURLResponse)?.statusCode != 200 {
          completion(nil, FetchError.badID)
      } else {
          guard let image = UIImage(data: data!) else {
              return
          }
          // prepareThumbnail(of:completionHandler:)
      }
  }
  task.resume()
}
```

If no image is produced, we're done. If an image is produced, finally we call
UIKit's method `prepareThumbnail` on it and pass a completion handler.

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in
      if let error = error {
          completion(nil, error)
      } else if (response as? HTTPURLResponse)?.statusCode != 200 {
          completion(nil, FetchError.badID)
      } else {
          guard let image = UIImage(data: data!) else {
              return
          }
          image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
              guard let thumbnail = thumbnail else {
                  return
              }
              completion(thumbnail, nil)
          }
      }
  }
  task.resume()
}
```

But there's a problem. `fetchThumbnail`'s caller expects to be notified when
`fetchThumbnail` finishes its work, even if it fails. We're not doing that.

Developer writing this function is so used to writing `guard` `else` `return` that
he forgot to invoke the completion handler - twice. If creating a UIImage from
data or preparing a thumbnail fails, the caller will never be notified and it'll
just throw a spinner forever. Every path through the function should notify.

To do that, we need to invoke completion handler if the error occurs, and pass
the error along:

```
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumbnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in
      if let error = error {
          completion(nil, error)
      } else if (response as? HTTPURLResponse)?.statusCode != 200 {
          completion(nil, FetchError.badID)
      } else {
          guard let image = UIImage(data: data!) else {
              completion(nil, FetchError.badImage)
              return
          }
          image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
              guard let thumbnail = thumbnail else {
                  completion(nil, FetchError.badImage)
                  return
              }
              completion(thumbnail, nil)
          }
      }
  }
  task.resume()
}
```

We can't use Swift's usual throwing mechanism here. This means Swift can't check
our work. For Swift, it's just a closure. We want to make sure it's always invoked,
but Swift doesn't have any way of doing that. Another developer had to manually
point out this was an issue.

This code is verbose, and has a lot of opportunities for subtle bugs. It's not
simple, easy, and safe. With async/await, we can do better.

Async/await version of the function:

```
func fetchThumbnail(for id: String) async throws -> UIImage {

}
```

Marking a function `async` allows it and its signature to be simpler. If an image
is thumbnailed successfully, that thumbnail is returned. If an error is encountered,
the error is returned.

```
func fetchThumbnail(for id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id)
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {throw FetchError.badID}
    // UIImage(data:)
    // thumbnail
}
```

In this example, `try` is needed because this `throws`, and `await` is needed
because it's `async`.

Next, `fetchThumbnail` will try to create an image from the data it downloaded.
If that succeeds, a thumbnail will be rendered for the image by accessing its
thumbnail property.

```
func fetchThumbnail(for id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id)
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {throw FetchError.badID}
    let maybeImage = UIImage(data: data)
    guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
    return thumbnail
}
```

This code is safer, shorter, and better reflects the intent.

In the second-to-last line, even though there's no function call, the expression
that kicks off rendering the thumbnail is marked with `await`. That's because
the thumbnail property is `async`.

## Async properties

Example async property:

```
extension UIImage {
  var thumbnail: UIImage? {
    get async {
      let size = CGSize(width: 40, height: 40)
      return await self.byPreparingThumbnail(ofSize: size)
    }
  }
}
```

1. Has an explicit getter. This is required to mark a property async. As of Swift
   5.5, property getters can also throw. Like with async function signatures, if
   a property is both async and throws, the `async` keyword goes right before
   `throws`.
2. The property has no setter. Only read-only properties can be async.

## Async sequences

`await` works in `for` loops!

Example async sequence:

```
for await id in staticImageIDsURL.lines {
  let thumbnail = await fetchThumbnail(for: id)
  collage.add(thumbnail)
}
let result = await collage.draw()
```

To learn more about async sequence, watch these sessions:
- Meet AsyncSequence
- Explore Structured Concurrency in Swift

## Suspending

Async functions can suspend. They hand control of the thread over to the system.
Eventually, the system hands control of the thread back to the function which can
then perform more work. It may suspend again, or it may complete and hand control
of the thread back over to the caller.

The `await` keyword is how you notice that the block doesn't execute as one
transaction. The function may suspend, and other things may happen while it's
suspended. The function may resume on an entirely different thread.

To learn more about these issues, check out:
- Protect mutable state with Swift actors

## Async/await facts:

- Marking a function `async` enables a function to suspend.
- When a function suspends itself, it suspends its callers, too, so its callers
  must be marked `async` as well.
- `await` marks where an async function _may_ suspend execution, one or several
  times
- While an `async` function is suspended, the thread is not blocked. Even work
  that gets kicked off later can happen first. So the app state can change a great
  deal while a function is suspended.
- When an async function resumes, the results returned from the async function
  it called flow back into the original function, and execution continues, right
  where it left off.

## Adopting async/await

### Testing async code

`async` makes testing a snap

Before async:

```
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnails() throws {
        let expectation = XCTestExpectation(description: "mock thumbnails completion")
        self.mockViewModel.fetchThumbnail(for: mockID) { result, error in
            XCTAssertNil(error)
            expectation.fulfill()
        }
        wait(for: [expectation], timeout: 5.0)
    }
}
```

Becomes:

```
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnails() async throws {
        XCTAssertNoThrow(try await self.mockViewModel.fetchThumbnail(for: mockID))
    }
}
```

### Bridging from sync to async

#### Calling async functions

Starting point:

```
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placedholder)
            .onAppear {
                self.viewModel.fetchThumbnail(for: post.id) { result, _ in
                    self.image = result                  
                }
            }
    }
}
```

First, remove the completion handler:

```
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placedholder)
            .onAppear {
                self.image = self.viewModel.fetchThumbnail(for: post.id)
            }
    }
}
```

Then, add `try` to handle errors, and `await` to complete the call to an async function

```
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placedholder)
            .onAppear {
                self.image = try? await self.viewModel.fetchThumbnail(for: post.id)
            }
    }
}
```

However, when you try this, an error occurs:

`async 'fetchThumbnail(for:)' used in a context that does not support concurrency`

This is because you can't call an `async` function in a context that isn't itself
`async`. Here, the `.onAppear` modifier takes a plain, non-async closure. There
needs to be a way to bridge the gap between synchronous and asynchronous worlds.

Here, the solution is to use the async task function:

```
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placedholder)
            .onAppear {
                async {
                  self.image = try? await self.viewModel.fetchThumbnail(for: post.id)
                }
            }
    }
}
```

An async task packages up the work in the closure and sends it to the system for
immediate execution on the next available thread. Similar to async function on a
global dispatch queue. The main benefit here is calling async code from inside of
sync contexts.

To learn more, see:
- Explore structured concurrency in Swift
- Discover concurrency in SwiftUI

### Async alternatives

Apple suggests adopting Swift concurrency gradually.

Start small with an async alternative to an existing API. Swap a call with a
`completionHandler` for an `async` function call. Swift compiler will provide
an async alternative in Swift 5.5.

To learn more about async APIs in the SDK, see:
- Use async/await with URLSession
- Bring Code Data concurrency to Swift and SwiftUI
- What's new in AVFoundation
- What's new in AppKit

#### Async alternatives and continuations

There will be times when you'll need to manually create an alternative. For example:

```
func getPersistentPosts(completion: @escaping ([Post], Error?) -> Void) {
    do {
        let req = Post.fetchRequest()
        req.sortDescriptors = [ NSSortDescriptor(key: "date", ascending: true) ]
        let asyncRequest = NSAsynchronousFetchRequest<Post>(fetchRequest: req) { result in
            completion(result.finalResult ?? [], nil)
        }
        try self.managedObjectContext.execute(asyncRequest)
    } catch {
        completion([], error)
    }

}
```

First, make an `async` function, and convert the return value. Since this
function can return an error, we mark this function `throws` as well.

```
func persistentPosts() async throws -> [Post]
```

Next, we call the completion handler version of getPersistentPosts. Now we're
stuck. We need to return the result from the callback back to the places awaiting
calls to the async persistentPosts function. Not only that, those callers are in
a suspended state. We need to make sure to resume them at the right point in
time, and with the right data, so they can get on with the rest of their work.

*Continuation Pattern*
Methods that take completion blocks.

The caller of the method awaits the result of the function call, and provides a
closure to specify what to do next. When the function call completes, it calls
the completion handler to resume whatever the caller wanted to do with the result.

To make this explicit, Swift provides a feature to create, manage, and resume
continuations in a high-level and safe way:

```
func persistentPosts() async throws -> [Post] {
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { (continuation: PostContinuation) in
        self.getPersistentPosts { posts, error in

        }
    }
}
```

Use these functions to gain access to a continuation value.

The continuation value provides a resume function, into which we place the results
from the completion handler. Resume provides the missing link we need to unsuspend
any calls awaiting the result of the persistentPosts function.

```
func persistentPosts() async throws -> [Post] {
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { (continuation: PostContinuation) in
        self.getPersistentPosts { posts, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: posts)
            }
        }
    }
}
```

##### Checked continuations

- Continuations must be resumed _exactly once_ on every path
- Discarding the continuation without resuming is not allowed
- Swift will check your work! If resume isn't called, you'll get a warning. If
  resume is called multiple times, you'll get a fatal error.

To learn more about low-level details, including continuations, see:
- Swift concurrency: Behind the scenes
