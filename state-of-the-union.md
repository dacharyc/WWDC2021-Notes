# Platform State of the Union WWDC2021

Monday, June 7, 2021

## Developing Apps

- Code
- Test
- Integrate
- Deliver
- Refine

Context switching disrupts focus and pulls you away from code. Remove friction
and let you and your team focus on creating great experiences.

### Xcode Cloud

A new CI/CD service built into Xcode and hosted in the cloud. Manage every stage
of the development process. Designed for all Apple platforms. Built on Apple's
cloud infrastructure to offload builds, tests, and code signing for distribution.
Integrates with Apple services like Test Flight and App Store Connect. Every
major git provider.

How does it work?
- Create and manage Xcode Cloud workflows in Xcode
- When Xcode Cloud finishes a build, you see the results in Xcode

Steps:
1. Go to Product -> Xcode Cloud -> Create Workflow
2. Xcode Cloud automatically detects the products and platforms for project. Press
   the Next button.
3. Xcode Cloud suggests workflow steps. Review, edit if needed, or press the Next
   button. The Default actions build every change you make.
4. Xcode Cloud securely connects to hosted account for source code. You should
   see the linked git repository for the project in the "Grant Access to Your
   Source Code" step. Press the Next button.
5. Xcode Cloud will check App Store Connect and confirm whether the app is
   already on the App Store. If it is, confirm it's the correct app and press the
   Complete button. If it's not already on the App Store, Xcode Cloud can
   register it for you.

Press the Start Build button to start the build in the cloud. When the build is
finished, you can view the report in the Report Navigator.

There is now a Cloud tag in the Report Navigator. Click it to review the build.
View the results of each step. Check out the source, or trigger a rebuild.

Add a Pull Request workflow to run tests on every new pull request to the main
branch. Set it for Beta builds or specific versions. Add tests and simulators.
Add a Notify post-action to find out when things succeed or fail.

You can also run custom build scripts and use Xcode Cloud's webhooks and APIs
to integrate with other systems.

Configure to run multiple test plans across multiple platforms/simulators, all
in parallel. Xcode Cloud helps you test more, and Xcode 13 helps you test better.

### Test

Run UX tests, get screenshots of different modes on different simulators.

Run unit tests, run inconsistent unit tests a set number of times to determine
whether it's actually unreliable, adopt the new "Expected Fail API" and include
a message about reliability for team members. Use the "Run test again" option
from the Product menu. Xcode remembers what you did last time so it's really
easy. Test is raising assertions but not failing. Gentle reminder to fix it down
the road.

### Integrate

Peer feedback. Integrate with git hosting providers. Get the discussions with
your team directly in Xcode.

In Navigator on left, there's a new source control "Changes" tab. You'll see the
files that are changed. The PR. Changes that are included in the PR.

When you select the PR, you get a full overview of activity, code feedback,
conversations, etc. At the top, you can see a live status from Xcode Cloud
workflows/tests.

View diff inline in Xcode. Compare against any branch or tag in history. Use
code review in any editor window, including split windows. View and respond
to everything directly in Xcode.

### Deliver

Code signing in the Cloud. No need to keep certificates and profiles up-to-date
on Mac. Archive uses the same system to sign app for distribution.

Post action to Xcode Cloud workflow gives you automatic delivery of betas through
Test Flight to all Apple platforms, including Mac. Xcode 13 includes crash logs
from Test Flight, written feedback from users, a broader view into your app's
usage.

Debug navigator has backtrace, editor highlights the assertion, PR conversation
displays. Everything you need to fix the problem is right in Xcode.

Xcode Cloud will be available as a free, limited beta. Developer account holders
can sign up now. Gradually add more teams. More details about pricing and
availability this fall.


## Swift

### CONCURRENCY.

Writing concurrent code is really hard to get right without language support.
Our approach to building concurrency into the language is focused on making it
easy to write modern, safe, fast code.

Modern code = structured, easy to express what you want to do. Most of today's
async code uses completion handlers that are complex and difficult to express.

Async/Await pattern.

Example:

```
func loadImage(at url: URL) async -> Image

let image = await loadImage(at: url)
```

Example of code using completion handlers:

```
func prepareForShow(completion: @escaping (Result<Scene, Error>) -> Void) {
  danceCompany.warmUp(duration: .minutes(45)) { result in
    switch result {
    case .success(let dancers):
      self.crew.fetchStageScenery { scenery in
        self.setStage(with: scenery) { openingScene in
          dancers.moveToPosition(in: openingScene) { result in
            completion(result)
          }
        }
      }
    case .failure(let error):
      completion(.failure(error))
    }
  }
}
```

Adopting async/await in this example looks something like this:

```
func prepareForShow() async throws -> Scene {
  let dancers = try await danceCompany.warmUp(duration: .minutes(45))
  let scenery = await crew.fetchStageScenery()
  let openingScene = setStage(with: scenery)
  return try await dancers.moveToPosition(in: openingScene)
}
```

