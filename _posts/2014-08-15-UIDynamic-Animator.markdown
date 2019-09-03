---
layout: post
title:  UIDynamic Animator
date:   2014-08-15 15:24:00 +0800
---

Set up physics relating animatable objects and let them run until they resolve to stasis
Easily possible to set it up so that stasis never occurs, but that could be performance problem

```objc
//Create a UIDynamicAnimator
UIDynamicAnimator *animator = [[UIDynamicAnimator alloc] initWithReferenceView:aView];

//Add UIDynamicBehaviors to it (gravity, collisions, etc.)
//  gravity
UIGravityBehavior *gravity = [[UIGravityBehavior alloc] init];
[animator addBehavior:gravity];

//  or collision
UICollisionBehavior *collider = [[UICollisionBehavior alloc] init];
[animator addBehavior:collider];

//Add UIDynamicItems (usually UIViews) to the UIDynamicBehaviors
id <UIDynamicItem> item1 = ...;
id <UIDynamicItem> item2 = ...;
[gravity addItem:item1];
[collider addItem:item1];
[gravity addItem:item2];
```
The items have to implement the UIDynamicItem protocol ...
```objc
@protocol UIDynamicItem
@property (readonly) CGRect bounds;
@property (readwrite) CGPoint center;
@property (readwrite) CGAffineTransform transform;
@end
```
UIView implements this @protocol.

If you change center or transform while animator is running, you must call UIDynamicAnimator’s
```objc
- (void)updateItemUsingCurrentState:(id <UIDynamicItem>)item;
```

#### UIDynamicBehavior
Superclass which is inherited by the primitive dynamic behavior classes: `UIAttachmentBehavior`, `UICollisionBehavior`, `UIGravityBehavior`, `UIDynamicItemBehavior`, `UIPushBehavior`, and `UISnapBehavior`.

可以客制化subclass combination
```objc
- (void)addChildBehavior:(UIDynamicBehavior *)behavior
```
behaviors可以知道自己位於哪個 `UIDynamicAnimator`
```objc
@property UIDynamicAnimator *dynamicAnimator;
```

This get called when the dynamic behavior is added to, or removed from, a dynamic animator.
```objc
- (void)willMoveToAnimator:(UIDynamicAnimator *)dynamicAnimator
```
**Parameters** : `dynamicAnimator`
The dynamic animator that the behavior is being added to, or 若被animator移除: `nil`.

#### UIDynamicBehavior 發生的瞬間
但通常不會放太多的運算在裡面
```objc
uiPushBehavior.action = ^{
	NSLog(@"action block");
};
```

#### References
[http://furnacedigital.blogspot.tw/2013/10/uikit-dynamics.html](http://furnacedigital.blogspot.tw/2013/10/uikit-dynamics.html)
[http://furnacedigital.blogspot.tw/2013/10/uikit-dynamics_17.html](http://furnacedigital.blogspot.tw/2013/10/uikit-dynamics_17.html)
