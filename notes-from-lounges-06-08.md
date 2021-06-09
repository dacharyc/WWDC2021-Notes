# Notes from WWDC2021 Lounges on June 8, 2021

## devtools-lounge

### Concurrency

Q: In what OSes will async/await and actors be supported?

A: On Apple platforms, we currently support iOS 15, macOS Monterey, etc.  This
   is tied to runtime integration with improvements to Dispatch in those OS
   releases for better performance.  We are exploring the possibility of backward
   deployment.

   Concurrency works on Linux.  Still a WIP on Windows.

Q: With async/await now available, how does Combine coexist with it going
   forward? What are some rules of thumb one can apply to decide whether you
   need to use async/await or Combine?

A: Streaming data still makes a lot of sense in the realm of async/await. In
   Swift 5.5 we now have the `AsyncSequence` protocol, which is similar to a
   simplified Combine `Publisher`, and you can use similar functional APIs to
   manipulate async sequences as you would Combine streams, or adapt Combine
   streams to AsyncSequence as well. In async/await code, you can use `for await`
   loops to process async sequences. AsyncSequence and Combine are still great
   for modeling streams of async events, and you can use async/await in functions
   to process streams at the boundary when that stream model no longer makes
   sense.

Q: Lots of WWDC videos are using a free `async()` function to do work – has that
   been deprecated in favor of `Task.init()` or is it changing?

A: The API for creating unstructured tasks in the first seed of Xcode 13 is the
   async function, as you mention. Keep in mind that the design of Swift
   Concurrency is still being refined, since Swift is an open source project, so
   you can help shape it on the Swift Forums:
   https://forums.swift.org/c/evolution/18

Q: We can use Structured Concurrency with Apple  async functions with completion
   handlers. Will we have a kind of bridging support for  custom ones our of the
   box? Or we will need to refactor all existed code to support it?

A: For Swift APIs, you'll need to do the adaptation to `async` on your own. The
   "Meet async/await in Swift" session gives some examples of how to use
   `CheckedContinuation` to adapt existing callback-based APIs to async
   functions. You can do this gradually, though, since the callback and async
   code can coexist, so you don't have to rewrite the world all at once to get
   the benefits of async.

Q: Will Swift Collections and Algorithms packages eventually be added to the
   standard library? What would be the criteria to do so?

A: We're still exploring the balance of how much should go into the Standard
   Library versus come in separate libraries or packages.  Packages provide a
   way for these APIs to evolve more rapidly than Swift releases, but come at
   the downside of not being installed by default. Package collections help with
   this as well, making these packages more easily discoverable.

Q: How lightweight are Actors? What should be considered when declaring an Actor
   in regard of overhead or cost?

A: Creating an instance of an actor type is no more costly than creating an
   instance of a class type. It's only during the access of an actor's protected
   state that a task suspension may happen, to switch to that actor's executor.
   The implementation of that switching is lightweight, and is optimized by both
   the compiler and runtime system. You can find out more about how this works
   in Thursday's session *Swift concurrency: Behind the scenes*:
   https://developer.apple.com/wwdc21/10254

Q: How to run async function from sync code on main thread? Do I need to use
   `detached` or there is something else?

