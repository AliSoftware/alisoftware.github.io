---
layout: post
title: "Pattern Matching, Part 3: ranges & custom pattern matching"
date: 2016-04-20
categories: swift
---

## Declaring our own pattern matching operator

Actually works with everything implementing the ~= operator

1900..<2000 ~= 1950

1900..<2000 ~= book

## Syntactic sugar

Using x? as a syntactic sugar for .Some(x)
