# Private Train Assistant - SDD AI Sprint Spec

## Core Intent

一個以 AI 對話為核心的個人重訓助理 Web App。用戶透過固定 5 步驟對話，在 5 分鐘內獲得符合自身需求（時間、部位、器材）的當日訓練菜單，並在訓練後完成打卡記錄。解決新手訓練者無工具可系統化安排與追蹤訓練習慣的核心痛點。

## Target User

使用手機瀏覽器的個人重訓新手，目前以 LINE 記事本或純記憶管理訓練內容，需要一個能彈性調整菜單、方便快速記錄的私人工具。

## MVP Features (Max 5)

- [ ] Feature 1 (Day 1) - AI 對話式菜單安排：固定 5 步驟問答流程，產出當日訓練菜單（含替代動作功能）
- [ ] Feature 2 (Day 1) - 聊天介面 + 側邊欄導覽：ChatGPT 風格聊天畫面，左側邊欄含訓練分類入口
- [ ] Feature 3 (Day 2) - 訓練記錄儲存：將 AI 產出的菜單存入 Firebase，含動作名稱、組數、次數、重量
- [ ] Feature 4 (Day 2) - 類別記錄頁面：依分類（上肢/下肢/核心/全身/伸展）瀏覽歷史記錄卡片列表
- [ ] Feature 5 (Day 3) - 訓練打卡完成流程：標記菜單為「已完成」並給予明顯成功回饋動畫

## Non-Features (What we're NOT building)

- 用戶帳號系統 / 登入驗證（MVP 使用匿名或 localStorage，v2 再加）
- 訓練數據圖表與進度分析（痛點是「沒有記錄」，分析是 v2 的事）
- 自訂動作資料庫管理介面（AI 內建動作庫即可）
- 社交分享或多人功能
- Apple Watch / 穿戴裝置整合
- 推播通知與提醒系統
- 離線完整功能（基本 localStorage fallback 即可）
- 多語系支援

## Success Criteria

- 可在 5 分鐘 Demo 中展示：完整對話流程產出菜單 → 儲存記錄 → 查看歷史
- 從開啟 App 到獲得今日菜單不超過 5 個對話來回
- 訓練打卡後有明顯視覺回饋（動畫 + 狀態更新）
- 可在手機瀏覽器正常操作（RWD，主要支援 375px 以上寬度）
- Day 3 可部署至可公開存取的 URL

## Technical Constraints

- 單人開發，48 小時執行
- 使用既有技術堆疊，不引入額外後端服務
- AI 對話使用現有 LLM API（OpenAI / Gemini），prompt engineering 控制流程
- Firebase Firestore 儲存訓練記錄，localStorage 作為離線 fallback
- 不建置自訂 API server（直接從前端呼叫 LLM API）

## Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Platform | Web (RWD, mobile-first) | 手機為主，無需安裝，快速部署 |
| Framework | Vue 3 + Vite + TypeScript | 既定技術堆疊，開發速度快 |
| UI Kit | Ant Design Vue | 既定技術堆疊，元件豐富 |
| Styling | Tailwind CSS + SCSS | 既定技術堆疊，utility-first 加速樣式開發 |
| AI | OpenAI Chat API (GPT-4o-mini) | 對話流程控制穩定，成本低 |
| Database | Firebase Firestore | 既定技術堆疊，無後端即可使用 |
| Local Cache | localStorage | 離線 fallback，減少 Firestore 讀取 |
| Testing | Vitest (unit) | 既定技術堆疊 |
| Deployment | Vercel | 免費、一鍵部署、支援環境變數 |

## Visual Design Spec

### Design Language

- **Platform style**: Custom Web，Mobile-first RWD
- **Color mode**: Dark mode 為主（固定深色，不跟隨系統）
- **Primary color**: `#1E3A5F`（深藍）
- **Accent color**: `#C9A84C`（金色，用於 CTA、highlight、成功狀態）
- **Background**: `#0D1117`（近黑深色底）
- **Surface**: `#161B22`（卡片、側邊欄底色）
- **Border**: `#30363D`（分隔線、卡片邊框）
- **Text primary**: `#E6EDF3`（主要文字）
- **Text secondary**: `#8B949E`（次要文字、時間戳記）
- **Font**: Inter（英文）+ Noto Sans TC（中文），fallback system-ui
- **Border radius**: 12px（卡片）、8px（按鈕、輸入框）、20px（訊息泡泡）
- **Spacing base unit**: 8px grid

