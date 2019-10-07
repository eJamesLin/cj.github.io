---
layout: post
title:  "About typeof in Objective-C"
date:   2018-01-18 +0800
tags:   [iOS, Objective-C]
---

I want to share something about my misunderstanding about `typeof`.

The story goes from my [pull request](https://github.com/mopub/mopub-ios-sdk/pull/209) to mopub repo.
I found following code occurs in a block. 
It seems not using `weakSelf` for weak-strong dance, but capturing `self` by the `__typeof__(self)`.

```swift
__strong __typeof__(self) strongSelf = weakSelf;
```

So I create a pull request and change it to

```swift
__strong __typeof__(weakSelf) strongSelf = weakSelf;
```

In fact, my PR had a mistake. The `typeof`, `__typeof` and `__typeof__` are compile-time flag. Thus `self` in this block is not retained.

Another thing is that, `__typeof__(weakSelf)` is not a good pattern.
This will carry `__weak` attribute, thus we need an explicit `__strong` attribute to overwrite it.
Thus using `__typeof__(self)` is better, to prevent accidentally creating a `strongSelf` which is actually not.
