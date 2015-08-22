---
layout: post
title: Swift enums for JSON parsing
excerpt: When your JSON contains string fields that can only contain a limited set of values, Swift enums and their ability to have String raw types can be very handy.
categories: swift enum json
---

I discovered Swift since some time now, and even if I haven&rsquo;t really used it in a production app yet, I really love playing around with it.

There is plenty of stuff to love about Swift, so in the upcoming articles I&rsquo;ll probably talk about a lot of them. But for now, let start with enums


## Enums for parsing JSON

One of the nice feature of enums is their ability to have a String raw type. This makes easy to &ldquo;convert&rdquo; from an enum to its String representation and vice-versa. This feature is particularly useful when parsing a JSON that comes from a WebService.

Let&rsquo;s imagine your WebService returns a JSON in which some fields can only contain a finite number of possible values. Like:


    [
        {
            "name": "Alice",
            "role": "Developer"
        },
        {
            "name": "Bob",
            "role": "Manager"
        },
        {
            "name": "Clark",
            "role": "Architect"
        }
    ]


Typically, the `role` field will only accept a finite list of values, like or `Developer`, `Manager`, `Architect`, that could typically be bound to an enum once transformed in your DataModel.

Well, in Swift that's easy, simply define an enum with a String raw type that will match the string in the JSON:

    enum Role : String {
      case Developer = "Developer"
      case Manager = "Manager"
      case Architect = "Architect"
    }

If you're using Swift 2.0, you can even omit to explicitly define the value if its the same as the string representation of the case name like here, and Swift 2.0 will automatically compute and associate the corresponding string!

Then to convert the `let fieldValue = "Developer"` string extracted from your JSON, simply use `Role(rawValue: fieldValue)` and you're done!
Even better, this works in the other direction as well, having a `Role` enum value that you need to insert into your JSON is a simple matter of `Role.Developer.rawValue`.

So what we were doing typically using a huge `switch/case` in ObjC to do that mapping, or using an `NSDictionary` to map the string to the enum (but that would only work in one way) can now be done natively and in both directions in Swift!


So doing a model type that now handle each entry in our JSON is easy now:

    struct Person {
        var name: String
        var role: Role
    }

    let staff = [
        Person(name: "Alice", role: .Developer),
        Person(name: "Bob", role: .Manager),
        Person(name: "Clark", role: .Architect),
    ]

    extension Person {
        var toJSONDict : [NSObject:AnyObject] {
            return ["name":name, "role":role.rawValue]
        }
    }

    let jsonDict = staff.map { $0.toJSONDict }
    let jsonData = try! NSJSONSerialization.dataWithJSONObject(jsonDict, options: [])
    let jsonString = NSString(data: jsonData, encoding: NSUTF8StringEncoding)

    print(jsonString)
    // [{"role":"Developer","name":"Alice"},{"role":"Manager","name":"Bob"},{"role":"Architect","name":"Clark"}]


That's all for today, but rest assured that Swift enums have a lot more to offer and we'll surely talk about more advanced usages later in there!