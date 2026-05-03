# 樹適圈平台開發文件

> 給組員接手修改用的說明文件  
> 最後更新：2026-05-03

---

## 快速啟動

```bash
cd ~/Desktop/tree_project
python3 -m http.server 8080
# 開瀏覽器打開 http://localhost:8080/index.html
```

> **注意**：index.html 直接用瀏覽器打開（file://）會因為跨來源限制無法讀取 CSV，一定要透過本機伺服器。

---

## 檔案結構

```
tree_project/
├── index.html                   # ★ 平台主程式（唯一需要改的 HTML 檔）
│
├── tree_integrated_wgs84.csv    # 行道樹主表（92,927 筆，WGS84 座標）
├── brownrot_taipei.csv          # 褐根病通報（5,420 筆）
├── protected_standalone.csv     # 獨立列管保護老樹（2,994 棵）
├── tree_pits_wgs84.csv          # 樹穴資料（2,254 筆，已轉 WGS84）
│
├── brownrot_trend.csv           # 褐根病年度趨勢（統計圖用）
├── budget.csv                   # 公園處預算對比（統計圖用）
├── collapse_cause.csv           # 路樹倒塌原因（統計圖用）
├── compensation.csv             # 各縣市國賠（統計圖用）
└── institution.csv              # 22 縣市制度完備度（統計圖用）
```

### 資料欄位速查

**tree_integrated_wgs84.csv**（地圖主圖層）

| 欄位 | 說明 |
|------|------|
| `tree_id` | 樹木編號（如 BT0614021096） |
| `dist` | 行政區（如 士林區） |
| `region` | 道路名稱 |
| `tree_type` | 樹種名稱 |
| `diameter` | 胸徑（公分） |
| `tree_height` | 樹高（公尺） |
| `health_風險結果` | 官方風險等級（高/重/中/低/D級） |
| `health_健檢扣分` | 健檢扣分（負數，越低越危險） |
| `health_健檢關鍵因子` | 扣分原因描述 |
| `health_樹木狀況` | P=正常，I=異常 |
| `is_protected` | TRUE/FALSE，是否為列管保護樹木 |
| `lat` / `lng` | WGS84 緯度 / 經度 |

---

## index.html 結構總覽

```
<head>
  ├── Leaflet CSS
  ├── Google Fonts（Orbitron、Share Tech Mono）
  └── <style>  ← 所有 CSS 在這裡

<body>
  ├── #scanline-overlay        掃描線覆蓋（載入時顯示）
  ├── #weather-alert           氣象警告橫幅（有警報才出現）
  ├── #header                  頂部導覽列
  │     ├── h1                 標題
  │     ├── #btn-route         路線規劃按鈕
  │     └── #weather-widget    天氣小卡
  │
  └── #map-container
        ├── #map               Leaflet 地圖
        ├── #loading           載入遮罩
        ├── #stats             風險統計面板（左上）
        ├── #filter-panel      顯示篩選面板（右上）
        ├── #route-sidebar     路線規劃側欄（左側滑出）
        ├── #btn-locate        浮動定位按鈕（右下）
        └── #legend            圖例面板（左下）

<script>
  ├── CONFIG                   所有設定值
  ├── 風險邏輯                  getRiskLevel、RISK_META
  ├── 地圖初始化
  ├── 行道樹圖層               loadTrees、buildPopupHtml
  ├── 篩選控制                  applyFilter
  ├── 天氣功能                  loadWeather 系列
  ├── 路線查詢                  handleRouteSearch 系列
  ├── 使用者定位                locateUser
  ├── 褐根病圖層               loadBrownrot
  ├── 保護老樹圖層              loadProtectedTrees
  └── 樹穴圖層                  loadTreePits
```

---

## 各功能說明

### 1. 行道樹主圖層

**相關函式：** `loadTrees()`、`buildPopupHtml(row, level)`、`getRiskLevel(row)`

**風險等級對照：**

```javascript
// index.html 約第 724 行
function getRiskLevel(row) {
  const r = (row['health_風險結果'] || '').trim();
  if (r === '高' || r === '重')   return 'high';  // 紅色脈衝動畫
  if (r === '中')                  return 'mid';   // 橘色圓點
  if (r === '低' || r === 'D級' || r === '低風險' || r === '極低風險') return 'low';
  return 'none';  // 灰色，載入到 allMarkers 但預設不顯示
}
```

**標記樣式：**

```javascript
const RISK_META = {
  high : { color: '#ff4757', radius: 8  },  // 改這裡換顏色
  mid  : { color: '#ffa502', radius: 6  },
  low  : { color: '#2ed573', radius: 4  },
  none : { color: '#6e7681', radius: 3  },
};
```

**高風險樹改用脈衝動畫 DivIcon**，中低風險用一般 `circleMarker`。

**無健檢資料（`none`）的樹：** 資料有載入到 `allMarkers` 陣列，但 `marker.addTo(map)` 沒有執行，所以地圖上看不到。未來要做搜尋功能可以直接從 `allMarkers` 取用這批資料。

---

### 2. 篩選功能

**相關函式：** `applyFilter()`

**新增篩選條件：** 在 HTML 的 `#filter-panel` 加 checkbox，然後在 `applyFilter()` 的判斷邏輯裡加對應條件即可。

```javascript
// index.html 約第 936 行
function applyFilter() {
  const showHigh = document.getElementById('chk-high').checked;
  const showMid  = document.getElementById('chk-mid').checked;
  const showLow  = document.getElementById('chk-low').checked;
  // 在這裡加新的 checkbox 讀取

  allMarkers.forEach(({ marker, level, dist }) => {
    const levelOk = (level === 'high' && showHigh) || ...;
    const distOk  = !selDist || dist === selDist;
    if (levelOk && distOk) marker.addTo(map);
    else marker.remove();
  });
}
```

