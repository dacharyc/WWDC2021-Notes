# Notes from WWDC2021 Lounges on June 10, 2021

## devtools-lounge

### Swift Package Manager

Q: Our project has multiple build configurations, not just release and debug.
   Currently with Cocoapods we can have different dependencies for each build
   configuration. Last time we checked this was not possible with SPM. Has this
   changed? If not, do you expect this to change in the future?

A: There is a proposal for conditional target dependencies (see
   https://github.com/apple/swift-evolution/blob/main/proposals/0273-swiftpm-conditional-target-dependencies.md)
   but this has not yet been fully implemented.

Q: Is there anything we can do on our end to help get this implemented?

A: Because Swift Package Manager is open source, the best way would be to get
   involved in the forums at forums.swift.org.

Q: Is there a way to only update a single package? If I have multiple packages
   specifying "Up to next major", Xcode will try to update all of them when I
   run "Update to Latest Package Versions" but sometimes I might only want to
   update a specific one.

A: Not at the moment, but it is something we are looking to make available on
   both the CLI and Xcode.

Q: Does the presence of Playgrounds for iPad mean that we can replace
   xcodeproj-based projects with SPM-based projects even for non-console apps?
   Thanks!

A: Because projects in Swift Playgrounds are built on top of the Swift Package
   format, they'll also work great in Xcode when making apps for iPadOS.

Q: What is the best way to integrate SPM into a large project such that it does
   not impact, ideally it improves, build times?

A: Swift Packages are primarily intended to improve modularization of the code.  
   In some cases there may be runtime or build time improvements based on how
   much time is spent compiling vs linking, but there is general no build time
   difference between using packages vs projects.

Q: In terms of app-startup performance which approach is better. Many concise
   dynamic libraries or few large dynamic libraries. I have been told embedding
   many dynamic libraries affect app-startup. Can some please clarify this doubt.

A: Tomorrow's 1-4pm "C, C++, Obj-C, compiler, analyzer, debugger, and linker lab"
   is a great place to ask for advice on the launch-time impact of using many
   dynamic libraries. The performance penalty may be less now than it was in
   prior releases, but you'll be able to get the most up-to-date answer there.

Q: My current workflow for updating packages is to: make the update, check out
   the commit window to see what the old version numbers are and the new version
   numbers, then go search the internet to find release notes/changelogs for
   each to find out what has changed. Is there something yall have done this
   year or could do to improve this? :)

A: If the package in question is part of a package collection, you can see
   release notes right in the "add package" workflow in Xcode 13.

Q: Is it possible to compile a Package with -Osize optimization? I noticed it
   uses -O by default, I tried to modify it using -Xswiftc flag from swift
   compiler without success (Of course I am looking for a solution that doesn't
   involve unsafeFlags)

A: This is not currently possible without using `unsafeFlags()`.

Q: Is it possible to "stage" a release for pre-release testing? Or what's the
   best workflow for testing before publishing a release?

A: You can publish a pre-release of your package. Pre-release versions are
   opt-in for users of your package.

Q: And that would be by tagging using the following pattern
   https://semver.org/#spec-item-9 ?

A: Yep, exactly that.

Q: What's the best method for simultaneously develop an app and its dependent
   Swift packages? (local development with submodule?)

A: Check out https://developer.apple.com/documentation/swift_packages/editing_a_package_dependency_as_a_local_package

Q: Are there any plans to expand the build settings so we don't have to use
   `.unsafeFlags` (thinking of framework search path). And using the .unsafeFlags
   is really not recommended if you want to do a release build ;-)

   Additionally any plans for supporting custom configurations?

A: This is certainly a common request.  Since SwiftPM is open source, any
   improvements here would start with a pitch and a proposal on forums.swift.org
   and that is a great place to discuss the kinds of options that would be needed
   for your situation.

   This would also be the case for discussions about custom configurations (in
   addition to the currently supported Release and Debug configurations).

Q: If I've got private dependencies from, let's say GitHub via ssh. If they
   require different credentials, I can use `~/.ssh/config` to tie ssh keys to
   specific configurations. SPM uses that perfectly. But if I put same Package.swift
   file into Xcode, it does not. Is it possible to make Xcode behave like
   `swift package` commands in this case? Thanks!

