# 手機版響應式設計 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `index.html` 加入 `@media (max-width: 768px)` 響應式支援，讓手機掃 QR code 後正常顯示，包含底部抽屜（統計/篩選/圖例/路線四 Tab）、精簡 header、浮動 3D FAB。

**Architecture:** 單一檔案修改。CSS 加在現有 `</style>` 前；手機版 HTML（抽屜 + FAB）加在 `</body>` 前；手機版 JS 加在現有 `</script>` 前。所有既有 JS 邏輯不動，只在 `applyFilter()`、統計更新、路線初始化三處做最小修改。

**Tech Stack:** 純 HTML/CSS/JS（無框架）、Leaflet.js、Touch Events API

---

## 檔案修改

| 檔案 | 變更 |
|------|------|
| `index.html` | 唯一修改檔案。新增 media query CSS、底部抽屜 HTML、FAB HTML、手機版 JS |

---

### Task 1：Header 手機版 CSS + 隱藏桌機浮動元素

**Files:**
- Modify: `index.html` — 在 `</style>` 標籤**之前**插入以下 CSS

- [ ] **Step 1：在 `</style>` 前插入 media query 區塊**

找到 `index.html` 第 633 行附近的 `</style>`，在它**之前**加入：

```css
  /* ══════════════════════════════════════════
     手機版響應式（≤768px）
     ══════════════════════════════════════════ */
  @media (max-width: 768px) {
    /* ── Header 精簡 ── */
    #header {
      height: 48px;
      padding: 0 12px;
      gap: 8px;
    }
    #header h1 {
      font-size: 13px;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      max-width: 140px;
    }
    #header-sub { display: none; }
    #btn-route  { display: none; }
    #btn-3d     { display: none; }

    /* 天氣小卡精簡：只留圖示+溫度 */
    #weather-widget {
      min-width: unset;
      padding: 4px 10px;
      gap: 6px;
    }
    #weather-desc     { display: none; }
    #weather-location { display: none; }
    #weather-temp     { font-size: 13px; }
    #weather-icon     { font-size: 20px; }

    /* ── 隱藏桌機浮動面板 ── */
    #stats         { display: none; }
    #filter-panel  { display: none; }
    #legend        { display: none; }
    #route-sidebar { display: none; }

    /* ── Popup 手機寬度 ── */
    .leaflet-popup-content {
      min-width: 82vw !important;
      max-width: 92vw !important;
    }
    .leaflet-popup-close-button {
      width: 44px !important;
      height: 44px !important;
      display: flex !important;
      align-items: center;
      justify-content: center;
      font-size: 18px !important;
    }
  }
```

- [ ] **Step 2：確認桌機版不受影響**

在瀏覽器開啟 `index.html`，視窗寬度 > 768px，確認統計面板、篩選面板、路線按鈕全部仍然顯示。

---

### Task 2：底部抽屜 CSS

**Files:**
- Modify: `index.html` — 在 Task 1 插入的 media query 區塊內，接在已有內容後面**繼續加入**

- [ ] **Step 1：在 media query 區塊內加入抽屜樣式**

緊接在 Task 1 的 `}` **之前**（media query 結尾的 `}` 前面）插入：

