---
layout: post
title:  "Integrating Tailor into your Swift Project"
date:   2015-11-12 00:15:00
---

Finally, we have a style-checker for our [Swift](https://developer.apple.com/swift/) projects (read: iOS and OS X applications). [Tailor](https://github.com/sleekbyte/tailor) is a command line tool for OS X, Linux, and Windows that analyzes your Swift source code and enforces the rules for the format of your code. This is similar to other tools like [checkstyle](https://github.com/checkstyle/checkstyle) for Java. In this post I will walk through using Tailor, integrating it into your Swift project through Xcode, and getting it to work in your build environments.

# Using Tailor

Installation is very easy, and if you're working on a Mac, you can even use [Homebrew](http://brew.sh) to install Tailor.

Once you have it on your machine, you can analyze any Swift file with the following command via the command line:

    $ tailor foo.swift

Or analyze all Swift files in a directory:

    $ tailor path/to/directory/

Easy! Tailor uses a set of pre-defined rules for analyzing your code. These rules are described [here](https://github.com/sleekbyte/tailor/wiki/Rules). Any of these can be disabled if you don't want Tailor to check for them. For example, to disable the `brace-style` rule, you can use the following:

    $ tailor --except=brace-style foo.swift

This is great if you don't agree with a certain rule. Unfortunately, there is no way to specify your own rules right now, but that is a feature that could come later on.

# Integrating Tailor Into Your Project

Now that you know how Tailor works, it's time to let it work its magic on your Xcode Project. If you decide to integrate Tailor into your project, it will run and analyze your code at build time, letting you know of any style violations that you have each time you compile.

Tailor makes integration easy, but you'll have to do the customization yourself. To start, use the tool that tailor provides:

    $ tailor --xcode /path/to/foo.xcodeproj/

This will add a custom script to the end of your Xcode build. If you open up this project in Xcode and go to the `.xcodeproj` file, you'll see this script under the name "Tailor". Currently, you'll see that the script simply does something similar to the following:

    /usr/local/bin/tailor

This will run Tailor on all of the swift files in your project without any options. This is a great starting point, but we often want ignore some rules or add other options. You can do this by modifying the script yourself.

    /usr/local/bin/tailor --except=brace-style

It looks nearly identical to our command-line commands, which makes it easy to customize. Once you have it the way you like it, try to build your project! You'll notice the custom script running toward the end of your build, and if any warnings occur they will show in Xcode. It will even highlight the line that causes the warning in the file view, just like default Xcode warnings.

Often, we want these warnings to fail the build so that we can consistently enforce the style. This is also easily done with Tailor. Simply changing the severity level from warning to error will make sure that the build fails. Add the following to your Xcode build script:

    /usr/local/bin/tailor --max-severity=error

Now, if you build your app, you'll notice many of your warnings turn into errors, along with a build failure (that is - if you actually have violations in your code. If you get no errors, then you've been following the Swift style guide very well!).

Setting the severity to error is often helpful when you have a project shared across many contributors, in order to make sure that everyone is following the style.

# Continuous Integration

Now that we have Tailor working for our project locally, we want to make sure that when we push up our changes to our remote repository, our continuous integration server is able to run the custom script when it builds via Xcode. I'm going to show an example based on [Travis CI](https://travis-ci.org), but this should apply in a similar way to other CI services.

First things first, modify your `.travis.yml` file to have the build server install Tailor before building your project. Add the following lines:

    before_install:
      - brew update
      - brew install tailor

This will install Tailor on the Travis build machine before it pulls and builds your code.

If you try pushing these changes now, Tailor will be able to run successfully! This may be enough for your project. But, remember that Tailor examines **all** `.swift` files in your project. Unfortunately, this includes Swift code in your [CocoaPods](https://cocoapods.org) dependencies. Luckily, there is an easy fix for this.

Create a new file called `.tailor.yml` in the top-level directory of your project (same level as the `.travis.yml` file). Then, add the following lines to it:

    exclude:
      - Pods/

You can exclude certain files or directories from being analyzed. You'll definitely want to at least have the Pods directory in this list, because many of these frameworks do not follow the style rules.

Now when you push, the Travis server should install Tailor and run the custom script in your project. If you've fixed all of the style violations, it should build successfully and run your tests! Your project will now have a consistent style, among all developers, which can make for a much cleaner and easier to maintain codebase.

Tailor is a great tool that will only become more robust and useful over time, so I highly recommend trying it out for your projects and seeing how it works. I've already integrated it into my project, [Pilot](https://github.com/RohanNagar/pilot-osx), and plan to use it in any future Swift project. Tailor is also available on Linux and Windows, which is great for when Swift becomes open-source later this year.

Let me know what you think of Tailor and if you're having any problems using it - I'd love to hear your opinion.
