---
layout: post
title: "A Swifty Networking Architecture"
categories: swift, network, architecture
published: false
---

Discussing about how to make a great networking architecture in Swift, leveraging some patterns unique to Swift.

* `Model` are `structs`
* `SerializationService` (pros and cons)
* `Request<T>` + `parsing` closure
* `NetworkStack`
* …

Use of Generics for Requests

(Will be an open discussion, not an absolute solution)
