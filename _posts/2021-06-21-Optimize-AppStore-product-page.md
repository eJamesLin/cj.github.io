---
layout: post
title:  "[WWDC Note] Get ready to optimize your App Store product page"
date:   2021-06-21 +0800
tags:   [iOS, WWDC]
---

在 WWDC 2021 中，有發佈未來 AppStore 將有的新功能。也許跟 Developer 無直接相關，但從產品推廣的角度上很有幫助。分別為
- Custom product pages
- Product page optimization

![](/assets/2021/appstore-product-page.png)

## Custom product pages
![](/assets/2021/appstore-product-page1.png)

除了預設的產品頁之外，可以設計額外的 custom product page，去客製化預覽影片，螢幕截圖，都可以做語言對應。

從搜尋或瀏覽來的使用者，會進到預設產品頁，而從專屬的連結進來的使用者，則會連到客製化的產品頁。

然後可以分析在這樣的情況下各自的
- Impression
- Downloads
- Conversion rate
- Retention
- Average proceeds per paying user

最多可以建立 35 個客製化產品頁，而且此流程不需要重新上傳 App，只需要審核產品頁。

## Product page optimization
而在預設的產品頁中，可以設定不同組合，或稱 bucket or A/B testing，來優化下載轉換率。

如圖中，右邊的紅色 App Icon 有較高的轉換率，則可以考慮修改產品頁。
![](/assets/2021/product-page-optimization.png)

能夠變動的部分，有 App Icon，螢幕截圖及預覽影片。

最多可以做出三個 treatments 組合，然後設定各自出現在使用者目前的機率，圖中設定機率為各 10%

![](/assets/2021/product-page-optimization-percentage.png)

App Icon 是重要的使用者體驗，也會是要不要打開 App 的重要關鍵。從 A/B testing 的分眾下載後的 App Icon，會維持著跟當初 AppStore 產品頁組合一致的樣式。
![](/assets/2021/product-page-optimization-icon.png)

可以指定針對特定語系的 AppStore 產品頁做優化。

要開一組新的 A/B testing 仍然要過審核，但不用上傳新的 binary。

如果要測試 App Icon 的不同組合，則要事先把所有的 Icon 放在 binary 中。最終預設的 App Icon 如果要更換的話，仍然要在下一個新版本的 binary 中更換。

最後，上述的 Custom product pages 以及 Product page optimization 都有支援 App Store Connect API.

能夠開 35 個 custom product page，手動處理會是場災難，善用 API 與自動化才能將效率提到最高。

## Reference
- [https://developer.apple.com/app-store/product-page-updates/](https://developer.apple.com/app-store/product-page-updates/)
- [https://developer.apple.com/videos/play/wwdc2021/10295/](https://developer.apple.com/videos/play/wwdc2021/10295/)
