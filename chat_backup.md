# 面試準備聊天紀錄備份 (Alan)

## 關卡一：行為面試與 STAR 故事
**Q: 與設計師意見分歧怎麼辦？**
A: 使用 Apple HIG (Human Interface Guidelines) 作為客觀標準。以「包裹卡片陰影」為例，設計師想拿掉陰影，但我以實機展示在深色模式下的層級問題 (Depth & Clarity)，並提出「彌散陰影」作為折衷方案。

**Q: 主動推動技術重構的經驗？**
A: 統一數網從 Flutter 轉回 iOS 原生。痛點在於 Flutter 第三方套件在 OS 升級時會引發 SDK 依賴地獄導致 Crash。轉為原生後，導入 MVVM 解耦核心包裹狀態邏輯，提升效能與穩定性。

**Q: 最大的技術錯誤與事故？**
A: Crashlytics 上的偶發 Crash。查出是背景 API 輪詢與 UI 同時讀寫造成的 Data Race。解法是導入 Swift Actor 隔離狀態，而非使用傳統 Lock (避免 Deadlock)。

**Q: 一人開發如何做 Code Review？**
A: 建立三道防線：1. CI/CD 掛載 SwiftLint 自動檢查排版。 2. 與 Android 工程師跨端對齊變數命名與邏輯簡化。 3. 技術主管做最後的商業邏輯與測試確認。

## 關卡二：系統設計 (Mobile System Design)
**Q: 體育直播與即時賠率 App 架構？**
*   **架構選型：** 主打 MVVM 混合開發，簡單頁面用 MVC。面對極度複雜狀態可提 TCA 作為未來升級彩蛋。
*   **資料流 (Data Flow)：** 背景 Custom Actor 接 Socket 解決寫入衝突 -> Combine/AsyncStream 降頻 (Throttle) -> @MainActor ViewModel 更新 UI。
*   **AVFoundation：** 捨棄 AVPlayerViewController，改用底層 AVPlayer + AVPlayerLayer 鋪底，方便 SwiftUI/UIKit 在上層疊加客製化賽況與賠率 UI。
*   **防範 OOM (記憶體管理)：**
    *   棄用 Closure 改用 `async/await`，天然消除 Retain Cycle。
    *   實作 雙層快取：LRU Memory Cache (NSCache) + Disk Cache (FileManager)。
    *   導入 Downsampling (降採樣)，在解碼階段將大圖縮小，避免撐爆 RAM。

## 關卡三：底層技術快問快答
**Q: weak vs unowned?**
A: 皆用於打破 Retain Cycle。`weak` 是 Optional，物件釋放後變 nil，非常安全。`unowned` 物件釋放後不會變 nil，若存取會 Crash (Fatal Error)。實戰一律優先用 `weak`。

**Q: Closure 與 async/await 差異？**
A: 舊版 Closure 容易造成嵌套地獄與忘記寫 `[weak self]` 導致 OOM。`async/await` 底層利用 Continuation 將非同步程式碼拉直，天然避免 Retain Cycle。

**Q: 為何不用寫 DispatchQueue.main.async？**
A: 因為 `@MainActor` 是 Global Actor，編譯器會在編譯期自動進行「執行緒路由 (Thread Routing)」，在 `await` 結束後強制將執行權切回主執行緒。

**Q: setNeedsLayout vs layoutIfNeeded?**
A: `setNeedsLayout()` 是異步的重繪標記 (Flag)。`layoutIfNeeded()` 是同步的強制立即重繪。在寫 AutoLayout 動畫時，需將 `layoutIfNeeded()` 放入 `UIView.animate` 閉包內。

**Q: 什麼是 Retain Cycle (循環強參照)？**
A: 兩個物件使用強參考 (預設 var) 互相抓著對方，導致 ARC 參考計數永遠無法降到 0，記憶體無法釋放。最常發生在 Closure 內部使用了 `self`，此時需用 `[weak self]` 打破循環。