#### Structured Concurrency

A way of organizing concurrent tasks to make them easier to reason about.

In above example, crew waits for warmup to complete before fetching scenery.
Could instead create concurrent child tasks like this:

```
func prepareForShow() async throws -> Scene {
  async let dancers = try await danceCompany.warmUp(duration: .minutes(45))
  async let scenery = await crew.fetchStageScenery()
  let openingScene = setStage(with: await scenery)
  return try await dancers.moveToPosition(in: openingScene)
}
```

Safety feature. Compiler helps to eliminate common concurrency issues by making
sure access to shared state is safely coordinated between concurrent tasks.

#### Actors

Industry-proven model for safe concurrent programming, and a powerful
synchronization primitive. Conceptually, an actor is an object that protects its
own state by only providing mutually-exclusive access. This completely eliminates
concurrent access, and the low-level data races that come with it.

Similar to pattern you already use with DispatchQueue. Pattern is prone to
mistakes. There's a lot of boilerplate, it's easy to forget to manually use the
queue just once and introduce a race condition into your code.

Example:

```
class StageManager {
  var stage: Stage
  let queue = DispatchQueue(label: "stage")

  func setStage(with scenery: Scenery, completion: @escaping (Scene) -> Void) {
    queue.async {
      self.stage.backdrop = scenery.backdrop
      for prop in scenery.props {
        self.stage.addProp(prop)
      }
      completion(self.stage.currentScene)
    }
  }
}
```

We went back to core idea of actors and built it into Swift as a first-class
construct.

You can declare an actor type in Swift with a simple keyword. Same structure as
the constructs you already know. No need for manual synchronization.

Example:

```
actor StageManager {
  var stage: Stage

  func setStage(with scenery: Scenery) -> Scene {
    stage.backdrop = scenery.backdrop
    for prop in scenery.props {
      stage.addProp(prop)
    }
    return stage.currentScene
  }
}
```

With actors build into Swift:
- Synchronizing access to actor state can be managed for you automatically
- An actor can access its own properties directly
- Interacting with an actor externally uses async/await to guarantee mutual exclusion

This concept solves common source of concurrency problems. Main thread.

Instead of manually dispatching to the main queue like so each time you call an
API that must be run on the main thread:

```
DispatchQueue.main.async
```

We're introducing a way to make sure that an API always runs on the main thread
using `@MainActor`. Annotate the declaration:

```
@MainActor
func display(scene: Scene)
```

Make it easier to write safe, concurrent code that you don't have to manage
yourself.

Async/await makes it easier to optimize. Reducing reference counts, addressing
concurrency-specific issues such as excessive context switches. Code will get
faster in the coming years.

Existing async APIs in the SDK have been refined. Async/await is enabled with
these APIs. We've added new, purposely-crafted APIs that take advantage of
async/await re: URLs.

## SwiftUI

This year's focus is on APIs that are critical to building apps.

List
- Easily add a swipe action
- Adding pull to refresh is one more line
- Add a modifier to a single platform
- Add a search field as one more line
- Add search suggestions

Accessibility support
- Inherit accessibility implementation from standard UI components

Easily add new list views

New Materials support. Add background images. Apply material styles behind the
content to automatically apply rendering to text, symbols and UI to make it
more readable.

## Swift Playgrounds

Great way to learn, experiment. You can now build apps and submit to App Store
right from Swift Playgrounds. View live preview of app while developing in
Swift Playgrounds

## AR

Historically, building great AR experiences has required deep knowledge of 3D
modeling and mastery of sophisticated rendering engines. Apple wants everyone to
be able to create great AR experiences. They've released a suite of AR tools.

### RealityKit

RealityKit is Apple's 3D Rendering, Audio, Animation and Physics engine.

- Photorealistic rendering
- Camera effects
- People and Object Occlusion

Takes advantage of latest hardware, like lidar scanner, to enable virtual
objects to behave as if they were really there.

Apple announces RealityKit 2, with lots of new stuff. Make it easy to create
a 3D model with ObjectCapture.

Take a bunch of photos to capture the object from every angle. Use an app like
Clone to make the workflow easy.

Use a few lines of code to create the 3D object. For example:

```
let session = try! PhotogrammetrySession(input: imagesURL)

try! session.process(requests: [.modelFile("model.usdz", detail: .medium)])

async {
  for try await output in session.outputs {
    // Receive .modelFile(URL)
  }
}
```

Use ObjectCapture today w/sample code. Working with pro apps to bring it into
things like Unity Mars, Cinema 4D, and Clone.

New APIs to enable 3d model building and immersive AR experiences.

### Metal

Metal Compute APIs accelerating the next phase of gaming development.

Three key features:
- Graphics & Compute Engine
- 