```css
    /* ── 底部抽屜 ── */
    #mobile-sheet {
      position: fixed;
      bottom: 0; left: 0; right: 0;
      background: rgba(2,9,20,0.97);
      border-top: 1px solid var(--border);
      backdrop-filter: blur(14px);
      z-index: 1500;
      height: 56px;
      transition: height 0.25s ease;
      display: flex;
      flex-direction: column;
      overflow: hidden;
      box-shadow: 0 -4px 24px rgba(0,0,0,0.6);
    }

    .sheet-handle-area {
      flex-shrink: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 8px 16px 6px;
      cursor: pointer;
      user-select: none;
    }
    .sheet-handle {
      width: 36px; height: 4px;
      background: rgba(0,212,255,0.3);
      border-radius: 2px;
      margin-bottom: 6px;
    }
    .sheet-mini-stats {
      display: flex;
      gap: 16px;
      font-family: 'Share Tech Mono', monospace;
      font-size: 12px;
      align-items: center;
    }
    .sheet-mini-stats span { display: flex; align-items: center; gap: 4px; }
    .mob-cnt { font-size: 14px; }

    .sheet-tabs {
      display: none;
      flex-shrink: 0;
      border-bottom: 1px solid var(--border);
    }
    .sheet-tab {
      flex: 1;
      padding: 8px 4px;
      background: none;
      border: none;
      color: #5a7a9a;
      font-size: 11px;
      font-family: 'Microsoft JhengHei', sans-serif;
      cursor: pointer;
      border-bottom: 2px solid transparent;
      transition: all 0.15s;
    }
    .sheet-tab.active {
      color: var(--cyan);
      border-bottom-color: var(--cyan);
    }

    .sheet-content {
      display: none;
      flex: 1;
      overflow-y: auto;
      padding: 12px 16px;
    }
    .sheet-content::-webkit-scrollbar { width: 2px; }
    .sheet-content::-webkit-scrollbar-thumb { background: rgba(0,212,255,0.2); }

    .sheet-panel { display: none; }
    .sheet-panel.active { display: block; }

    /* 抽屜內的篩選器樣式 */
    .mob-filter-label {
      display: flex; align-items: center; gap: 8px;
      font-size: 13px; padding: 6px 0; cursor: pointer;
    }
    .mob-filter-label input[type=checkbox] {
      accent-color: var(--cyan); width: 16px; height: 16px; cursor: pointer;
    }
    .mob-filter-select {
      width: 100%; margin-top: 10px; padding: 8px 10px; border-radius: 3px;
      background: rgba(0,212,255,0.05); color: #c8d8f0;
      border: 1px solid var(--border); font-size: 13px;
      font-family: 'Microsoft JhengHei', sans-serif;
    }
    .mob-filter-select option { background: #020d1c; }
    .mob-divider { border: none; border-top: 1px solid rgba(255,255,255,0.08); margin: 8px 0; }

    /* 抽屜內路線規劃 */
    .mob-rs-input {
      width: 100%; padding: 10px 12px; margin-bottom: 8px;
      background: rgba(0,212,255,0.04); border: 1px solid rgba(0,212,255,0.15);
      border-radius: 3px; color: #c8d8f0; font-size: 14px; outline: none;
      font-family: 'Microsoft JhengHei', sans-serif;
    }
    .mob-rs-input:focus { border-color: var(--cyan); box-shadow: 0 0 8px var(--cyan-dim); }
    .mob-rs-input::placeholder { color: #2a3a50; }
    .mob-rs-status {
      font-size: 11px; color: var(--cyan); min-height: 14px; margin-bottom: 8px;
      font-family: 'Share Tech Mono', monospace;
    }
    .mob-rs-err { color: var(--high); }
    .mob-rs-btn {
      width: 100%; padding: 11px; border-radius: 3px; border: none;
      font-size: 13px; font-weight: 700; cursor: pointer; margin-bottom: 8px;
      font-family: 'Share Tech Mono', monospace; letter-spacing: 1px;
    }
    .mob-rs-btn-primary   { background: linear-gradient(135deg, #0a4f9e, #1060bf); color: var(--cyan); border: 1px solid rgba(0,212,255,0.3); }
    .mob-rs-btn-secondary { background: rgba(0,212,255,0.05); color: #5a7a9a; border: 1px solid rgba(0,212,255,0.1); }
    .mob-rs-mode { display: flex; gap: 6px; margin: 8px 0 12px; }
    .mob-rs-mode label {
      flex: 1; display: flex; align-items: center; justify-content: center;
      gap: 4px; padding: 8px 4px; border: 1px solid rgba(0,212,255,0.15);
      border-radius: 3px; cursor: pointer; font-size: 12px; transition: all 0.2s;
    }
    .mob-rs-mode input[type=radio] { display: none; }
    .mob-rs-mode label:has(input:checked) {
      background: var(--cyan-dim); border-color: var(--cyan); color: var(--cyan);
    }
    #mob-route-result { margin-top: 8px; }

    /* 3D FAB */
    #fab-3d {
      position: fixed;
      right: 16px;
      bottom: 72px;
      width: 44px; height: 44px;
      border-radius: 50%;
      background: rgba(74,158,255,0.1);
      border: 1px solid rgba(74,158,255,0.35);
      color: var(--blue);
      font-family: 'Share Tech Mono', monospace;
      font-size: 11px; font-weight: 600;
      cursor: pointer;
      z-index: 1400;
      display: none;            /* 預設隱藏，手機版 JS 控制顯示 */
      align-items: center;
      justify-content: center;
      box-shadow: 0 2px 12px rgba(0,0,0,0.5);
      transition: all 0.2s;
    }
    #fab-3d.active {
      background: rgba(74,158,255,0.2);
      border-color: var(--blue);
      box-shadow: 0 0 12px rgba(74,158,255,0.35);
    }
```

