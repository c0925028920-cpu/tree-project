# 手機版響應式設計規格

**日期：** 2026-05-31
**專案：** 台北市路樹倒塌風險查詢平台（樹適圈）
**目標：** 在現有 `index.html` 加入 `@media (max-width: 768px)` 響應式支援，讓 QR code 掃描後在手機上正常顯示

---

## 實作策略

**單一檔案修改**：只修改 `index.html`，在現有 CSS 末尾加入 media query 區塊，並在 `<body>` 加入手機專用 HTML 元素。所有 JS 邏輯（天氣 API、CSV 載入、篩選、路線規劃）完全不動。桌機版 CSS 完全不動。

---

## Header 手機版

| 項目 | 桌機 | 手機 |
|------|------|------|
| 高度 | 56px | 48px |
| 標題 | 完整標題 + 副標 | 「樹適圈」13px，副標 `display:none` |
| 天氣小卡 | 圖示 + 溫度 + 描述 + 地點（min-width 210px） | 只顯示圖示 + 溫度（`☁ 28°C`） |
| 路線按鈕 `#btn-route` | 顯示於 header | `display:none`（功能移至底部抽屜 Route tab） |
| 3D 按鈕 `#btn-3d` | 顯示於 header | `display:none`（功能移至地圖右下角 FAB） |

---

## 隱藏的桌機浮動元素

手機版以下元素 `display: none`：
- `#stats`（左上統計面板）
- `#filter-panel`（右上篩選面板）
- `#legend`（左下圖例面板）
- `#route-sidebar`（左側路線規劃側欄）

---

## 底部抽屜（`.mobile-sheet`）

### 三個狀態

| 狀態 | 高度 | 觸發方式 |
|------|------|---------|
| 收合 | 56px | 往下滑 / 初始狀態 |
| 半展開 | 45vh | 往上滑 / 點 tab / 點迷你統計列 |
| 全展開 | 85vh | 繼續往上滑 |

### 結構

```
.mobile-sheet
  .sheet-handle          ← 灰色拖拉橫條
  .sheet-mini-stats      ← 收合時顯示：● 高 N  ● 中 N  ● 低 N  ● 未健檢 N
  .sheet-tabs            ← Tab 列（半展開/全展開時顯示）
    .tab[data-tab=stats]    📊 統計
    .tab[data-tab=filter]   🔍 篩選
    .tab[data-tab=legend]   🗺 圖例
    .tab[data-tab=route]    🧭 路線
  .sheet-content
    #mobile-stats-content   ← 複製 #stats 的內容
    #mobile-filter-content  ← 複製 #filter-panel 的內容
    #mobile-legend-content  ← 複製 #legend 的內容
    #mobile-route-content   ← 複製 #route-sidebar 的內容
```

### 互動行為

- `touchstart` / `touchmove` / `touchend` 監聽拖拉手勢
- 拖拉距離超過 60px → 切換到下一個狀態（snap）
- 點任一 Tab → 自動展開到半展開，切換 tab 內容
- 抽屜本身 `pointer-events: auto`，抽屜**以外**的地圖區域仍可正常點擊

### 資料同步

手機版 sheet 內的篩選元素使用**獨立 DOM**（ID 前綴 `mob-`，如 `mob-chk-high`、`mob-sel-dist`），與桌機版並存。

- **`applyFilter()`**：根據 `window.innerWidth <= 768` 決定讀哪組 checkbox / select
- **統計數字**（`cnt-high` 等）：`updateCounts()` 同時寫入桌機版和手機版兩組元素（`cnt-high` + `mob-cnt-high`）
- **行政區切換天氣**：手機版 `mob-sel-dist` 的 `change` 事件同樣觸發 `loadDistrictWeather()`
- **路線規劃輸入**：手機版 route tab 的起點終點欄位使用新 ID（`mob-inp-from`、`mob-inp-to`），route 相關函式接受參數傳入而非直接讀固定 ID

---

## 地圖區調整

- **Popup 寬度**：手機上 `min-width: 85vw`（原 230px），接近全寬易讀
- **Popup 關閉按鈕**：點擊區域 44×44px（符合 iOS/Android 觸控最小建議）
- **Popup 自動 pan**：點擊樹木標記展開 popup 後，呼叫 `map.panBy([0, -120])` 將地圖往上移，避免 popup 被抽屜遮住

---

## 3D 模式 FAB

```html
<button id="fab-3d" class="fab-3d">3D</button>
```

- 位置：`position: fixed; right: 16px; bottom: 72px`（抽屜收合狀態上方）
- 樣式：圓形 44×44px，深色背景 + 藍色邊框，與現有 `#btn-3d` 相同 active 樣式
- 點擊觸發：與桌機版 `#btn-3d` 相同的 toggle 邏輯，共用同一個 JS handler

---

## 不修改的範圍

- 所有現有 JS 函式（`applyFilter`、`buildPopup`、`loadWeather`、`planRoute` 等）
- 桌機版 CSS（768px 以上完全不受影響）
- CSV 資料載入與天氣 API 邏輯
- GitHub Pages 網址與 QR code

---

## 實作檔案

| 檔案 | 變更類型 | 說明 |
|------|---------|------|
| `index.html` | 修改 | 在 `</style>` 前加入 media query CSS；在 `</body>` 前加入 `.mobile-sheet` 和 `#fab-3d` HTML；在現有 JS 末尾加入 touch handler 與 tab 切換邏輯 |
