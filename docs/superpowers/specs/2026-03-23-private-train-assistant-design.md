# Private Train Assistant — Design Spec

> **For agentic workers:** 實作前請先閱讀 `docs/PRD.md` 取得完整產品需求。本文件為架構設計決策記錄，補充 PRD 未涵蓋的技術細節。

**Goal:** 以 AI 對話為核心，讓使用者在 5 步驟內獲得當日訓練菜單並記錄至 Firebase。

**Architecture:** Feature-based 模組化。每個功能（chat、records、sidebar）自成一個 feature 資料夾，透過 Pinia stores 共享跨 feature 狀態，services 層純粹處理外部 API 不涉及 UI。

**Tech Stack:** Vue 3 + Vite + TypeScript、Ant Design Vue、Tailwind CSS + SCSS、Google Gemini API (gemini-2.0-flash)、Firebase Firestore、localStorage、Vitest、Vercel

---

## 目錄結構

```
src/
  features/
    chat/
      components/     # MessageBubble、TypingIndicator、QuickReplyChips、InputBar
      composables/    # useChat.ts（對話狀態 + 5步驟狀態機）
      types.ts        # Message、ConversationStep 型別
    records/
      components/     # RecordCard、SkeletonCard、EmptyState
      composables/    # useWorkouts.ts（讀取/寫入訓練記錄）
      types.ts        # WorkoutRecord、Exercise 型別
    sidebar/
      components/     # SidebarNav、CategoryItem
      composables/    # useSidebar.ts（開關狀態）
  services/
    gemini.ts         # Gemini API 呼叫、system prompt、JSON parse
    firebase.ts       # Firestore CRUD 封裝
    localStorage.ts   # localStorage 讀寫、離線 fallback
  stores/
    chat.ts           # Pinia：訊息列表、當前步驟
    workouts.ts       # Pinia：訓練記錄快取
  router/
    index.ts          # 路由：/chat、/records/:category
  assets/
    styles/           # 全域 CSS token、Tailwind 設定
```

**模組邊界規則：**
- `features/` 內的元件只能使用自己 feature 的 composable
- 跨 feature 共享狀態一律透過 `stores/` 傳遞
- `services/` 不知道 UI 的存在，只處理外部資料

---

## AI 對話狀態機

### ConversationStep 型別

```typescript
type ConversationStep =
  | 'idle'           // 初始狀態，顯示歡迎訊息
  | 'ask_category'   // Step 1：今天要練什麼？
  | 'ask_duration'   // Step 2：預計多久？
  | 'ask_equipment'  // Step 3：器材？（伸展時跳過）
  | 'show_menu'      // AI 產出菜單，詢問是否替換
  | 'replace_action' // 使用者要替換動作
  | 'confirm_save'   // Step 5：儲存？
  | 'done'
```

### 步驟流程

```
idle
  └─→ ask_category（顯示歡迎訊息）
        └─→ ask_duration
              ├─→ ask_equipment（選重訓類別時）
              │     └─→ show_menu（呼叫 Gemini，帶入 category + duration + equipment）
              └─→ show_menu（選「伸展」時）
                    ↑ equipment 在此自動設為「純徒手」，由 useChat.ts 在
                      ask_duration → show_menu 的轉換邏輯中賦值：
                      if (category === '伸展') equipment = '純徒手'
                    ├─→ confirm_save（選「不用，就這樣」）
                    └─→ replace_action（選「要替換」）
                          ↑ 進入此狀態時：
                            - Quick reply chips 消失
                            - Input bar placeholder 改為「輸入要替換的動作名稱」
                            - 使用者輸入動作名稱後，儲存至 store.targetExerciseName
                            - 呼叫 Gemini（帶入 currentWorkoutPlan + targetExerciseName）
                            - Gemini 回傳新菜單 → 更新 store.currentWorkoutPlan
                          └─→ show_menu（顯示更新後菜單，可再次替換或儲存）
                                └─→ confirm_save
                                      └─→ done
```

### Quick Reply Chips 對應

| Step | AI 問題 | Quick reply 選項 |
|---|---|---|
| ask_category | 今天要練什麼？ | [上肢] [下肢] [核心] [全身] [伸展] |
| ask_duration | 預計運動多久？ | [30 分鐘] [45 分鐘] [60 分鐘] [90 分鐘] |
| ask_equipment | 需要器材嗎？ | [有器材] [純徒手] |
| show_menu | 需要替換動作嗎？ | [不用，就這樣] [要替換] |
| confirm_save | 要儲存今天的訓練計畫嗎？ | [儲存] [不用了] |

