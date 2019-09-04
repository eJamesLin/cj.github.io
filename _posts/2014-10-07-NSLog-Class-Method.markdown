---
layout: post
title:  在 NSLog 中印出所在的 class 及 method
date:   2014-10-07 23:44:00 +0800
tags:   [iOS, Objective-C]
toc:    false
pinned: false
---

```objc
NSLog(@"<%@:%@:%d>", NSStringFromClass([self class]), NSStringFromSelector(_cmd), __LINE__);
NSLog(@"%s", __PRETTY_FUNCTION__);
```
