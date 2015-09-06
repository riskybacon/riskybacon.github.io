---
layout: post
title: Swift Error - Cannot Subscript Value
tags:
 - swift
---

This is part of my series of Swift Error blog posts where I discuss errors that I couldn't easily find a solution for.

## Error Message

Cannot subscript a value of type `'[String : AnyObject]?'` with an index of type `'String'`

## Discussion

This did not make any sense to me. I read this as not being able to use a String to index into a dictionary that has a key type of String.

Here's the code:

```swift
let userDefaults = NSUserDefaults.standardUserDefaults().persistentDomainForName(NSGlobalDomain)
let style = userDefaults["AppleInterfaceStyle"]
```

The problem is subtle. `NSUserDefaults.persistentDomainForName(String)` returns `[String : AnyObject]?`. This is an optional and subscripting an optional is not allowed. The solution is to unwrap the optional, then subscript:

```swift
if let userDefaults = NSUserDefaults.standardUserDefaults().persistentDomainForName(NSGlobalDomain) {
    let style = userDefaults["AppleInterfaceStyle"]
}
```

## Versions

* Xcode 7 beta 6
* Swift 2
* OS X Yosemite 10.10