---
layout: post
title:  "Put the ++ and -- operators back at Swift"
date:   2018-01-18 +0800
tags:   [iOS, Swift]
---

Swift had remove the `++` and `--` operators for the reason that it might be confusing to users.

Reference: [Proposal from Chris Lattner](https://github.com/apple/swift-evolution/blob/master/proposals/0004-remove-pre-post-inc-decrement.md)

```swift

extension Int {
    static prefix func --(i: inout Int) -> Int {
        i -= 1
        return i
    }
    static prefix func ++(i: inout Int) -> Int {
        i += 1
        return i
    }
    static postfix func --(i: inout Int) -> Int {
        let n = i
        i -= 1
        return n
    }
    static postfix func ++(i: inout Int) -> Int {
        let n = i
        i += 1
        return n
    }
}

```

Just saw this extension, now we get to use these handy operators yet again. :p