---

## Gemini API 設計

### Replace Action Prompt 設計

`replace_action` 狀態時，送給 Gemini 的 payload：

```typescript
// 包含原始完整菜單 + 指定替換的動作名稱
const prompt = `
目前訓練菜單如下：
${JSON.stringify(currentWorkoutPlan.exercises)}

使用者想替換「${targetExerciseName}」這個動作。
請提供一個替代動作，其他動作保持不變，輸出完整更新後的 JSON 菜單。
`
```

- 使用者可連續替換多個動作（每次替換後回到 `show_menu`，可再選「要替換」）
- 每次替換都帶入**最新版本**的 `currentWorkoutPlan`
- `currentWorkoutPlan` 存於 `stores/chat.ts`

### Gemini API 呼叫時機

| 步驟 | 是否呼叫 | 說明 |
|---|---|---|
| Step 1–3 | ❌ | Pure UI 狀態機，節省 token |
| Step 3 完成後 | ✅ | 帶入 category + duration + equipment 產出菜單 |
| 替換動作 | ✅ | 帶入原菜單 + 指定動作產出替代建議 |
| 儲存 | ❌ | 純 Firestore 寫入 |

整個對話只呼叫 Gemini 1–2 次。

### System Prompt

> **注意：** 步驟 1–3 由前端 UI 狀態機處理，Gemini 不參與。Gemini 只在步驟 3 完成後被呼叫一次，負責產出訓練菜單（或替換動作）。

```
你是一個專業的個人重訓助理，負責根據使用者的訓練條件產出訓練菜單。

規則：
1. 只回覆與訓練相關的問題，拒絕其他話題
2. 產出菜單時，必須以合法 JSON 格式輸出，格式如下：
   {"category":"上肢","duration":45,"equipment":"有器材","exercises":[...]}
3. JSON 之後才能加上自然語言說明（鼓勵語或備注）
4. 動作替換時，只替換指定動作，其他保持不變，輸出完整更新後的菜單 JSON
5. weight 欄位必須輸出具體值（如 "60kg" 或 "徒手"），不可輸出 "自訂"
```

Temperature: `0.3`

### JSON 解析策略

Gemini 回應格式預期為：先輸出 JSON 區塊，再接自然語言說明。
System prompt 嚴格要求 JSON 放在回應最前面，以降低解析失敗率。

```typescript
// gemini.ts — 從 AI 回應中提取第一個完整 JSON 物件
function extractJson(response: string): WorkoutPlan {
  // 找到第一個 { 的位置，從此開始解析
  const start = response.indexOf('{')
  if (start === -1) throw new Error('No JSON found in response')

  // 用括號計數找到對應的結尾 }
  let depth = 0
  let end = -1
  for (let i = start; i < response.length; i++) {
    if (response[i] === '{') depth++
    if (response[i] === '}') depth--
    if (depth === 0) { end = i; break }
  }
  if (end === -1) throw new Error('Malformed JSON in response')

  return JSON.parse(response.slice(start, end + 1))
}
```

失敗時自動 retry 一次，再失敗則顯示錯誤 bubble。

---

## 型別定義

### 共用型別

```typescript
// 定義於 features/chat/types.ts，其他 feature 透過 import 使用

// Exercise 唯一定義，WorkoutPlan 與 WorkoutRecord 共用
interface Exercise {
  name: string    // "槓鈴臥推"
  sets: number    // 4
  reps: number    // 10
  weight: string  // "60kg" | "徒手"（Gemini 須輸出具體值，不可輸出「自訂」）
}

type WorkoutCategory = '上肢' | '下肢' | '核心' | '全身' | '伸展'
type Equipment = '有器材' | '純徒手'
```

### WorkoutPlan（AI 輸出，尚未儲存）

```typescript
// gemini.ts 解析後的暫存型別，用於對話過程中
interface WorkoutPlan {
  category: WorkoutCategory
  duration: number
  equipment: Equipment
  exercises: Exercise[]
}
```

`WorkoutPlan` 在使用者確認儲存後，由 `useChat.ts` 補上 `id`、`name`、`completed`、`createdAt` 欄位，轉為 `WorkoutRecord` 寫入 Firestore。

## 資料模型

```typescript
// WorkoutRecord — Firestore collection: workouts
interface WorkoutRecord {
  id: string
  date: string              // "2026-03-23"
  category: WorkoutCategory
  duration: number          // 分鐘
  equipment: Equipment      // 伸展時自動設為「純徒手」
  name: string              // 預設為日期字串（"2026-03-23"），MVP 不提供自訂 UI
  exercises: Exercise[]
  completed: boolean        // 打卡狀態，初始為 false
  createdAt: Timestamp      // import { Timestamp } from 'firebase/firestore'
}
```