- [ ] **Step 2：存檔，瀏覽器重整，確認桌機版版面正常**

---

### Task 3：底部抽屜 + FAB HTML

**Files:**
- Modify: `index.html` — 在 `</body>` 標籤之前插入

- [ ] **Step 1：在 `</body>` 前插入抽屜與 FAB HTML**

找到 `</body>`，在它**之前**插入：

```html
  <!-- ══ 手機版底部抽屜 ══ -->
  <div id="mobile-sheet">
    <!-- 拖拉把手 + 收合時迷你統計 -->
    <div class="sheet-handle-area" id="sheet-handle-area">
      <div class="sheet-handle"></div>
      <div class="sheet-mini-stats" id="sheet-mini-stats">
        <span><span class="dot dot-high"></span><span class="mob-cnt c-high" id="mob-cnt-high">—</span></span>
        <span><span class="dot dot-mid"></span><span class="mob-cnt c-mid" id="mob-cnt-mid">—</span></span>
        <span><span class="dot dot-low"></span><span class="mob-cnt c-low" id="mob-cnt-low">—</span></span>
        <span><span class="dot" style="background:#6e7681"></span><span class="mob-cnt" style="color:#6e7681" id="mob-cnt-none">—</span></span>
      </div>
    </div>

    <!-- Tab 列（半展開/全展開時顯示） -->
    <div class="sheet-tabs" id="sheet-tabs">
      <button class="sheet-tab active" data-tab="stats">📊 統計</button>
      <button class="sheet-tab" data-tab="filter">🔍 篩選</button>
      <button class="sheet-tab" data-tab="legend">🗺 圖例</button>
      <button class="sheet-tab" data-tab="route">🧭 路線</button>
    </div>

    <!-- Tab 內容 -->
    <div class="sheet-content" id="sheet-content">

      <!-- 統計 tab -->
      <div class="sheet-panel active" id="mob-tab-stats">
        <div class="panel-title" style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);margin-bottom:10px">風險統計</div>
        <div class="stat-row">
          <span><span class="dot dot-high"></span><span class="c-high">高/重風險</span></span>
          <span class="cnt c-high" id="mob-stat-high">—</span>
        </div>
        <div class="stat-row">
          <span><span class="dot dot-mid"></span><span class="c-mid">中風險</span></span>
          <span class="cnt c-mid" id="mob-stat-mid">—</span>
        </div>
        <div class="stat-row">
          <span><span class="dot dot-low"></span><span class="c-low">低/D級</span></span>
          <span class="cnt c-low" id="mob-stat-low">—</span>
        </div>
        <div class="stat-row" style="opacity:0.6">
          <span><span class="dot" style="background:#6e7681"></span><span style="color:#6e7681">未健檢</span></span>
          <span class="cnt" style="color:#6e7681" id="mob-stat-none">—</span>
        </div>
        <hr class="divider">
        <div class="stat-row">
          <span class="c-total">總計</span>
          <span class="cnt c-total" id="mob-stat-total">—</span>
        </div>
      </div>

      <!-- 篩選 tab -->
      <div class="sheet-panel" id="mob-tab-filter">
        <div class="panel-title" style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);margin-bottom:10px">顯示篩選</div>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-high" checked> <span class="c-high">高/重風險</span></label>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-mid"  checked> <span class="c-mid">中風險</span></label>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-low"  checked> <span class="c-low">低/D級</span></label>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-none"> <span style="color:#6e7681">未健檢樹木</span></label>
        <hr class="mob-divider">
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-brownrot" checked> <span style="color:#a371f7">褐根病通報</span></label>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-otherdmg"> <span style="color:#6e7681">其他病害通報</span></label>
        <hr class="mob-divider">
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-protected" checked> <span style="color:#f0a500">列管保護老樹</span></label>
        <label class="mob-filter-label"><input type="checkbox" id="mob-chk-pit"> <span style="color:#4a9eff">樹穴位置</span></label>
        <select id="mob-sel-dist" class="mob-filter-select">
          <option value="">所有行政區</option>
        </select>
      </div>

      <!-- 圖例 tab -->
      <div class="sheet-panel" id="mob-tab-legend">
        <div class="panel-title" style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);margin-bottom:10px">風險等級（官方健檢）</div>
        <div class="legend-item"><div class="legend-dot" style="background:#f85149"></div>高/重風險</div>
        <div class="legend-item"><div class="legend-dot" style="background:#e3b341"></div>中風險</div>
        <div class="legend-item"><div class="legend-dot" style="background:#3fb950"></div>低/D級風險</div>
        <hr style="border:none;border-top:1px solid rgba(255,255,255,0.08);margin:8px 0">
        <div class="legend-item"><div class="legend-dot" style="background:#a371f7;border-radius:3px"></div>褐根病通報</div>
        <div class="legend-item"><div class="legend-dot" style="background:#6e7681;border-radius:3px"></div>其他病害通報</div>
        <div class="legend-item"><div class="legend-dot" style="background:#f0a500;border-radius:2px"></div>列管保護老樹</div>
        <div class="legend-item"><div class="legend-dot" style="background:#4a9eff;border-radius:2px"></div>樹穴位置</div>
        <div style="margin-top:10px;font-size:11px;color:#8b949e;line-height:1.6">
          資料來源：台北市政府工務局<br>公園路燈工程管理處<br>林業試驗所林木疫情監測系統
        </div>
      </div>

      <!-- 路線 tab -->
      <div class="sheet-panel" id="mob-tab-route">
        <div class="panel-title" style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);margin-bottom:10px">◈ ROUTE PLANNER</div>
        <div style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);opacity:0.7;margin-bottom:5px">出發地</div>
        <input class="mob-rs-input" id="mob-inp-start" type="text" placeholder="輸入地址或地名" autocomplete="off">
        <div class="mob-rs-status" id="mob-start-status"></div>
        <div style="font-family:'Share Tech Mono',monospace;font-size:10px;letter-spacing:2px;color:var(--cyan);opacity:0.7;margin-bottom:5px">目的地</div>
        <input class="mob-rs-input" id="mob-inp-end" type="text" placeholder="輸入地址或地名" autocomplete="off">
        <div class="mob-rs-status" id="mob-end-status"></div>
        <div class="mob-rs-mode">
          <label><input type="radio" name="mob-tmode" value="foot" checked>🚶 步行</label>
          <label><input type="radio" name="mob-tmode" value="driving">🚗 開車</label>
          <label><input type="radio" name="mob-tmode" value="bike">🚲 騎車</label>
        </div>
        <button class="mob-rs-btn mob-rs-btn-primary" onclick="handleMobRouteSearch()">查詢路線</button>
        <button class="mob-rs-btn mob-rs-btn-secondary" id="mob-btn-clear-route" onclick="clearMobRoute()" style="display:none">清除路線</button>
        <div id="mob-route-result"></div>
      </div>

    </div>
  </div>

  <!-- 3D 模式 FAB（手機版） -->
  <button id="fab-3d" onclick="toggle3D()">3D</button>
```