---

### 3. 天氣功能

**相關函式：** `loadWeather()`、`displayWeather()`、`checkAlerts()`、`updateWeatherForDistrict()`

**API 使用：**

| API | 用途 |
|-----|------|
| `O-A0001-001` | 臺北氣象站即時觀測（溫度、風速、天氣） |
| `O-A0003-001` | 全台北自動站（行政區切換天氣） |
| `W-C0033-001` | 警特報（觸發紅色警告橫幅） |

**換氣象站：** 修改 `CONFIG.stationId`（目前是 466920 臺北站）。

**調整強風警告門檻：** 修改 `CONFIG.windAlert`（目前 10.8 m/s = 蒲福 6 級）。

---

### 4. 路線規劃

**相關函式：** `handleRouteSearch()`、`analyzeRouteRisk()`、`renderRouteResult()`

**路線沿途風險分析：** `analyzeRouteRisk(coords)` 對路線每個座標點，找 `allMarkers` 中距離 40 公尺內的樹木。

**調整緩衝距離：**

```javascript
const ROUTE_BUFFER_M = 40; // 改這裡，單位：公尺
```

**使用 OSRM 免費路線 API**，支援 `foot`（步行）、`driving`（開車）、`bike`（騎車）。

---

### 5. 使用者定位

**相關函式：** `locateUser(fillRoute)`

**兩個入口：**
- 地圖右下角 `#btn-locate` 按鈕：只定位並移動地圖
- 路線側欄 `#btn-use-location` 按鈕：定位後自動填入「出發地」

**參數：**
- `locateUser()` → 只在地圖上放定位點，地圖飛到該位置
- `locateUser(true)` → 同上，另外把座標填入路線出發地

**定位標記（藍色脈衝點）：** 樣式在 CSS 的 `.user-location-marker`、`.user-loc-core`、`.user-loc-ring`。

**使用者位置只存一個**，再次定位會移除舊標記再建新的。

**注意：** Geolocation API 在 `http://`（非 https）下，部分瀏覽器（Chrome）會拒絕定位。本機開發用 `localhost` 沒問題，但部署到正式主機需要 HTTPS 才能使用。

---

### 6. 褐根病通報圖層

**資料：** `brownrot_taipei.csv`（5,420 筆，欄位：`lon`、`lat`、`date`、`commonDamageName`）

**相關函式：** `loadBrownrot()`、`buildDmgPopup(row)`、`applyBrownrotFilter()`

**病害種類與顏色：**
- `褐根病` → 紫色 `#bf5af2`，**預設顯示**
- 其他病害 → 灰色 `#6e7681`，**預設隱藏**

**切換開關：** `#chk-brownrot`、`#chk-otherdmg`

---

### 7. 列管保護老樹圖層

**資料：** `protected_standalone.csv`（2,994 棵，未出現在行道樹主表的獨立老樹）

> 另外 805 棵老樹已透過空間配對整合進主表，在行道樹 popup 內會顯示「⭐ 列管保護」標記，不另外建立標記。

**相關函式：** `loadProtectedTrees()`、`buildProtectedPopup(row)`、`applyProtectedFilter()`

**標記顏色：** 金色 `#f0a500`，**預設顯示**

**切換開關：** `#chk-protected`

---

### 8. 樹穴圖層

**資料：** `tree_pits_wgs84.csv`（2,254 筆，已從 TWD97 轉換為 WGS84）

**相關函式：** `loadTreePits()`、`buildPitPopup(row)`、`applyPitFilter()`

**標記顏色：** 藍色 `#4a9eff`，**預設隱藏**（資料量不大但與核心議題相關性較低）

**切換開關：** `#chk-pit`

---

## 常見修改情境

### 新增一個地圖圖層

1. 在 `CONFIG` 加路徑：`newLayerPath: 'xxx.csv'`
2. 新增 `newLayerMarkers = []` 陣列
3. 寫 `loadNewLayer()` 函式（參考 `loadBrownrot`）
4. 在篩選面板 HTML 加 checkbox
5. 寫 `applyNewLayerFilter()` 並綁定事件
6. 在圖例加說明
7. 在啟動區最後呼叫 `loadNewLayer()`

### 修改 popup 顯示的欄位

找到對應的 `build*Popup()` 函式，修改裡面的 HTML 字串。行道樹主圖層是 `buildPopupHtml(row, level)`（約第 770 行）。

### 更換地圖底圖

找到以下這段，換成別的 tile URL：

```javascript
L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', { ... })
```

常用替代：
- `dark_all` → CartoDB 深色（目前使用）
- `dark_nolabels` → CartoDB 深色無標籤
- OpenStreetMap 標準：`https://tile.openstreetmap.org/{z}/{x}/{y}.png`

### 修改配色主題

CSS 最上方有 `:root` 區塊，統一定義所有顏色變數：

```css
:root {
  --cyan:    #00d4ff;   /* 主要強調色 */
  --high:    #ff4757;   /* 高風險紅 */
  --mid:     #ffa502;   /* 中風險橘 */
  --low:     #2ed573;   /* 低風險綠 */
  --protected: #ffd700; /* 保護老樹金 */
  --brownrot:  #bf5af2; /* 褐根病紫 */
}
```

---

## 部署注意事項

- 所有 CSV 必須和 index.html **在同一個目錄**
- 需要 HTTP 伺服器（不能直接用 `file://` 開啟）
- 定位功能需要 **HTTPS**（localhost 本機測試不受影響）
- CSV 檔案總大小約 18MB，首次載入需要數秒是正常的
