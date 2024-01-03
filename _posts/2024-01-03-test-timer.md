---
layout: post
title: 'Strategies of Testing Timers'
categories: [iOS dev]
tag: [swift, protocol, mock, unit test, timer]
---

Time flies. Since we iOS developers have already suffered the slowness of Xcode, there is no reason to waste more valuable time on other time-consuming things, such as unit tests. Unit tests should be fast, ideally, but sometimes we have to address the cases using timer API. For example, the [lecture](https://developer.apple.com/videos/play/wwdc2018/417) in WWDC18 introduced a scenario that the App change locations every 10 seconds. 

```swift
func testScheduleNextPlace() {
    let manager = FeaturedPlaceManager()

    let beforePlace = manager.currentPlace
    manager.scheduleNextPlace()

    RunLoop.current.run(until: Date(timeIntervalSinceNow: 11))

    XCTAssertNotEqual(manager.currentPlace, beforePlace)
}
```

