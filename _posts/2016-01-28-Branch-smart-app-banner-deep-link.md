---
layout: post
title:  "Use Branch.io to implement App Banner and Deep Link"
date:   2016-01-28 01:05:00 +0800
tags:   [iOS]
---

這篇網誌主要是說明為何應該使用 Branch 來實作 Smart App Banner 及 Deep Link，以及 Web 端的設定方法。

### Smart App Banner Intro
Smart App Banner 能提高網頁到 App 的轉換率，如下方附圖。

按下 banner 動作後
 - 如果沒有安裝 App，跳轉到 AppStore 以便使用者進行下載。
 - 如果已安裝 App，則跳轉打開 App

![](https://developer.apple.com/library/ios/documentation/AppleApplications/Reference/SafariWebContent/Art/smartbanner_2x.png)

<!--more-->

### Deep Linking
在處理跳轉進App時，除了打開相對應的 Custom URL Scheme之外，還可以帶相對應的參數與路徑，讓App跳至指定畫面。
例如 `mamilove://qa/question/38729`，在手機瀏覽器點擊這個鏈結將可以跳轉至 [媽咪愛App](https://mamilove.com.tw/app) 並呈現到相對應的問題討論頁面。

### Universal Links
跳轉進App的方式，除了上面的Custom URL Scheme之外，iOS9 推出了新的Universal Link，具有以下優點
 - Custom URL Scheme 在沒有安裝App時無法打開會顯示錯誤，而Universal Link是http鏈結，可跳轉進相對應的AppStore下載頁面
 - 不同的App如果定義相同的Custom URL Scheme會有衝突，而Universal Link則去跟你提供的server查詢對應的獨一無二的AppID，所以可打開正確的App。([具體原理及apple-app-site-association設定參考](https://blog.branch.io/how-to-setup-universal-links-to-deep-link-on-apple-ios-9))

### iOS9.2 Safari 處理 Custom URL Scheme 的問題
手機瀏覽器其實是無法得知使用者是否有安裝App的，所以實作Smart Banner作法一般是如下：
直接打開App，然後如果Timeout後還沒跳轉，則跳轉到AppStore。
```js
window.location = 'imdb://title/tt3569230';
setTimeout(function() {
  window.location = 'itms-apps://itunes.apple.com/us/app/imdb-movies-tv/id342792525'
}, 250);
```
iOS9時，打開App時會多出一個惱人的Modal提示，要按下確認後才會真正打開App
<img class="center" src="http://user-image.logdown.io/user/9212/blog/9082/post/458926/bxPsfzpHSM6Fhffm3FUz_image02-2.png" alt="image02-2.png">
這還好，機車的是iOS9.2時，打開App時的Modal提示，從blocking變成non-blocking，跳出提示後javascript會繼續運行，造成的結果是Timeout的方式失效，所以即使有裝App，還是會導到AppStore，於是Smart Banner就不Smart了。可參考[Branch做的Demo影片](https://www.youtube.com/watch?v=OtyuQ5fX40s)

那Apple不是有做一個真正的[Smart Banner](https://developer.apple.com/library/ios/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html)嗎?
可是瑞凡，那只有iOS的Safari能用，iOS的Chrome 和 Android表示...

而Branch向蘋果反應以後得到回應如下，所以建議使用Universal Link了。So be it...
![](https://cdn2.hubspot.net/hub/480264/hubfs/AppleiOS9.2Response.png?t=1453876233775&width=640)

### Branch Universal Link 的優點
自己做Universal Link的方法，可參考[教學](https://blog.branch.io/how-to-setup-universal-links-to-deep-link-on-apple-ios-9)。
但如果使用Branch的服務，具有以下優點
- 步驟可簡化很多
- 兼容 iOS8 以下的手機版本
- 下載App後，可再導回App中對應網頁的內容，如下圖
![](https://cdn2.hubspot.net/hub/480264/hubfs/1_Blog/how_branch_improves_universal_links.png?t=1453915433517&width=980)

### Web實作方法
在Branch後台設定好後，就是一個GET Method這麼簡單。
```
GET https://bnc.lt/a/<branch_key>?AnyOptionalQueryParams
```
example: 在手機瀏覽器點擊此 <a href="https://bnc.lt/a/key_live_gol1gc7x428W2F8Zds9yrfoeAFlWFIlS?\$deeplink_path=qa/question/38729">媽咪愛 deep link demo</a> 連結 (當然得安裝新版App)

詳細官方文件：<https://dev.branch.io/references/http_api/#structuring-a-dynamic-deeplink>

### iOS實作方法
詳細步驟請參考：<https://dev.branch.io/recipes/branch_universal_links/ios/>

參考資料1：<https://blog.branch.io/how-to-setup-universal-links-to-deep-link-on-apple-ios-9>

參考資料2：<http://tech.glowing.com/cn/deferred-deep-linking-and-branch-sdk-in-ios/>