---

## Pinia Store 結構

### `stores/chat.ts`

```typescript
export const useChatStore = defineStore('chat', () => {
  const messages = ref<Message[]>([])       // 所有對話訊息（含 AI + User）
  // Message 型別定義於 features/chat/types.ts：
  // { id: string, role: 'ai' | 'user', content: string, timestamp: Date }

  const currentStep = ref<ConversationStep>('idle')

  // 步驟 1–3 的使用者選擇，呼叫 Gemini 時帶入
  const selectedCategory = ref<WorkoutCategory | null>(null)
  const selectedDuration = ref<number | null>(null)
  const selectedEquipment = ref<Equipment | null>(null)

  const currentWorkoutPlan = ref<WorkoutPlan | null>(null)  // AI 產出的最新菜單
  const targetExerciseName = ref<string | null>(null)       // replace_action 時儲存使用者指定的動作名稱
  const isLoading = ref(false)              // Gemini API 呼叫中

  function resetChat() {
    messages.value = []
    currentStep.value = 'idle'
    selectedCategory.value = null
    selectedDuration.value = null
    selectedEquipment.value = null
    currentWorkoutPlan.value = null
    targetExerciseName.value = null
  }

  return {
    messages, currentStep,
    selectedCategory, selectedDuration, selectedEquipment,
    currentWorkoutPlan, targetExerciseName,
    isLoading, resetChat,
  }
})
```

### `stores/workouts.ts`

```typescript
export const useWorkoutsStore = defineStore('workouts', () => {
  // key: category，value: 該分類的記錄陣列（從 Firestore 讀取後快取）
  const recordsByCategory = ref<Record<string, WorkoutRecord[]>>({})
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  return { recordsByCategory, isLoading, error }
})
```

---

## 錯誤處理

| 情境 | 處理方式 |
|---|---|
| Gemini API 失敗 | 聊天區顯示錯誤 bubble + 「重試」按鈕，重送同一步驟 |
| JSON 解析失敗 | try-catch 後自動 retry 一次，再失敗顯示錯誤訊息 |
| Firestore 寫入失敗 | 先寫入 localStorage，背景 retry，失敗時提示用戶 |
| Firestore 讀取失敗 | 顯示「載入失敗 + 重新整理」，嘗試從 localStorage 讀取 fallback |
| 網路斷線 | localStorage 資料正常顯示，新對話功能 disabled + 提示 |

---

## 打卡完成流程（Feature 5）

**觸發位置：** Category Records 頁面（Screen 3）的每張 RecordCard 底部加入「標記完成」按鈕。

```
RecordCard
  ├─ Card header（日期 + 時間 badge）
  ├─ Completion badge（已完成時顯示）
  ├─ Action list（動作列表）
  └─ [標記完成] 按鈕（completed: false 時顯示，金色文字按鈕）
       └─ 點擊 → 更新 Firestore completed: true
                → 顯示成功動畫（scale pulse + 金色 Completion badge 淡入）
                → 按鈕消失
```

**注意：** PRD Screen 3 說明卡片「不可點擊」是指展開詳細資訊的互動，打卡按鈕是卡片內的獨立 CTA，不衝突。

---

## Firestore 安全規則（MVP）

MVP 不實作登入，使用開放讀寫規則（僅限開發期間）：

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /workouts/{document=**} {
      allow read, write: if true;  // MVP：開放存取，v2 加 Auth 後收緊
    }
  }
}
```

**風險說明：** 任何知道 Firebase project ID 的人可讀寫資料。MVP 個人私用且無敏感資料，可接受。正式上線前必須改為 Auth-based 規則。

---

## 測試策略（Vitest）

MVP 階段只測最高風險的兩個單元：

**1. `services/gemini.ts` — JSON 解析邏輯**
- 正確格式 → 回傳 WorkoutPlan
- JSON 格式跑掉 → 拋出 Error
- 缺少必要欄位 → 拋出 Error 或回傳 null
- 非法 category 值 → 拋出 Error

**2. `features/chat/composables/useChat.ts` — 狀態機轉換**
- 正常流程：idle → ask_category → ask_duration → ask_equipment → show_menu → confirm_save → done
- 伸展流程：ask_duration 後跳過 ask_equipment 直接進 show_menu
- 替換動作分支：show_menu → replace_action → show_menu → confirm_save

UI 元件測試 MVP 略過，等核心穩定後補。