- [ ] **Step 2：存檔，手機模擬（Chrome DevTools → 375px），確認抽屜出現在底部**

---

### Task 4：底部抽屜 JS（Touch handler + Tab 切換 + FAB）

**Files:**
- Modify: `index.html` — 在 `</script>` 之前插入

- [ ] **Step 1：在最後的 `</script>` 之前插入手機版 JS**

```javascript
  // ════════════════════════════════════════
  // 手機版底部抽屜
  // ════════════════════════════════════════
  (function initMobileSheet() {
    const sheet      = document.getElementById('mobile-sheet');
    const handleArea = document.getElementById('sheet-handle-area');
    const miniStats  = document.getElementById('sheet-mini-stats');
    const tabBar     = document.getElementById('sheet-tabs');
    const content    = document.getElementById('sheet-content');
    if (!sheet) return;

    const COLL_H = 56;
    let   currentState = 'collapsed';
    let   startY = 0, startH = 0, dragging = false;

    function halfH() { return Math.round(window.innerHeight * 0.45); }
    function fullH() { return Math.round(window.innerHeight * 0.85); }

    function setSheetHeight(h, animate) {
      sheet.style.transition = animate ? 'height 0.25s ease' : 'none';
      sheet.style.height = h + 'px';
    }

    function snapToState(state) {
      currentState = state;
      const isColl = state === 'collapsed';
      setSheetHeight(isColl ? COLL_H : (state === 'full' ? fullH() : halfH()), true);
      miniStats.style.display  = isColl ? 'flex' : 'none';
      tabBar.style.display     = isColl ? 'none' : 'flex';
      content.style.display    = isColl ? 'none' : 'block';
      sheet.dataset.state      = state;
    }

    function switchTab(tabName) {
      document.querySelectorAll('.sheet-tab').forEach(t =>
        t.classList.toggle('active', t.dataset.tab === tabName));
      document.querySelectorAll('.sheet-panel').forEach(p =>
        p.classList.toggle('active', p.id === 'mob-tab-' + tabName));
    }

    // 拖拉
    handleArea.addEventListener('touchstart', e => {
      startY   = e.touches[0].clientY;
      startH   = sheet.offsetHeight;
      dragging = true;
      sheet.style.transition = 'none';
    }, { passive: true });

    handleArea.addEventListener('touchmove', e => {
      if (!dragging) return;
      const dy   = startY - e.touches[0].clientY;
      const newH = Math.max(COLL_H, Math.min(fullH(), startH + dy));
      setSheetHeight(newH, false);
    }, { passive: true });

    handleArea.addEventListener('touchend', e => {
      if (!dragging) return;
      dragging = false;
      const dy = startY - e.changedTouches[0].clientY;
      if (Math.abs(dy) < 10) {
        snapToState(currentState === 'collapsed' ? 'half' : 'collapsed');
        return;
      }
      const h = sheet.offsetHeight;
      if      (h > halfH() + 80) snapToState('full');
      else if (h > COLL_H  + 60) snapToState('half');
      else                        snapToState('collapsed');
    }, { passive: true });

    // Tab 點擊
    document.querySelectorAll('.sheet-tab').forEach(tab => {
      tab.addEventListener('click', () => {
        switchTab(tab.dataset.tab);
        if (currentState === 'collapsed') snapToState('half');
      });
    });

    // 點 mini-stats → 展開統計 tab
    miniStats.addEventListener('click', () => {
      switchTab('stats');
      snapToState('half');
    });

    // FAB 3D：手機版顯示
    const fab = document.getElementById('fab-3d');
    if (fab && window.innerWidth <= 768) fab.style.display = 'flex';

    // 初始
    snapToState('collapsed');
  })();
```