A: You can configure Xcode to use the system git instead of the builtin one,
   which is what SwiftPM uses on the command line. See "Use Your System’s Git
   Tooling" in
   https://developer.apple.com/documentation/swift_packages/building_swift_packages_or_apps_that_use_them_in_continuous_integration_workflows

Q: Is it technically feasible to add multi-language support for single targets?
   (Objective-C and Swift in this case). We'd love to start building our new APIs
   in Swift but have some difficulties when using separate targets.

A: Swift Packages do not currently support mixing Swift and Clang-based languages
   (C, ObjC, C++, etc) in the same target; it would be technically feasible and
   would be a great topic for a forum.swift.org pitch.  However, it should be
   possible to use separate targets in the same package to accomplish this — if
   you are running into problems doing so, please file a bug report on bugs.swift.org
   if the issues are in open source SwiftPM, or through the Xcode feedback process
   if there it is a problem in Xcode.

Q: Would that be 2 targets. One for a Clang based language and the other Swift
   combined into one product? Something like:
   `.library(name: "MyLib",  targets: ["MyClangTarget", "MySwiftTarget"])`

A: Yes, that is correct — there would be two separate targets, one with Swift
   sources and the other with Objective-C sources, and they can be combined into
   the same library product.  It is also possible (and fairly common) to have
   the Swift target depend on the Clang target, if this is being used to provide
   Swift wrappers on C-based APIs.  In this case, only the Swift target would
   need to be added to the product, and its dependency on the Clang target would
   indirectly include it in the product too.

Q: Is there a way to check to see if there are updates available without
   installing them, equivalent to `pod outdated`?

A: There's not a convenient command to do this yet. As a workaround, you could
   perform an update, then revert your `Package.resolved` file to its prior
   state with your source control system to go back to your old dependency
   versions.

Q: Is it possible to distribute a DocC together with an XCFramework using SPM?

A: You can find an answer for this question from a prior lab here: [Link to a
   different Slack thread - thread content imported here for convenience]

   Q: Will the DocC compiler run only over source code dependencies or will it
      also generate the docs for the public interfaces of consumed xcframeworks?

   A: DocC integrates with the Swift compiler to extract information about the
      public symbols and their documentation during the build.

      The author of an xcframework can build and export that framework’s
      documentation and distribute it alongside the xcframework to enable
      consumers to read the framework's documentation in Xcode.

   Q: Do you have any tips or references on how to best distribute them together
      with the xcframework so that they are detected by Xcode? Is there any SPM
      tooling around this or is your answer more about a context of a manual
      import?

   A: That's a great question and something that we want to explore more.

      For now, this can be accomplished if the framework author exports and
      uploads a separate zip file of the DocC Archive. Consumers of the
      xcframework can then download the documentation archive, and double-click
      it to import that documentation into Xcode.

Q: I'm currently using Swift Lambda and I very much looking forward to the
   upcoming SPM plugin feature. I'm wondering if the plugin feature could be
   used for packing the Swift Lambda for AWS deployment (I'm thinking for
   creating the bootstrap symbolic link and package up the binaries into a zip
   file)

A: One of the future usage of the plugins feature is packaging for deployment,
   such as the one you describe. This is not covered by
   [SE-0303](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md),
   but we foresee a follow up proposal to add this functionality.

Q: It would then be up to the Swift Lambda project to provide the plugin for
   deployment of the Lambda runtime right?

A: Yes, it would make sense for the runtime library to also provide such plugin.

Q: What is the recommended way to cache packages for a CI setup. I use Gitlab CI
   and fastlane, and when running CI, my packages get pulled down fresh every
   time, causing long build times (and some times timeouts).

A: Starting Swift 5.4, Swift Package Manager caches package dependencies on a
   per-user basis, which reduces the amount of network traffic and increases
   performance of dependency resolution for subsequent uses of the same package.
   This means that while the initial clone may still be slow depending on the
   repository size, subsequent clones are much faster. This can also be used in
   CI by configuring the cache location. See
   https://swift.org/blog/swift-5-4-released/

Q: Are there any recommended workflows for using run scripts inside of a
   Package.swift file? There have definitely been a few times where I've wanted
   to have my Package.swift file generate things like compiled protobuf models.

A: [SE-0303](https://github.com/apple/swift-evolution/blob/main/proposals/0303-swiftpm-extensible-build-tools.md)
   defines new functionality in SwiftPM enabling the creation of structured
   plugins for code generation.
