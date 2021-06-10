# Meet AsyncSequence

Presenter:

Philippe Hausler, Software Engineer

## What is AsyncSequence?

Example:

```
@main
struct QuakesTool {
    static func main() async throws {
        let endpointURL = URL(string:
            "https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_month.csv")!

        // skip the header line and iterate each one
        // to extract the magnitude, time, latitude, and longitude
        for try await event in endpointURL.lines.dropFirst() {
            let values = event.split(separator: ",")
            let time = values[0]
            let latitude = values[1]
            let longitude = values[2]
            let magnitude = values[4]
            print("Magnitude \(magnitude) on \(time) at \(latitude) \(longitude)")
        }
    }
}
```

This tool fetches lines and emits each line as we get them, instead of waiting
until the entire thing completes.

To reiterate a few things from other sessions:
- Async functions let you write concurrent code without the need for callbacks
  by using the `await` keyword
- Calling an async function will suspend then be resumed whenever a value or
  error is produced

AsyncSequence, on the other hand, will suspend on each element, and resume when
the underlying iterator produces a value or throws.

Basically, they're just like regular sequences, but with a few differences:
- Is just like Sequence except async
- May throw
- Terminates at end or an error

Let's look at iterating:

```
// Iterating a sequence

for quake in quakes {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

The compiler has knowledge of how this iteration should work. But what it does
isn't magic. The compilation step just does some straightforward transformations:

```
// How the compiler handles iteration

var iterator = quakes.makeIterator()
while let quake = iterator.next() {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

First, the compiler creates an iterator variable, and then uses a while loop to
get every quake produced by the iterator when `next` is invoked.

To use the new async/await functionality, there's one slight alteration; change
the next function to be an asynchronous one:

```
// How the compiler handles iteration

var iterator = quakes.makeAsyncIterator()
while let quake = await iterator.next() {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

If the loop was on an AsyncSequence:

```
// Iterating an AsyncSequence

for await quake in quakes {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

### for-await-in syntax

```
// Asynchronously loop over every item from a source
for await quake in quakes {
    if quake.location == nil {
        break
    }
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

If the source could throw an error:

```
for await quake in quakes {
  ...
}

// but AsyncSequence might throw

do {
    for try await quake in quakeDownload {
        ...
    }
} catch {
    ...
}

```

You'd need `try`.

You could also create an async task to encapsulate the iteration:

```
async {
    for await quake in quakes {
      ...
    }
}

// but AsyncSequence might throw
async {
    do {
        for try await quake in quakeDownload {
            ...
        }
    } catch {
        ...
    }
}
```

This could be useful if you know the async sequences you're using might run
indefinitely.

This can also be helpful if you might want to cancel the iteration externally:

```
let iteration1 = async {
    for await quake in quakes {
      ...
    }
}

// but AsyncSequence might throw
let iteration2 = async {
    do {
        for try await quake in quakeDownload {
            ...
        }
    } catch {
        ...
    }
}

// ... later on
iteration1.cancel()
iteration2.cancel()
```

It's pretty easy with tasks to scope the work of an iteration that might be
indefinite.

## Usage and APIs

AsyncSequnce APIs available in:
- MacOS Monterey
- iOS 15
- tvOS 15
- watchOS 8

There are a lot, but these are some highlights:

### Reading from files & networks

Read bytes asynchronously from a `FileHandle`

`public var bytes: AsyncBytes`

```
for try await line in FileHandle.standardInput.bytes.lines {

}
```

Read bytes asynchronously or read lines asynchronously from a `URL`

`public var resourceBytes: AsyncBytes`

`public var lines: AsyncLineSequence<AsyncBytes>`

```
let url = URL(fileURLWithPath: "/tmp/somefile.txt")
for try await line in url.lines {

}
```

Bytes from a `URLSession`

`func bytes(from: URL) async throws -> (AsyncBytes, URLResponse)`

`fund bytes(for: URLRequest) async throws -> (AsyncBytes, URLResponse)`

```
let (bytes, response) = try await URLSession.shared.bytes(from: url)
guard let httpResponse = response as? HTTPURLResponse,
      httpResponse.statusCode == 200 /* OK */ else {
    throw MyNetworkingError.invalidServerResponse
}
for try await byte in bytes {

}
```

### Notifications

Await notifications asynchronously

`public func notifications(named: Notification.Name, object: AnyObject) -> Notifications`

```
let center = NotificationCenter.default
let notification = await center.notifications(named: .NSPersistentStoreRemoteChange).first {
    $0.userInfo[NSStoreUUIDKey] == storeUUID
}
```

## Adopting AsyncSequence

Build your own
- Callbacks
- Some delegates

Pretty much anything that does not need a response back and is just informing of
a new value that occurs can be a prime candidate for making an AsyncSequence.

```
// Example of a common handler pattern

class QuakeMonitor {
    var quakeHandler: (Quake) -> Void
    func startMonitoring()
    func stopMonitoring()
}
```

Implementing this might look like this as a starting point:

```
// Handlers can often be great candidates for AsyncSequence
let monitor = QuakeMonitor()
monitor.quakeHandler = { quake in
    ...
}
monitor.startMonitoring()
...
monitor.stopMonitoring()
```

And this once it's adapted:

```
// Use AsyncStream to adapt existing callbacks or handlers to AsyncSequence

let quakes = AsyncStream(Quake.self) { continuation in
    let monitor = QuakeMonitor()
    monitor.quakeHandler = { quake in
        continuation.yield(quake)
    }
    continuation.onTermination = { _ in
        monitor.stopMonitoring()
    }

    monitor.startMonitoring()
}
```

And then using this AsyncStream would look like:

```
// Using an AsyncStream

let significantQuakes = quakes.filter { quake in
    quake.magnitude > 3
}

for await quake in significantQuakes {
    ...
}
```

AsyncStream is a great way to adapt existing code to use an AsyncSequence:
- AsyncSequence created with callbacks or delegates
- Handles buffering
- Handles cancellation

```
public struct AsyncStream<Element>: AsyncSequence {
    public init(
        _ elementType: Element.Type = Element.self,
        maxBufferedElements limit: Int = .max,
        _ build: (Continuation) -> Void
    )
}
```

AsyncThrowingStream is just like AsyncStream, but can handle errors
- AsyncSequence that can produce errors
- Handles buffering
- Handles cancellation

```
public struct AsyncThrowingStream<Element>: AsyncSequence {
    public init(
        _ elementType: Element.Type = Element.self,
        maxBufferedElements limit: Int = .max,
        _ build: (Continuation) -> Void
    )
}
```