### Typography Scale

| Role | Size | Weight |
|------|------|--------|
| Page Title | 20px | Bold (700) |
| Section Heading | 16px | Medium (500) |
| Body | 14px | Regular (400) |
| Caption / Timestamp | 12px | Regular (400) |
| Button Label | 14px | Medium (500) |

---

## Screen Specs

### Screen 1: Chat（聊天主畫面）

- **Purpose**: AI 對話式菜單安排的核心操作畫面
- **Layout**: 三欄結構 — 左側邊欄（手機收合）+ 中央聊天區（全寬）+ 底部輸入列
- **Key components**:
  - **Header bar**: 高度 56px，左側漢堡 icon（24px）觸發側邊欄，中央顯示「Private Train Assistant」文字，右側無按鈕
  - **Message list**: 滾動區域，佔滿畫面高度扣除 header(56px) + input bar(72px)；訊息由上而下排列，最新訊息置底
  - **AI message bubble**: 左對齊，背景 `#161B22`，border `1px solid #30363D`，border-radius 20px，padding 12px 16px，max-width 80%
  - **User message bubble**: 右對齊，背景 `#1E3A5F`，border-radius 20px，padding 12px 16px，max-width 80%
  - **Typing indicator bubble**: 左對齊，與 AI bubble 同樣樣式，內含三個點動畫（每個點 400ms delay 交錯上下 bounce）
  - **Quick reply chips**: 出現在 AI 訊息下方，橫向排列可左右捲動；背景 `#161B22`，border `1px solid #C9A84C`，border-radius 20px，padding 8px 16px，字色 `#C9A84C`，點擊後自動填入輸入框並送出
  - **Input bar**: 固定底部，高度 72px，padding 12px 16px；左側為 textarea（自動高度，最多 3 行，背景 `#161B22`，border-radius 8px），右側為送出 icon button（24px，金色 `#C9A84C`，disabled 時 opacity 0.4）
- **Interactions**:
  - 送出訊息 → 立即顯示 User bubble → 顯示 Typing indicator → AI 回應後替換 Typing indicator（動畫：bubble fade-in 150ms）
  - 點擊 Quick reply chip → 自動送出，chips 消失（動畫：chip tap scale 0.95，100ms）
  - 新訊息出現 → 自動 scroll to bottom（smooth scroll 200ms）
  - 漢堡 icon 點擊 → 側邊欄從左滑入（動畫：translateX -100% → 0，250ms ease-out）
- **Empty state**: 首次進入顯示 AI 歡迎訊息：「嗨！我是你的訓練助理。請問今天要做重訓還是伸展？」並附帶「重訓」「伸展」兩個 Quick reply chips
- **Loading state**: 等待 AI 回應時顯示 Typing indicator bubble
- **Error state**: AI API 失敗時在聊天區顯示錯誤訊息 bubble（背景 `#3D1A1A`，文字：「連線發生問題，請點擊重試」），附「重試」按鈕（金色文字按鈕）

---

### Screen 2: Sidebar（側邊欄）

- **Purpose**: 訓練分類導覽，進入各類別歷史記錄
- **Layout**: 手機上覆蓋畫面左側，寬度 280px；桌機（768px+）固定展開，不覆蓋主內容
- **Key components**:
  - **Overlay backdrop**: 側邊欄開啟時，右側區域覆蓋半透明黑色遮罩（`rgba(0,0,0,0.6)`），點擊關閉側邊欄
  - **Sidebar header**: 高度 56px，顯示「訓練記錄」文字（16px Bold），右側 X 關閉 icon（手機限定）
  - **New chat button**: 側邊欄頂部，全寬按鈕，label「+ 新對話」，背景 `#1E3A5F`，hover/active 背景 `#264D7A`，高度 44px，border-radius 8px
  - **Category list**: 垂直排列，每項高度 48px，左側分類 icon（20px）+ 分類名稱（14px）；分類項目：上肢、下肢、核心、全身、伸展
  - **Active state**: 當前選中分類背景 `#1E3A5F`，文字 `#E6EDF3`；非選中文字 `#8B949E`
  - **Divider**: 分類列表上下各有 `1px solid #30363D` 分隔線
