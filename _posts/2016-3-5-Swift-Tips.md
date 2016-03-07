---
layout: post
title: Swift Tips
comments: true
---

# Lazy Initialization

Lazy Initialization is a technique for delaying the creation of an object or some other expensive process until it's needed.

Format like following:

```swift
lazy var players = [String]()

// If you want to add some logic
lazy var players: [String] = {
    var startWithJohn = [String]()
    startWithJohn.append("John") 
}()
```
