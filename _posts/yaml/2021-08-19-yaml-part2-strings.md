---
layout: post
title: "YAML Superpowers, part 2: Multiline Strings"
tags: ci yaml
category: yaml
---

This is part 2 of a series of post about lesser-known features about YAML – especially the ones useful in contexts like CI or tools' config files. In this post we will discover the many ways YAML can represent strings, including multiline strings, keeping or stripping indentation, and more!

<p class="small"><a href="/yaml/2021/08/17/yaml-part1-json">Missed part 1? head here!</a></p>

## Simple strings ("Flow" scalars)

Let's start easy with single-line strings. It's already worth noting that, in YAML:

 - When using double quotes `""` around your strings, it allows you to also escape some special characters using the `\` sigil inside that string (see below).
 - When you use single quotes `''`, the string won't be escaped. This means that every character will be interpreted verbatim – including `\`. The only exception is if you need to include a literal `'` inside a single-quoted string, in which case you can repeat it (i.e. use `''` inside the single-quoted string)
 - Quotes can be omitted around strings, unless it's ambiguous to do so. Omiting quotes is the same as using single quotes (i.e. characters won't be escaped)

When you use double-quotes, you can then use escape codes for special characters, like the following ones:

 - `"\x12"`, `"\u1234"` or `"\U00102030"` for 8, 16 and 32 bits unicode codepoints respectively
- `"\\"` (same as unescaped `'\'`) and `"\""` (same as `'"'`)
- `"\r"` and `"\n"` for CR and LF, respectively, and `"\t"` for tab
- `"\_"` for a non-breaking space (NBSP)
- You can also put a `\` at the end of a line (before your newline character) to escape that newline and thus ignore it. This is a nice trick for manually wrapping long strings, though the block scalar syntax below is usually way nicer to use.
- And some more (see the spec or [refcard](http://yaml.org/refcard.html))

This means that the strings in the following array of strings are all valid:

```yaml
 - Hello
 - 'Hello'
 - "Hello"
 - '"Hello", they said.'
 - '# this is not a comment but a string'
 - "Smile \u263A"
 - "column1\tcolumn2\tcolumn3\n"
 - "\x0d\x0a is the same as \r\n, namely CRLF"
 - 'Single quotes means \ is not an escape character ¯\_(ツ)_/¯' 
 - 'Except there''s still a way to include the single quote character into those.'
```

## Multi-line blocks and long strings

YAML also supports writing strings on multiple lines in your YAML, with two different styles: literal and folded. Those are represented by a `|` (for literal style) or `>` (for folded style) header line (see examples below), and we will go into their differences below.

But first, note that any leading indentation in those multi-line blocks is stripped by YAML, so you don't have to worry about this; in fact, let's talk about this first before going further.

### Indentation in multi-line blocks

In a multi-line block, YAML first determines the indentation level of the whole block:

 - Either by guessing it from the number of leading spaces of the first non-empty line, if you don't specify anything (most common case)
 - Or by using the explicit value you provide after the `|` or `>`, to indicate the number of *additional* spaces compared to the parent node.

The leading spaces corresponding to (implicit or explicit) indentation are then ignored in the final interpreted string.

```yaml
multi-line-yaml-blocks:
  examples:
    string1: >
      This is a long string, where the indentation
      was guessed to be 6 leading spaces (aka 2 more
      leading spaces than the parent node). Also note that
      this long string will be folded into a single-line string with no newline.
    string2: >2
         This is another long string, where the indentation was explicitly
      indicated to be 2 more spaces than the parent node.
         Which means that the 1st and 3rd lines, which happens to start with
      5 additional leading spaces relative to the parent node's existing indentation,
      will start with 3 literal space characters (as the first 2 are the indentation).
```

In most cases you'll use implicit indentation (I've rarely seen the explicit indentation value being used in concrete use cases to be honest), but it's good to know you can make it explicit if needed, especially if your string is supposed to start with spaces that are not to be interpreted as indentation.


### Literal style: `|` – keep the newlines

When using the block syntax, the literal style, denoted by `|` header, is the simplest.

As the name suggests, it keeps the content as is, including rendering newlines as actual newlines in the end content. The only thing that is stripped is the leading indentation.

This is most useful when you want to embed strings with multiple lines in your YAML, typically like the content of a bash script with multiple command lines.

```yaml
steps:
  - label: "Build the app"
    key: "build"
    command: |
      echo "--- Install gems"
      bundle install
      echo "--- Build the app"
      bundle exec fastlane build
```

### Folded style: `>` – fold the newlines into spaces

On the other hand, the folded style, denoted by a `>` header, replaces newlines by a space character – unless it ends an empty or more-indented line.

This is very useful when you want to break long lines of text into smaller ones (i.e. manual hard-wrapping) for readability. Note that empty (or all-spaces) lines, as well as more-indented lines, are not affected by this folding.

```yaml
notify:
  - slack:
      channels: ["#build-notifs"]
      message: >
        Your build have failed. You might want to check your
        CI logs for more details about the failure, or ping
        your friendly neighbourhood Infrastructure Engineer
        on call to ask for help.
    if: build.state == "failed"
```

### Chomp mode

In addition to the style (literal `|` or folded `>`), and optional explicit indentation number, you can also specify a chomp mode in the block header. The chomp mode defines how YAML will interpret trailing newlines and spaces in your block, and can be one of the following:

 - "Clip" mode: this is the default behavior, the one used when you don't specify any specific chomp indicator in your header. In this mode, the final newline is preserved, but any additional trailing empty line are ignored.
 - "Strip" mode: indicated by a `-` in the block header, this will strip not only trailing empty lines like in clip mode, but also the final newline at the end of the string.
 - "Keep" mode: indicated by a `+` in the block header, this will keep both the final newline and any potential trailing empty lines too.

```yaml
examples:
  clip: >
    This content will end with a LF character
    but not include the final empty lines.
    
    
  strip: >-
    This content will neither contain a trailing LF character
    nor the trailing empty line.
    
  keep: >+
    This content will keep both the LF
    and the trailing empty lines.
    
    
equivalent-output:
  clip: "This content will end with a LF character but not include the final empty lines.\n"
  strip: "This content will neither contain a trailing LF character nor the trailing empty line."
  keep: "This content will keep both the LF and the trailing empty lines.\n\n\n"
```

### Block Comment

Every block syntax starts with a header (with `|` or `>`, optional indentation number, and optional chomp indicator). The only other thing that is also allowed on that header line is a single-line `# comment`; the rest (the content of the block literal) has to start on the next line.

Adding a single-line comment can be nice to provide some context for future you:

```yaml
notify:
  - slack:
      channels: ["#build-notifs"]
      message: > # This message will be folded, i.e. newlines will be replaced by spaces.
        Your build have failed. You might want to check your
        CI logs for more details about the failure, or ping
        your friendly neighbourhood Infrastructure Engineer
        on call to ask for help.
    if: build.state == "failed"
```

## Conclusion

And here you thought strings would be a pretty simple topic in YAML… 😅

In this post we saw how YAML allows us to easily represent long strings, as well as strings containing special characters and newlines, without having to escape all the newlines manually, and making those long, potentially-multiline strings to be way more readable.

I hope you learned something and you will be able to put this to good use in your YAML config files 😊

Next up will be an even more powerful and lesser-known concept of YAML: anchors and aliases. Meet me in part 3 to discover how you can avoid repeating yourself, and how to mutualise common, repeated values in your YAML files!