- **Interactions**:
  - 點擊分類 → 進入該類別記錄頁面，側邊欄關閉（手機）（動畫：側邊欄 slide-out 250ms）
  - 點擊「+ 新對話」→ 回到 Chat 畫面，清空對話記錄，側邊欄關閉
  - 點擊遮罩或 X → 側邊欄關閉
- **Empty state**: 不適用（分類為固定清單）
- **Loading state**: 不適用
- **Error state**: 不適用

---

### Screen 3: Category Records（類別記錄頁面）

- **Purpose**: 瀏覽特定訓練分類的所有歷史記錄
- **Layout**: 全頁垂直捲動卡片列表；Header + 卡片列表（padding 16px）
- **Key components**:
  - **Page header**: 高度 56px，左側返回箭頭 icon，中央顯示分類名稱（如「上肢訓練」），背景與 `#0D1117` 相同
  - **Record card**: 背景 `#161B22`，border `1px solid #30363D`，border-radius 12px，padding 16px，margin-bottom 12px
    - **Card header row**: 左側菜單名稱（預設為日期，格式 `YYYY/MM/DD`，16px Medium），右側總時數 badge（背景 `#1E3A5F`，文字 `#C9A84C`，12px，border-radius 12px，padding 4px 10px）
    - **Completion badge**: 已完成顯示金色「已完成」tag（`#C9A84C` 文字，border `1px solid #C9A84C`，border-radius 12px，12px，padding 2px 8px）；未完成不顯示
    - **Action list**: 卡片下方以 row 排列，每行顯示：動作名稱（14px）+ 組數x次數（14px `#8B949E`）+ 重量或「徒手」（12px `#C9A84C`）
    - **Divider**: card header 與 action list 之間 `1px solid #30363D`，margin 8px 0
  - **Floating action button**: 右下角固定，直徑 56px，背景 `#C9A84C`，icon 為「+」（24px 白色），點擊跳回 Chat 畫面開始新對話
- **Interactions**:
  - 點擊卡片 → 展開詳細資訊（預留，MVP 不實作，卡片不可點擊）
  - 點擊 FAB → 跳回 Chat 畫面，側邊欄關閉
  - 下拉到頂 → 不需 pull-to-refresh（MVP 省略）
- **Empty state**: 無任何記錄時，畫面中央顯示 icon（啞鈴）+ 文字「還沒有訓練記錄」（`#8B949E`，16px）+ 文字「點擊右下角開始安排今天的菜單」（`#8B949E`，14px）
- **Loading state**: 卡片列表讀取中顯示 3 張 skeleton card（灰色佔位，動畫：shimmer 左右掃過，1200ms loop）
- **Error state**: 讀取失敗時顯示「載入失敗」文字 + 「重新整理」文字按鈕（金色）

---

## AI Conversation Flow

### 固定 5 步驟對話流程

```
Step 1 [AI]:  「今天要做重訓還是伸展？」
              Quick replies: [重訓] [伸展]

Step 2 [AI]:  「預計運動多久？」
              Quick replies: [30 分鐘] [45 分鐘] [60 分鐘] [90 分鐘]

Step 3 [AI]:  「需要器材（啞鈴/槓鈴/機器）還是純徒手？」
              Quick replies: [有器材] [純徒手]

              → AI 產出第一版菜單（結構化格式）
              → 同時詢問：「需要替換其中任何動作嗎？」
              Quick replies: [不用，就這樣] [要替換]

Step 4 [若選替換]:
       [AI]:  「哪個動作需要替換？請輸入動作名稱」
       [User]: 輸入動作名稱
       [AI]:  產出替代動作建議，更新菜單

Step 5 [AI]:  「菜單已準備好！要儲存今天的訓練計畫嗎？」
              Quick replies: [儲存] [不用了]
              → 儲存後顯示成功回饋，提示可在側邊欄查看記錄
```

### AI Prompt 設計原則