- [ ] **Step 2：在手機模擬器測試**
  - 往上滑：展開到半展開
  - 繼續往上滑：展開到全展開
  - 往下滑：收合
  - 點「📊 統計」tab：顯示統計內容

---

### Task 5：篩選器 JS 同步 + 統計更新

**Files:**
- Modify: `index.html` — 修改 `applyFilter()` 函式 + 統計更新區塊 + 行政區下拉

- [ ] **Step 1：修改 `applyFilter()` 函式**

找到 `function applyFilter()` 並**整個取代**：

```javascript
  function applyFilter() {
    const mob     = window.innerWidth <= 768;
    const pfx     = mob ? 'mob-' : '';
    const showHigh = document.getElementById(pfx + 'chk-high').checked;
    const showMid  = document.getElementById(pfx + 'chk-mid').checked;
    const showLow  = document.getElementById(pfx + 'chk-low').checked;
    const showNone = document.getElementById(pfx + 'chk-none').checked;
    const showBr   = document.getElementById(pfx + 'chk-brownrot').checked;
    const showProt = document.getElementById(pfx + 'chk-protected').checked;
    const selDist  = document.getElementById(pfx + 'sel-dist').value;

    allMarkers.forEach(({ marker, level, dist, hasBrownrot, isProtected }) => {
      const levelOk = (level === 'high' && showHigh) ||
                      (level === 'mid'  && showMid)  ||
                      (level === 'low'  && showLow)  ||
                      (level === 'none' && showNone);
      const brOk    = showBr   && hasBrownrot;
      const protOk  = showProt && isProtected;
      const distOk  = !selDist || dist === selDist;

      if ((levelOk || brOk || protOk) && distOk) marker.addTo(map);
      else                                        marker.remove();
    });

    if (is3DMode) do3DUpdate();
  }
```

- [ ] **Step 2：在 `applyFilter()` 的 event listeners 後加入手機版 listeners**

找到：
```javascript
  ['chk-high','chk-mid','chk-low','chk-none'].forEach(id => {
    document.getElementById(id).addEventListener('change', applyFilter);
  });
```

