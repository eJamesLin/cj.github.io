---
layout: post
title:  Solve the issue about cannot open iOS simulator in Android Studio
date:   2020-04-22 18:17:00 +0800
tags:   [iOS, Flutter]
---

Start using Flutter in our new project. Record some of the issues and solutions.

One of the weird things that happened recently, is that although iOS simulator is opened, it still cannot be selected as deploy target in Android Studio. The opened iOS simulator not shown in the list.

![](/assets/2020/AndroidStudio-iOS-Simulator.png)

At this time, try to check the simulator version, with the current selected Xcode version.

```
xcode-select -p                                                        
/Applications/Xcode-11.3.app/Contents/Developer
// Should be same as opened simulator version as below image.
```

![](/assets/2020/iOS-Simulator.png)