- System prompt 限制 AI 角色：只能回覆訓練相關問題，依固定步驟引導
- 菜單輸出格式為 JSON（前端 parse 後渲染為結構化卡片），包含：
  ```json
  {
    "category": "上肢",
    "duration": 45,
    "equipment": "有器材",
    "exercises": [
      { "name": "槓鈴臥推", "sets": 4, "reps": 10, "weight": "自訂" },
      { "name": "啞鈴飛鳥", "sets": 3, "reps": 12, "weight": "自訂" }
    ]
  }
  ```
- Temperature: 0.3（保持輸出穩定，避免格式跑掉）

---

## Day-by-Day Execution Plan

### Day 1 (Hours 1-16)

- **Hours 1-2**: 專案初始化，設定 Firebase、環境變數、Tailwind dark mode 設定、路由結構（Chat / Records）
- **Hours 3-5**: 視覺基礎建設：全域 CSS token（色彩、字體、spacing），Sidebar 元件（含手機 overlay 邏輯）
- **Hours 6-10**: Chat 畫面實作：訊息泡泡元件、Typing indicator、Quick reply chips、底部輸入列
- **Hours 11-14**: AI 對話流程串接：OpenAI API 呼叫、System prompt 設計、固定 5 步驟流程狀態機
- **Hours 15-16**: 整合測試：完整跑一次對話流程，確認 AI 輸出格式正確

### Day 2 (Hours 17-32)

- **Hours 17-20**: Firestore 資料模型設計與 CRUD 操作封裝（collection: `workouts`，document per session）
- **Hours 21-25**: 訓練記錄儲存功能：對話完成後觸發儲存，localStorage cache 同步
- **Hours 26-30**: Category Records 頁面：卡片元件、skeleton loading、empty state、從 Firestore 讀取資料
- **Hours 31-32**: Bug fix + RWD 驗證（375px iPhone SE、390px iPhone 14）

### Day 3 (Hours 33-48)

- **Hours 33-36**: 打卡完成流程：「已完成」狀態更新、成功回饋動畫（confetti 或 scale pulse）
- **Hours 37-39**: 錯誤處理：AI API 失敗重試、Firestore 讀取失敗提示
- **Hours 40-42**: 整體 UI polish：動畫、transition、細節對齊
- **Hours 43-45**: Vercel 部署設定、環境變數設定、smoke test
- **Hours 46-47**: Vitest 補寫關鍵 unit test（AI 訊息 parser、資料轉換邏輯）
- **Hours 48**: Demo 準備，確認完整流程可在 5 分鐘內展示

---

## Data Model

```typescript
// Firestore collection: workouts
interface WorkoutRecord {
  id: string              // auto-generated
  date: string            // "2026-03-16"
  category: string        // "上肢" | "下肢" | "核心" | "全身" | "伸展"
  duration: number        // minutes
  equipment: string       // "有器材" | "純徒手"
  name: string            // 預設為日期，可自訂
  exercises: Exercise[]
  completed: boolean      // 打卡狀態
  createdAt: Timestamp
}

interface Exercise {
  name: string            // "槓鈴臥推"
  sets: number            // 4
  reps: number            // 10
  weight: string          // "60kg" | "徒手"
}
```

---

## Risk Log

| Risk | Impact | Mitigation |
|------|--------|-----------|
| AI 輸出 JSON 格式不穩定 | High | System prompt 嚴格規範輸出格式，加 JSON.parse try-catch fallback，必要時 retry |
| OpenAI API 費用超出預期 | Mid | 使用 gpt-4o-mini，設定每日 token 上限，MVP 期間手動監控 |
| Firebase 設定耗時過長 | Mid | Day 1 Hour 1-2 優先完成，卡住立即改用純 localStorage 先行 |
| RWD 在小螢幕破版 | Mid | 全程以 375px 為開發基準，375px / 390px / 768px 三個斷點驗收 |
| 對話流程狀態管理複雜 | Mid | 使用簡單的 step index 狀態機（0-4），避免過度工程化 |
| 側邊欄動畫效能在低階手機卡頓 | Low | 使用 CSS transform 而非 width/left 動畫，啟用 will-change |
