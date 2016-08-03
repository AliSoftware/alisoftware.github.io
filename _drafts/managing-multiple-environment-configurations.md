---
layout: post
title: "Managing Multiple Environment Configurations in an Xcode Project"
categories: xcode
published: false
---


https://iosdevelopers.slack.com/archives/links/p1467748960000715

So for the record, our solution is to have:

- an "Environments" folder, with a "current" subfolder + one subfolder per environment (staging, prod, etc)
- a Build Script Phase that copies the content of the subfolder matching the env we want into the subfolder "current"
- only the content of the "current" subfolder is built with the target, other subfolders are not included in the Xcode target as build artefacts

Then in these folders we can put whatever we want to be different depending on the env, like an `AppIcon.xcassets`, an `Environment.swift` file defining some constants, etc.
And to define the environment we want to use, we use a custom Build Setting we call `APP_ENVIRONMENT`.

Lastly, we also have a custom build setting `APP_BUNDLE_IDENTIFIER` that can be based on that `APP_ENVIRONMENT` so it switches bundleIDs and different configs can be installed on the same phone

(we then make the `PRODUCT_BUNDLE_IDENTIFIER` of the app target depend on that Project-level `APP_BUNDLE_IDENTIFIER` build setting, and if we have a WatchApp or a Today Widget we can also derive their own `PRODUCT_BUNDLE_IDENTIFER` from `$(APP_BUNDLE_IDENTIFIER).watchapp` etc)

it also makes it super easy to switch environments with Fastlane, simply using `fastlane ota --env prod` and the `Fastfile` simply overrides that `APP_ENVIRONEMENT` and possibly also that `APP_BUNDLE_IDENTIFIER` directly from the command line if needs be.
