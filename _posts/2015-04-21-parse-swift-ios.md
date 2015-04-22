---
layout: post
title: Parse and Swift
---

This tutorial was written for experienced iOS developers with an
Objective-C background that want to write a Swift app that uses Parse.

## Versions

 * Xcode 6.3
 * Parse SDK 1.7.1
 * iOS 8.3

## Create Parse App

Go to [parse.com](https://parse.com) and create an app. Get an account if
you don't already have one. Make a note of your keys, you'll need them later

## Download SDK

Download the [Parse iOS SDK](https://www.parse.com/downloads/ios/parse-library/latest). Create a new project iOS Swift project, copy in the frameworks from the SDK download.

## Create Xcode Project

Open up Xcode, create new project. Make it an iOS Single View Application.
Set the language pop up menu to Swift.

## Add Frameworks

Copy in the following frameworks:

* Parse.framework
* Bolts.framework

Make sure to check the "Copy items if needed" checkbox. If you want Parse
support in your test target, check that box, too.

<img src = "/public/images/parse_swift/add_bolts_parse_frameworks.gif"/>

Add these Linked Frameworks and Libraries to your target:

* Accounts.framework
* Social.framework
* MobileCoreServices.framework
* Security.framework
* SystemConfiguration.framework
* AudioToolbox.framework
* libz.dylib
* libsqlite3.dylib
* libstdc++.dylib

## Create a Bridging Header

File -> New -> iOS -> Source -> Header File

Name it *YourProject-Bridging-Header.h*

Open up that file, replace the existing content with:

```swift
#import <Parse/Parse.h>
#import <Bolts/BFTask.h>
```

* Select your project, target, build settings
* Add YourProject/YourProject-Bridging-Header.h to the Objective-C bridging header 
* If you have compile errors, you may need to restart Xcode

<img src = "/public/images/parse_swift/add_bridging_header.gif"/>

## AppDelegate

In AppDelegate.swift, make your *application:didFinishLaunchingWithOptions:* function look like:

```swift
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    let parseAppId     = "your_parse_app_id"
    let parseClientKey = "your_parse_client_key"
        
    Parse.setApplicationId(parseAppId, clientKey: parseClientKey)
        
    var testObject = PFObject(className: "TestObject")
    testObject["foo"] = "bar"
    testObject.saveEventually()

    return true
}
```

## Run It

Run the code, head over to Parse and check to see if a table called TestObject exists with your test object.