A: Yeah, you can launch either of the forms of unstructured task from sync code.
   In Xcode beta 1, there is `async { }`, which will start a new async task that
   runs on the same actor as the launching context (which can be important if
   that's the MainActor, for main-thread-bound UI work), and `asyncDetached { }`,
   which will start a completely independent task. If you've been following
   Swift open source, though, you may know that these names have been revised
   due to feedback from the open source community; they're known as `Task` and
   `Task.detached` currently.

### DocC Documentation

Q: Really like what I'm seeing on DocC - I've heard its possible to output HTML
   pages from a generated docset - will there be some kind of simplified
   mechanism to apply styling to those pages? Thinking about how an organization
   might want to apply their own brand coloring and such to published docs on
   web.

A: Today there isn’t any integrated affordance for theming the generated site.
   This is certainly something we can see as valuable, though. Also a great
   topic to bring up when we make the project open source later this year.

Q: The "Host and Automate" presentation mentions: "DocC comes with a built-in
   clean design". Will it be possible to theme DocC builds?

A: Ah, already a popular request. Today, there is no integrated support for
   theming the generated site, but would love for you to file a feedback request
   so we can track your interest.

Q: Is there a detailed spec for the .doccarchive output, and/or alternative
   output formats?

A: You can get details about how you can host a `.doccarchive` on your website
   in the following session:  https://developer.apple.com/wwdc21/10236.  We’ll
   have more details about the structure of the `.doccarchive` when we open
   source.

   You can post a followup question with details if you have something specific
   you’re looking to do.

Q: Are there any plans for DocC to support Objective-C documentation builds? My
   org currently uses a docs generator to build both Swift and Objective-C doc
   sets for our SDKs, and we would likely have to continue using that if DocC
   can't build Objective-C docs sets.

A: We’ve definitely heard about the importance of Objective-C, and feel it
   ourselves. It is indeed a priority. Thank you for the feedback!

Q: What’s the best way (via CLI/CI) to generate a documentation archive from a
   library that’s just `Package.swift`, no wrapping Xcode project?

A: xcodebuild docbuild works with a Package.swift file. You'll need to add a
   `-scheme` argument to tell xcodebuild which target to build; you can run
   `xcodebuild -list` to see the list. For example, you can run
   `xcodebuild docbuild -scheme MyPackage` to build a DocC Archive for the
   `MyPackage` scheme. The archive bundle will appear in your Derived Data
   directory (for example,
   `~/Library/Developer/Xcode/DerivedData/MyPackage-[hash]/Build/Products/Debug/MyPackage.doccarchive`).

Q: Is there built-in integration with Xcode Cloud workflows? Or would we need to
   implement it via a script?

A: You’re correct, the best way to support documentation builds on Xcode Cloud
   would be via a script that invokes `xcodebuild docbuild`. There’s a session here
   that is a great reference for a use case like that:
   https://developer.apple.com/videos/play/wwdc2021/10236/

Q: Are there any versioning capabilities in DocC? For example, being able to
   view the documentation for older versions of a package for users that can't
   or don't want to update?

A: If a developer imports a specific version of your package into their project
   as a dependency and uses Build Documentation, they’ll be able to read the
   documentation for that version in Xcode’s documentation window.

   If you have a different workflow in mind, please post a followup and we can
   look at that specifically, or if you have other feedback we’d love to get
   your specific use case in a Feedback Assistant.

Q: It looks like the DocC single page web app requires server side rewrite rules.
   Do you have any tips for using CDNs which don't support this? Github Pages
   has a longstanding issue (https://github.com/isaacs/github/issues/408) and,
   given that many Swift packages are hosted on Github, it seems like many
   projects will want to host their docs there.

Q: The "Host and Automate DocC Documentation" presentation mentions: "The main
   page groups important symbols into topics related to higher-level tasks. Each
   of those group related symbols into more specific topics."

   Can you talk more about how those groupings happen? Is that something we
   dictate by the way we organize and present content in the framework, or can
   we manually specify groupings?

A: Organizing your pages into groups is a great way to improve your
   documentation! To do that, you create a Topics section with a second-level
   heading called "Topics", followed by a group title in a third-level heading,
   and then a bulleted list of links.

   We describe the syntax in more detail here
   https://developer.apple.com/documentation/xcode/adding-structure-to-your-documentation-pages,
   under "Arrange Top-Level Symbols Using Topic Groups". We also have a session
   tomorrow that shows how to do this:
   https://developer.apple.com/videos/play/wwdc2021/10167/ along with some other
   ways you can improve your documentation.

Q: Usually a problem of documentation is that it becomes obsolete with time. Is
   there any way to prevent this by using docC? Is there any validation in the
   code that can be autocompleted?

A: We think so too! It's important to keep documentation up to date with the
   changes you make in your code. Like a code compiler, DocC can emit warnings
   and errors when it finds something that doesn't quite look right.

   For example, DocC checks all of the links in your documentation you've
   written using double-backtick syntax, like ``MyType``. If you remove or
   change the name of an API in your code, Xcode will display a warning on the
   line containing the documentation letting you know that you should update
   your docs.

   If you're interested in other ways to validate the documentation, we'd be
   interested to hear more via Feedback Assistant.

Q: Is there any way to know the documentation coverage of the public symbols of
   a framework? It would help a lot with adoption and finding blind spots on
   huge code bases.

A: DocC does not currently expose documentation coverage but this is
   great feedback. Could you please file a Feedback Assistant request with
   details about how you'd want this to work? These types of enhancement
   requests are super helpful.

Q: Is there any limitation that would inhibit the ability to extract docs for
   symbols other than public or open? Seems like it would be useful for internal
   use, and as a way to collect data on undocumented symbols.

A: Thanks for the feedback! This is something that a lot of people have been
   interested in over the last few days :). We built the DocC integration in
   Xcode first and foremost to support public-facing docs. We know that this is
   important for teams working on a framework and will keep this in mind in the
   future—thanks again for the question!

Q: When building docs locally, does the archive get stored in derived data?
   i.e. are devs going to have to rebuild docs every time they clear derived
   data to fix weird issues?

A: When you use the Build Documentation menu in Xcode,
   the doccarchive does go into the derived data directory, so if you clear
   that, you would have to rebuild the docs.

   If you want to keep a version of the built docs in the Documentation window,
   you can export the doccarchive, and then you can double click it which imports
   it back into the Documentation window.  When you import a .doccarchive, Xcode
   puts it into an “Imported Documentation” section of the Documentation window,
   and that doesn't get removed when cleaning the derived data.

Q: In some of our internal modules we have some utility methods in extensions of
   types that are part of another consumed frameworks (for example: Foundation).
   We're working towards removing the majority of them in favor of other
   solutions that avoid extensions. In the meantime though, I haven't seen them
   in our generated docs and would like to know if there's a way to include them
   so that they are discoverable while we deprecate/move them.

A: DocC in Xcode 13 supports
   documentation for APIs defined in your module, but we know that documenting
   extension symbols is also very important. This is something that would also
   be a great topic for us to discuss when we open source later this year.

Q: Is there a way to omit some public symbols from the generated docs? For
   example, my UIView subclass overrides a series of methods like layoutSubviews
   that I don’t want to be documented but they currently show in the
   documentation archive. I had a scan through the docs but couldn’t seem to see
   a way to opt out

A: If you prefix a public symbol with an underscore, DocC will automatically
   hide it from the built documentation.

   In your particular case, because you can't change the name of the method,
   that won't work. We're interested to hear more about this scenario and how it
   impacts your documentation via Feedback Assistant.

Q: Is it possible to use DocC for local Swift packages? We're writing a lot of
   documentation for local modules (mostly for services and core API), and it
   would be really helpful to use DocC for them too.

A: Yes, DocC works with both local and remote Swift packages.

   If you open the Swift package in Xcode, and use the Product > Build
   Documentation menu item, Xcode will build documentation for both that package
   and its dependencies (both local and remote).

   Note: if you use a documentation catalog in your Swift package, make sure
   that the Package manifest’s swift-tools-version is set to 5.5
   For more information about documenting a Swift package, see
   https://developer.apple.com/documentation/xcode/documenting-a-swift-framework-or-package

Q: I noticed that the website that gets generated in the .doccarchive doesn't have
   search functionality. Is there any plan to add this feature?

A: This is a great question, and a really good feature request. You’re correct-
   Today, search is supported when viewing documentation archives in Xcode’s
   documentation window and it would be great to have search available on the
   web as well.

   This is something we’ll definitely be interested in discussing further with
   the community when we open source DocC later this year.

DocC Documentation: https://developer.apple.com/documentation/docc

## swiftui-lounge

Q: is the refreshable property the only SwiftUI property that supports async
   code?

A: The `task` modifier also provides support for async code out of the box!
   Generally, we only make user provided closures `async` if we can provide some
   additional utility to you by doing so, such as canceling a `task` when the
   attached view’s lifetime ends, or finishing the refresh animation. If you
   want to dispatch to async code in other places, you can use an `async` block!

Q: What is the proper way to dismiss a .fullscreenCover when it is being
   presented based off an item: setting the item to nil or dismissing using the
   presentationMode?

A: That really depends on what is doing the dismissing. They both have the same
   effect in setting the item binding back to nil. It is more about where you
   are driving the dismissal state from. If you're dismissing from within the
   sheet's content, then using the presentation mode will be more flexible
   because that will work no matter what is controlling the presentation.

   There is also a new environment property introduced this year that can
   accomplish this as well, which is dismiss.
   https://developer.apple.com/documentation/swiftui/environmentvalues/dismiss/

Q: Is `.searchable` for local data only, or can it be used to query services?

A: Searchable specifies that the view supports search. You can implement that
   search however works best for your app.

   For example, you could kick off a query using the bound search term, then
   update the results when your query completes.
