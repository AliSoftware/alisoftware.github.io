---
layout: post
title: "YAML Superpowers, part 1: JSON is YAML"
permalink: yaml-1-json
date: 2021-08-17
tags: ci yaml
---

This post is part 1 of a series of posts where I plan to focus on little-known features of YAML like multiline string processing, aliases and anchors, base64 support, tags and more.

## Why all those posts? YAML is not that complicated!

Amongst other things, YAML is widely used as a configuration format for different tools, including configuration files for tools like [SwiftLint](https://github.com/realm/SwiftLint) or [SwiftGen](https://github.com/SwiftGen/SwiftGen), but also notoriously to configure most of the Continuous Integration providers, like [GitHub Actions](https://github.com/features/actions), [CircleCI](https://circleci.com), [BuildKite](https://buildkite.com), and many more.

But most people only know the basics of YAML, while **many less-known features of this format are powerful and could prove very useful** to improve your config files readability and ease of maintenance.

In this series, I'll focus on examples related to configuring a CI like BuildKite (because that's what we are currently migrating to where I work at [Automattic](https://automattic.com)) but those features apply to other CI provides and use cases of YAML.

### Back to Basics

I'm sure most of you already know the basics of the YAML syntax; they are not that complicated after all. But even about the basics, it's worth making sure we're all on the same page first:

 - YAML is a format to represent structured data. It can especially represent dictionaries (what YAML calls "maps"), lists/arrays (what YAML calls "sequences"), and literals (like strings, numbers, booleans, …)
 - One thing that most people might not know is that **YAML is a superset of JSON**. This means that any JSON is a valid YAML file! YAML just extends the JSON syntax to provide more features (like comments etc) and alternatives to represent the same data structures.

For example, in YAML, you often see list (aka "sequence" in YAML parlance) represented like this:

```yaml
 - "item1"
 - "item2"
 - "item3"
 - "etc"
```
But did you know you could also represent a list using the JSON "square brackets" syntax too?

```yaml
["item1", "item2", "item3", "etc"]
```
That's right, that syntax is the same as what you use to represent arrays in JSON. And it's no coincidence, because JSON is a subset of YAML, so that's also valid YAML!

Likewise, you often see dictionaries (aka "maps" in YAML parlance) represented like this in most YAML files:

```yaml
"key1": "value1"
"key2": "value2"
"key3": "value3"
```

But another, totally valid way to represent a map in YAML is to use this alternative syntax, which is the syntax you're used to in JSON already:

```yaml
{ "key1": "value1", "key2": "value2", "key3": "value3" }
```

Both those syntaxes represent the same thing; you can use whichever syntax you want when writing your YAML files. You can even write your file with purely JSON-compatible syntax and add it a `.yml` extension, and that would also be valid and be accepted by any YAML parser. Just try it: take any JSON file that you might have around and paste its content to [a YAML linter](http://www.yamllint.com) and it will gladly accept it!

What makes YAML usually more attractive as a config file format over JSON is that those alternative syntaxes are meant to make the structure more human-readable than JSON (by representing lists a bit like bullet points, etc), while JSON is meant to be more machine-oriented.

The fact that it also allows adding comments, unlike JSON[^1], and that quotes around strings are optional[^2], also helps make your YAML config files easier to read and write for a human.

### A full example

At that point I feel it can also be useful to take a concrete examples, especially because when used in the context of CI config files, YAML structures can become a bit more complex and specific.

For example, it's common in most CIs to have YAML nodes that appear as "lists of **single-key dictionaries**", which might be confusing at first – and not always straighforward to realize what those really are at first glance. Those look like regular dictionaries, but are not. This pattern of "sequence of single-key dictionaries" is in fact the way YAML represents an **ordered map** (while a regular dictionary/map is unordered by definition). Here's an example:

```yaml
steps:
  - label: "Build the app"
    key: build
    plugins:
      - automattic/bash-cache#v1.5.0:
          bucket: "a8c-cache"
      - automattic/git-s3-cache#v1.1.0:
          bucket: "a8c-repo-mirrors"
          repo: "wordpress-mobile/wordpress-ios/"
    env:
      IMAGE_ID: xcode-12.5.1
    command: "bundle exec fastlane build_for_testing"
  - label: "Run Tests"
    key: test
    plugins:
      - automattic/bash-cache#v1.5.0
      - automattic/git-s3-cache#v1.1.0:
          bucket: "a8c-repo-mirrors"
          repo: "wordpress-mobile/wordpress-ios/"
    env:
      IMAGE_ID: xcode-12.5.1
    command: "bundle exec fastlane tests"
```

This extract of a typical BuildKite config file[^3] defines:

 - A top-level dictionary with the key `steps`. The value associated with that key is an array of 2 elements – as you can see by the two `-` that are at indentation level 2
 - Each of these 2 items is a dictionary, which both happen to contain the same 5 keys:  `label`, `key`, `plugins`, `env` and `command`.
 - The value for the `label`, `key` and `command` keys are strings in both cases.
 - The value of the `env` key is itself another dictionary (map), with a single `IMAGE_ID` key.
 - The value of the `plugins` key is what might look the most unusual, as it is such a so-called ordered map, aka an _array of single-key dictionaries_.
   - In fact, for the first "step" described in this YAML, the value for `plugins` is an array of 2 items, each of them being a dictionary with only a single key – so deep down it's in fact 2 single-key dictionaries and not a single dictionary with 2 keys  as one might think… even if in practice for all intents and purposes you will probably read it as an ordered dictionary with 2 keys for interpretation of the config file.
   - The first single-key dictionary has the key `automattic/bash-cache#v1.5.0`, and its value is yet another dictionary which happens to only have the `bucket:` key
   - The second single-key dictionary has the key `automattic/git-s3-cache#v1.1.0` and its values is yet another dictionary, this time with 2 keys.
   - For the second step though (the one to configure the test step), the value of the `plugins` key is in fact an array of mixed value, the first one being a single string, while the second one being a single-key dictionary like in the first step.

These 2 ways of listing the various `plugins` in a CI `step` is actually common in most CIs (this is an example from a BuildKite config file, but e.g. CircleCI has similar use cases of arrays of mixed types, with Strings and single-key dictionaries too).

This is a common way in YAML to describe an ordered list of items (here BuildKite plugins) while allowing some of them to define "options" (by making the plugin "name" the key of a single-key dictionary, and providing the options as the value for that key), while others might not need any option (and most CI config syntaxes allow you to use simple strings with the plugin "name" for those cases[^4])

Already having your head spinning a bit? Yeah, that's why I wanted to make sure we were all on the same page on with twisted YAML structures even before going further to introduce lesser known features 😅.

### What's Next: Advanced features of YAML

Ready to go further? In addition to all that above, YAML also provides quite powerful features that will be the focus of the next parts of this article series. Here's our program!

 - Multiline strings (part 2), including various ways to process indentation
 - Anchors & aliases (part 3), to avoid repeating yourself
 - Merging dictionaries (part 4), which is especially useful when used with anchors
 - Hex/Binary/Octal numbers, Booleans and the Null value (part 5)
 - Tags (part 6), to add type information and help disambiguate values
 - Base64 representation of binary data (part 7)

There's even more to YAML (I'll probably not got into preprocessor directives, defining your own tags, or putting multiple YAML documents into a single YAML file, etc), but this will hopefully already give you an extensive enough tour to what will likely be the features you might find the most useful in the context of using YAML as config files.

See you in part 2!

[^1]: Technically [JSON5](https://json5.org), which is a successor of JSON, does support comments; but the more common JSON that you've seen around for ages and that we still find in most places doesn't.

[^2]: This is why in most cases people omit quotes in both single string literals, but also in arrays of strings and dictionary keys. Quotes are still required/useful if disambiguation is needed, e.g. using `"42"` to represent the _string_ "42" as opposed to the number 42.

[^3]: Borrowed from our Wordpress-iOS source code [here](https://github.com/wordpress-mobile/WordPress-iOS/blob/develop/.buildkite/pipeline.yml)

[^4]: In most CIs, including BuildKite, it would also be valid to use a single-key dictionary with a `null` value for items which don't need options to be provided as values, instaed of using a single String. So we could have used `- automattic/bash-cache#v1.5.0: null` as well here. But let's keey talk about null values for part 5 of this article series.