在這段**之後**加入：
```javascript
  // 手機版篩選器 listeners
  ['mob-chk-high','mob-chk-mid','mob-chk-low','mob-chk-none',
   'mob-chk-brownrot','mob-chk-protected'].forEach(id => {
    document.getElementById(id).addEventListener('change', applyFilter);
  });
  document.getElementById('mob-sel-dist').addEventListener('change', function () {
    applyFilter();
    updateWeatherForDistrict(this.value);
  });
```

- [ ] **Step 3：修改統計更新區塊，同步更新手機版元素**

找到這段（約第 1133–1139 行）：
```javascript
        // 更新統計
        const total = counts.high + counts.mid + counts.low;
        document.getElementById('cnt-high').textContent  = counts.high.toLocaleString();
        document.getElementById('cnt-mid').textContent   = counts.mid.toLocaleString();
        document.getElementById('cnt-low').textContent   = counts.low.toLocaleString();
        document.getElementById('cnt-none').textContent  = counts.none.toLocaleString();
        document.getElementById('cnt-total').textContent = total.toLocaleString();
```

**整個取代**成：
```javascript
        // 更新統計（桌機 + 手機版）
        const total = counts.high + counts.mid + counts.low;
        const setAll = (suffix, high, mid, low, none, tot) => {
          document.getElementById(suffix + 'high').textContent  = high.toLocaleString();
          document.getElementById(suffix + 'mid').textContent   = mid.toLocaleString();
          document.getElementById(suffix + 'low').textContent   = low.toLocaleString();
          document.getElementById(suffix + 'none').textContent  = none.toLocaleString();
          document.getElementById(suffix + 'total').textContent = tot.toLocaleString();
        };
        setAll('cnt-',      counts.high, counts.mid, counts.low, counts.none, total);
        setAll('mob-cnt-',  counts.high, counts.mid, counts.low, counts.none, total);
        setAll('mob-stat-', counts.high, counts.mid, counts.low, counts.none, total);
```

- [ ] **Step 4：修改行政區下拉，同步填入手機版 `mob-sel-dist`**

找到（約第 1141–1147 行）：
```javascript
        // 行政區下拉
        const sel = document.getElementById('sel-dist');
        [...distSet].sort().forEach(d => {
          const opt = document.createElement('option');
          opt.value = d; opt.textContent = d;
          sel.appendChild(opt);
        });
```

**整個取代**成：
```javascript
        // 行政區下拉（桌機 + 手機版）
        const distArr = [...distSet].sort();
        ['sel-dist', 'mob-sel-dist'].forEach(selId => {
          const sel = document.getElementById(selId);
          distArr.forEach(d => {
            const opt = document.createElement('option');
            opt.value = d; opt.textContent = d;
            sel.appendChild(opt);
          });
        });
```

- [ ] **Step 5：測試篩選器同步**

手機模擬器中：
1. 打開篩選 tab，取消「高/重風險」勾選 → 地圖上紅點消失
2. 切換行政區 → 地圖篩選正確

---

### Task 6：路線規劃手機版 JS

**Files:**
- Modify: `index.html` — 修改 `initRoutePanel()` 函式，新增 `handleMobRouteSearch()` 和 `clearMobRoute()`

- [ ] **Step 1：修改 `initRoutePanel()`，加入手機版 input 初始化**

找到：
```javascript
  function initRoutePanel() {
    setupGeoInput('inp-start', 'start-status', latlng => { startLatLng = latlng; });
    setupGeoInput('inp-end',   'end-status',   latlng => { endLatLng   = latlng; });
  }
```

**整個取代**成：
```javascript
  function initRoutePanel() {
    setupGeoInput('inp-start',     'start-status',     latlng => { startLatLng = latlng; });
    setupGeoInput('inp-end',       'end-status',       latlng => { endLatLng   = latlng; });
    setupGeoInput('mob-inp-start', 'mob-start-status', latlng => { startLatLng = latlng; });
    setupGeoInput('mob-inp-end',   'mob-end-status',   latlng => { endLatLng   = latlng; });
  }
```

- [ ] **Step 2：新增 `handleMobRouteSearch()` 和 `clearMobRoute()`**

在 `initRoutePanel()` **之後**插入：

