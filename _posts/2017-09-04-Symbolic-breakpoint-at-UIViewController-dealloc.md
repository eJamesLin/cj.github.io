---
layout: post
title:  Symbolic breakpoint at UIViewController dealloc
date:   2017-09-04 20:53:00 +0800
tags:   [iOS]
---

實用 Symbolic breakpoint, 對於抓出 memory leak 很有幫助。


```
--- dealloc @(id)[$arg1 description]@ @(id)[$arg1 title]@
```

![](/assets/2017/symbolic-breakpoint-dealloc.png)


Reference: <https://twitter.com/0xced/status/900692839557992449>
