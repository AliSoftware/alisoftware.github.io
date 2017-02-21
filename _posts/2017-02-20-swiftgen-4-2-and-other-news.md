---
layout: post
title: SwiftGen 4.2 and other news
date: 2017-02-19
categories: news oss
---

SwiftGen — my tool to generate Swift code so you can use your images, localized strings, fonts, storyboards and other assets in a type-safe way — has just been released in version 4.2 after a big internal refactoring. I've also been working on other OSS projects lately.  
  
This article intends to give you some news on all those various OSS projects as well as what has been going on lately and what's next to come.

Table of Contents: [SwiftGen](#swiftgen) • [Reusable](#reusable) • [fastlane](#fastlane) • [AppDevCon](#appdevcon-amsterdam-march-16-17) • [NSBudapest & CraftConf](#nsbudapest--craftconf-budapest-april-25-28) • [Blog Articles](#blog-articles)
{: .note }

## SwiftGen

The thing that took most of my time lately is probably the new SwiftGen release, which just went through a big internal refactoring, consisting of moving `SwiftGen` to its own GitHub organization and splitting it into multiple repositories.

We even have a brand new logo now!

<h3 align="center">
  <a href="https://github.com/SwiftGen/SwiftGen">
    <img src="https://raw.githubusercontent.com/SwiftGen/Eve/master/logo/logo-256.png" alt="SwiftGen's new logo" height="256" />
  </a>
</h3>

### Why splitting SwiftGen into multiple repositories?

* Since the [Sourcery](https://github.com/krzysztofzablocki/Sourcery) project came to life — and because this new tool also uses [Stencil](https://github.com/kylef/Stencil) the same as we always did at `SwiftGen` — Krzysztof asked us to share the code we use to drive `Stencil`, to use it with his new tool too. This way `SwiftGen` and `Sourcery` could use the same tags, filters, and extensions to write their templates similarly, an users would have consistent understanding of the template language between both tools. That meant splitting that common code in its own framework, to make it independent of `SwiftGen` so that `Sourcery` could depend on it too.
* Having separate repositories also helps separate the code of `SwiftGen` into clear modules with clear responsibilities.
  * It should also help potential contributors to understand the code better, as each part of the `SwiftGen` pipeline will be clearly separated and documented.
  * This also allows those modules to be used independently of SwiftGen's CLI, if other tools want to build something around them.
* It allows those modules, as well as templates, to evolve with their own versions lifecycles, for example making it possible to release a new module or tag new templates without having to release a new version of `SwiftGen` as a whole.

### Why move SwiftGen into its own GitHub organization?

* Since [David Jennes](https://github.com/djbe) has become a core contributor (thanks again to him as he did most of the work on the refactoring PRs!), I felt like `SwiftGen` has more and more become a community than just my own project
* This use of a GitHub organization will hopefully help making the `SwiftGen` community grow, by attaching `SwiftGen` to an organization rather than an individual (me)
* Since we needed to split `SwiftGen` in multiple repositories anyway, having a GitHub organizaton to host them all was the logical choice

### What changed for users?

Actually, not that much. That big refactoring was necessary internally, but that shouldn't change anything for the end users. That's why this release is a minor one (4.1 -> 4.2) and not a major version bump.

This release also fixes some bugs, mainly the warning on `import YourAppModule` that was present for storyboards, and adds some new features, mainly support for `--param X=Y` to pass arbitrary parameters to your templates. See [the CHANGELOGs](https://github.com/SwiftGen/SwiftGen/blob/4.2.0/CHANGELOG.md) for more details.

### What's next?

We already have the next milestores for `SwiftGen` planned:

* Version 4.2.1 will focus on improving documentation:
  * Update the global documentation explaining the new organization in the repo (what's the responsibility of each module)
  * A Markdown per template to document them each in detail and help you choose the right template
  * Providing a `CONTRIBUTING.md` and a `CODE_OF_CONDUCT.md` and documentation to help contributors feel even more welcome and help them contribute
* Version 5.0.0 will clean some stuff up:
  * Removing old legacy code and template variables (that were only present because Stencil wasn't as advanced back then as it is now)
  * Re-organize and rename the templates bundled with `SwiftGen` (especially so that `SwiftGen` won't default to using Swift 2 templates, providing more alternatives, and organizing them in directories to overcome their growing number)

**If you have created your own custom templates**, as some templates variables will be removed in SwiftGen 5.0, I highly suggest that you start adapting them as soon as possible so they'll be ready and still working for SwiftGen 5.0 when it's released.  
This simply consists of using the new variables instead of the old ones. Both the deprecated variables and the new ones are present in SwiftGen 4.2, so this transition should go smoothly.

To help you migrate your templates to be ready for the upcoming SwiftGen 5.0, you can follow the dedicated [Migration Guide](https://github.com/SwiftGen/SwiftGen/wiki/migration-guides).

## Reusable

<h3 align="center">
  <a href="https://github.com/AliSoftware/Reusable">
    <img src="https://raw.githubusercontent.com/AliSoftware/Reusable/master/Logo.png" alt="Reusable logo" height="128" />
  </a>
</h3>

During my absence from this blog, we also worked on my [Reusable](https://github.com/AliSoftware/Reusable) pod, which is a "Mixin" to help you work with reusable views (like `UITableViewCell` and `UICollectionViewCell` subclasses), views designed in a XIB, and VCs loaded via a Storyboard, all in a type-safe way.

This pod is a great complement to `SwiftGen` to make your code even safer and continue getting rid of all String-based APIs.

If you want to know more about how it works, you can read my blog post about [Mixins](http://alisoftware.github.io/swift/protocol/2015/11/08/mixins-over-inheritance/) as well as the [dedicated blog post about using Generics to dequeue cells safely](http://alisoftware.github.io/swift/generics/2016/01/06/generic-tableviewcells/).

We recently released [Reusable 4](https://github.com/AliSoftware/Reusable/releases/tag/4.0.0) which improves the API and makes it even safer, so take a look and update your projects!

## fastlane

<h3 align="center">
  <a href="https://github.com/fastlane/fastlane">
    <img src="https://github.com/fastlane/fastlane/raw/master/fastlane/assets/fastlane_text.png" alt="fastlane Logo" height="128" />
  </a>
</h3>

I also have the great pleasure to now [be part of the _fastlane_ Core Contributor team](https://twitter.com/FastlaneTools/status/824286337415135232) since recently.

As a Core Contributor, I plan to contribute to _fastlane_ more, but also help other people wanting to contribute to _fastlane_ to feel welcome, help them do their first ruby PRs and make the _fastlane_ community grow even more!

## AppDevCon (Amsterdam, March 16-17)

<h3 align="center">
  <a href="http://appdevcon.nl">
    <img src="https://pbs.twimg.com/profile_images/802779268631658496/N5IYeRr3.jpg" alt="AppDevCon Conference Logo" height="128" />
  </a>
</h3>

Next month, I'll be talking at the [AppDevCon](http://appdevcon.nl) conference (formerly known as "mDevCon") in Amsterdam.

This is a wonderful and popular conference in Europe, and I encourage you to go if you can. And Amsterdam is such a beautiful city, that would make such a good excuse to visit anyway!

Among other talks at AppDevCon are:

* A tutorial session about `Sourcery` and `SwiftGen` by Paweł Brągoszewski: ["Adding magic to Swift with SwiftGen and Sourcery"](http://appdevcon.nl/session/pawel-bragoszewski-adding-magic-to-swift-with-swiftgen-and-sourcery/)
* [My talk about Mixins over Inheritance](http://appdevcon.nl/session/olivier-halligon-mixins-over-inheritance/)
* A talk by Andrea Falcone, member of the _fastlane_ team, about ["Supercharging your mobile app release with fastlane"](http://appdevcon.nl/session/andrea-falcone-fabric-fastlane/)

## NSBudapest & CraftConf (Budapest, April 25-28)

<h3 align="center">
  <a href="https://www.meetup.com/NSBudapest/">
    <img src="https://pbs.twimg.com/profile_images/664400531687874560/m2uY7_RL_400x400.jpg" alt="NSBudapest Logo" height="128" />
  </a>
  <a href="https://craft-conf.com">
    <img src="https://pbs.twimg.com/profile_images/378800000824012775/97bb1d972974a336814f088a932add3f_400x400.png" alt="CraftConf Logo" height="128" />
  </a>
</h3>

At the end of April I'll also be talking at the [NSBudapest meetup](https://www.meetup.com/NSBudapest/) that will take place during the [Craft Conference](https://craft-conf.com) in Budapest, Hungary.

I'll be talking about SwiftGen here, as well as code generation in general, but there are also plenty of other very interesting talks there too, including about _fastlane_ and Sourcery too (and I hear Budapest is a wonderful city to visit as well!), so I'm definitely looking forward to it… and maybe meet some of you too!

## Blog Articles

I know that I've letting you down for a little while with my blog, and haven't been writing since a long time.

If you read all those news above, you probably understand how busy I have been these past few month (and that's just about my free OSS work, but my full-time paid work has been pretty full too) and why I didn't have much time for blogging!

Now that all those big tasks are mostly behind (even if I'll still continue to be busy with _fastlane_ and my talks, I expect to have a little more free time), it's about time I start writing again — starting with that part 2 of my capture semantics article that is long overdue!

In the meantime, I hope you'll enjoy all my hard work on my OSS tools and talks, and will see you soon for some technical articles again!