```javascript
  async function handleMobRouteSearch() {
    const startSt = document.getElementById('mob-start-status');
    const endSt   = document.getElementById('mob-end-status');
    if (!startLatLng) {
      startSt.className = 'mob-rs-status mob-rs-err';
      startSt.textContent = '請選擇出發地';
      return;
    }
    if (!endLatLng) {
      endSt.className = 'mob-rs-status mob-rs-err';
      endSt.textContent = '請選擇目的地';
      return;
    }
    // 讀取手機版交通方式
    const mode = document.querySelector('input[name="mob-tmode"]:checked')?.value || 'foot';
    // 複用桌機版路線查詢邏輯：暫時同步交通方式選擇到桌機版再呼叫
    const deskRadio = document.querySelector(`input[name="tmode"][value="${mode}"]`);
    if (deskRadio) deskRadio.checked = true;
    // 顯示查詢中
    document.getElementById('mob-route-result').innerHTML =
      '<div style="color:var(--cyan);font-family:\'Share Tech Mono\',monospace;font-size:12px;padding:8px 0">查詢路線中…</div>';
    // 呼叫桌機版 handleRouteSearch，結果寫入兩個 result div
    await handleRouteSearch(true);
  }

  function clearMobRoute() {
    clearRoute();
    document.getElementById('mob-route-result').innerHTML = '';
    document.getElementById('mob-inp-start').value = '';
    document.getElementById('mob-inp-end').value   = '';
    document.getElementById('mob-start-status').textContent = '';
    document.getElementById('mob-end-status').textContent   = '';
    document.getElementById('mob-btn-clear-route').style.display = 'none';
  }
```

- [ ] **Step 3：修改 `handleRouteSearch()` 接受 `fromMobile` 參數，同步更新手機版 result**

找到 `async function handleRouteSearch()` 的宣告行，改為：
```javascript
  async function handleRouteSearch(fromMobile = false) {
```

然後找到 `handleRouteSearch` 內部寫入 `route-result` 的行（`document.getElementById('route-result').innerHTML`），在**每一處**之後加一行同步到手機版：
```javascript
    document.getElementById('mob-route-result').innerHTML =
      document.getElementById('route-result').innerHTML;
    if (fromMobile) document.getElementById('mob-btn-clear-route').style.display = 'block';
```

- [ ] **Step 4：測試路線規劃**

手機模擬器中：
1. 路線 tab → 輸入「台北車站」和「大安森林公園」
2. 點「查詢路線」→ 結果顯示在 tab 內，地圖上出現路線

---

### Task 7：Popup 自動 pan + 最終測試 + Commit

**Files:**
- Modify: `index.html` — 在樹木 marker 點擊後加入 pan 邏輯

- [ ] **Step 1：找到 `buildPopup` 後 marker.bindPopup 的地方，加入 pan**

找到類似這樣的程式碼（marker.bindPopup 後）：

```javascript
          marker.bindPopup(buildPopup(row, level));
```

在這行**之後**加入：
```javascript
          marker.on('popupopen', () => {
            if (window.innerWidth <= 768) map.panBy([0, -100]);
          });
```

- [ ] **Step 2：全功能手機模擬測試**

在 Chrome DevTools，選 iPhone 12 Pro（390×844）：
- [ ] Header 只顯示「台北市路樹倒塌風險查詢平台」縮短版 + 天氣圖示+溫度
- [ ] 地圖全螢幕，底部 56px 抽屜把手可見
- [ ] 往上滑 → 展開到半展開，出現四個 Tab
- [ ] 統計 tab：高/中/低/未健檢數字正確
- [ ] 篩選 tab：取消高風險 → 紅點消失
- [ ] 行政區下拉：切換大安區 → 地圖篩選 + 天氣更新
- [ ] 圖例 tab：圓點說明顯示
- [ ] 路線 tab：輸入起訖地 → 查詢 → 路線出現在地圖上
- [ ] 點地圖上的樹 → Popup 出現且不被抽屜擋住
- [ ] 右下角 FAB「3D」按鈕可見，點擊可切換 3D 模式

- [ ] **Step 3：桌機版迴歸測試（視窗 > 768px）**
- [ ] 統計面板、篩選面板、路線按鈕全部正常顯示
- [ ] 路線側欄正常開關
- [ ] 地圖點擊 popup 正常

- [ ] **Step 4：Commit**

```bash
git add index.html
git commit -m "feat: 手機版響應式設計（底部抽屜 + 四 tab + FAB）"
```

- [ ] **Step 5：Push 到 GitHub，GitHub Pages 生效後掃 QR code 測試**

```bash
git push origin main
```

約 1-2 分鐘後，用手機掃 QR code 確認網站正常顯示。
