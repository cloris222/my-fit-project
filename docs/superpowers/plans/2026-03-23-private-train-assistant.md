# Private Train Assistant Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **必讀文件（實作前）：**
> - `docs/PRD.md` — 產品需求、畫面規格、視覺設計
> - `docs/superpowers/specs/2026-03-23-private-train-assistant-design.md` — 架構決策、型別定義、狀態機

**Goal:** 建立以 AI 對話為核心的個人重訓助手 Web App，讓使用者在 5 步驟內獲得當日訓練菜單並記錄至 Firebase。

**Architecture:** Feature-based 模組化（chat / records / sidebar），services 層封裝外部 API，Pinia stores 管理跨 feature 共享狀態。

**Tech Stack:** Vue 3 + Vite + TypeScript、Ant Design Vue、Tailwind CSS v3 + SCSS、Google Gemini API (gemini-2.0-flash)、Firebase Firestore、localStorage、Vitest、Vercel

---

## 檔案結構總覽

```
新建檔案：
src/
  features/
    chat/
      types.ts
      composables/useChat.ts
      components/MessageBubble.vue
      components/TypingIndicator.vue
      components/QuickReplyChips.vue
      components/InputBar.vue
    records/
      types.ts
      composables/useWorkouts.ts
      components/RecordCard.vue
      components/SkeletonCard.vue
      components/EmptyState.vue
    sidebar/
      composables/useSidebar.ts
      components/SidebarNav.vue
  services/
    gemini.ts
    firebase.ts
    localStorage.ts
  stores/
    chat.ts
    workouts.ts
  views/
    ChatView.vue
    RecordsView.vue
  assets/
    styles/
      variables.css
src/__tests__/
  services/gemini.spec.ts
  features/chat/useChat.spec.ts

修改檔案：
src/router/index.ts
src/App.vue
src/main.ts
tailwind.config.ts  (新建)
index.html
.env.example        (新建)
firestore.rules     (新建)
```

---

## Task 1: 安裝依賴與環境設定

**Files:**
- Modify: `package.json`
- Create: `.env.example`
- Modify: `index.html`

- [ ] **Step 1: 安裝所有缺少的依賴**

```bash
npm install ant-design-vue@4 @ant-design/icons-vue firebase @google/generative-ai
npm install -D tailwindcss@3 postcss autoprefixer sass
npx tailwindcss init -p
```

- [ ] **Step 2: 確認安裝成功**

```bash
npm run type-check
```

Expected: 無 type error（忽略 .vue 檔現有錯誤）

- [ ] **Step 3: 建立 `.env.example`**

```bash
# .env.example
VITE_GEMINI_API_KEY=your_gemini_api_key_here
VITE_FIREBASE_API_KEY=your_firebase_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_FIREBASE_APP_ID=your_app_id
```

- [ ] **Step 4: 建立 `.env`（填入真實金鑰，不 commit）**

確認 `.gitignore` 已包含 `.env`：

```bash
grep "^\.env$" .gitignore || echo ".env" >> .gitignore
```

- [ ] **Step 5: Commit**

```bash
git add package.json package-lock.json tailwind.config.js postcss.config.js .env.example .gitignore
git commit -m "chore: install ant-design-vue, firebase, gemini, tailwind, sass"
```

---

## Task 2: Tailwind 設定與全域樣式

**Files:**
- Modify: `tailwind.config.js`
- Create: `src/assets/styles/variables.css`
- Modify: `src/main.ts`
- Modify: `index.html`

- [ ] **Step 1: 設定 `tailwind.config.js`（dark mode + 自訂色彩）**

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{vue,js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        'bg-base': '#0D1117',
        'bg-surface': '#161B22',
        'border-base': '#30363D',
        'text-primary': '#E6EDF3',
        'text-secondary': '#8B949E',
        'brand-blue': '#1E3A5F',
        'brand-gold': '#C9A84C',
      },
      borderRadius: {
        'card': '12px',
        'bubble': '20px',
      },
    },
  },
  plugins: [],
}
```

- [ ] **Step 2: 建立 `src/assets/styles/variables.css`**

```css
/* src/assets/styles/variables.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  html {
    font-family: 'Inter', 'Noto Sans TC', system-ui, sans-serif;
    background-color: #0D1117;
    color: #E6EDF3;
  }

  * {
    box-sizing: border-box;
  }
}
```

- [ ] **Step 3: 更新 `index.html`（加 dark class + Google Fonts）**

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="zh-TW" class="dark">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" href="/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&family=Noto+Sans+TC:wght@400;500;700&display=swap" rel="stylesheet" />
    <title>Private Train Assistant</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

- [ ] **Step 4: 更新 `src/main.ts`（引入 Tailwind CSS + Ant Design Vue）**

```typescript
// src/main.ts
import './assets/styles/variables.css'

import { createApp } from 'vue'
import { createPinia } from 'pinia'
import Antd from 'ant-design-vue'
import 'ant-design-vue/dist/reset.css'

import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(createPinia())
app.use(router)
app.use(Antd)
app.mount('#app')
```

- [ ] **Step 5: 啟動 dev server 確認樣式生效**

```bash
npm run dev
```

Expected: 瀏覽器顯示深色背景，無 console error

- [ ] **Step 6: Commit**

```bash
git add tailwind.config.js postcss.config.js src/assets/ index.html src/main.ts
git commit -m "feat: setup tailwind dark mode and global styles"
```

---

## Task 3: 路由與 App.vue

**Files:**
- Modify: `src/router/index.ts`
- Modify: `src/App.vue`

- [ ] **Step 1: 更新路由**

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      redirect: '/chat',
    },
    {
      path: '/chat',
      name: 'chat',
      component: () => import('@/views/ChatView.vue'),
    },
    {
      path: '/records/:category',
      name: 'records',
      component: () => import('@/views/RecordsView.vue'),
    },
  ],
})

export default router
```

- [ ] **Step 2: 更新 `src/App.vue`（清空模板，只留 RouterView）**

```vue
<!-- src/App.vue -->
<script setup lang="ts"></script>

<template>
  <RouterView />
</template>

<style scoped></style>
```

- [ ] **Step 3: 建立空的 View 佔位元件（讓路由不報錯）**

```bash
mkdir -p src/views
```

建立 `src/views/ChatView.vue`：
```vue
<template><div>Chat</div></template>
```

建立 `src/views/RecordsView.vue`：
```vue
<template><div>Records</div></template>
```

- [ ] **Step 4: 確認路由正常**

```bash
npm run dev
```

瀏覽 `http://localhost:5173/chat` → 顯示「Chat」
瀏覽 `http://localhost:5173/records/上肢` → 顯示「Records」

- [ ] **Step 5: Commit**

```bash
git add src/router/index.ts src/App.vue src/views/
git commit -m "feat: setup router with /chat and /records/:category"
```

---

## Task 4: 共用型別定義

**Files:**
- Create: `src/features/chat/types.ts`
- Create: `src/features/records/types.ts`

- [ ] **Step 1: 建立 `src/features/chat/types.ts`**

```typescript
// src/features/chat/types.ts
import type { Timestamp } from 'firebase/firestore'

export type WorkoutCategory = '上肢' | '下肢' | '核心' | '全身' | '伸展'
export type Equipment = '有器材' | '純徒手'

export type ConversationStep =
  | 'idle'
  | 'ask_category'
  | 'ask_duration'
  | 'ask_equipment'
  | 'show_menu'
  | 'replace_action'
  | 'confirm_save'
  | 'done'

export interface Message {
  id: string
  role: 'ai' | 'user'
  content: string
  timestamp: Date
}

export interface Exercise {
  name: string
  sets: number
  reps: number
  weight: string
}

export interface WorkoutPlan {
  category: WorkoutCategory
  duration: number
  equipment: Equipment
  exercises: Exercise[]
}

export interface WorkoutRecord {
  id: string
  date: string
  category: WorkoutCategory
  duration: number
  equipment: Equipment
  name: string
  exercises: Exercise[]
  completed: boolean
  createdAt: Timestamp
}
```

- [ ] **Step 2: 建立 `src/features/records/types.ts`（re-export 共用型別）**

```typescript
// src/features/records/types.ts
export type { WorkoutRecord, Exercise, WorkoutCategory } from '@/features/chat/types'
```

- [ ] **Step 3: Commit**

```bash
git add src/features/
git commit -m "feat: add shared type definitions"
```

---

## Task 5: Gemini Service（TDD）

**Files:**
- Create: `src/services/gemini.ts`
- Create: `src/__tests__/services/gemini.spec.ts`

- [ ] **Step 1: 建立測試檔 `src/__tests__/services/gemini.spec.ts`**

```typescript
// src/__tests__/services/gemini.spec.ts
import { describe, it, expect } from 'vitest'
import { extractWorkoutPlan } from '@/services/gemini'

describe('extractWorkoutPlan', () => {
  it('正確格式 → 回傳 WorkoutPlan', () => {
    const response = `{"category":"上肢","duration":45,"equipment":"有器材","exercises":[{"name":"槓鈴臥推","sets":4,"reps":10,"weight":"60kg"}]}\n好的，菜單已準備好！`
    const plan = extractWorkoutPlan(response)
    expect(plan.category).toBe('上肢')
    expect(plan.exercises).toHaveLength(1)
    expect(plan.exercises[0].name).toBe('槓鈴臥推')
  })

  it('JSON 前有文字也能解析', () => {
    const response = `以下是菜單：{"category":"下肢","duration":30,"equipment":"純徒手","exercises":[]}`
    const plan = extractWorkoutPlan(response)
    expect(plan.category).toBe('下肢')
  })

  it('沒有 JSON → 拋出 Error', () => {
    expect(() => extractWorkoutPlan('沒有任何 JSON')).toThrow('No JSON found')
  })

  it('不完整的 JSON → 拋出 Error', () => {
    expect(() => extractWorkoutPlan('{"category":"上肢"')).toThrow('Malformed JSON')
  })

  it('缺少 exercises 欄位 → 拋出 Error', () => {
    expect(() => extractWorkoutPlan('{"category":"上肢","duration":45,"equipment":"有器材"}')).toThrow('Invalid workout plan')
  })
})
```

- [ ] **Step 2: 執行測試，確認全部失敗**

```bash
npm run test:unit -- src/__tests__/services/gemini.spec.ts
```

Expected: 5 tests FAIL（extractWorkoutPlan is not defined）

- [ ] **Step 3: 建立 `src/services/gemini.ts`**

```typescript
// src/services/gemini.ts
import { GoogleGenerativeAI } from '@google/generative-ai'
import type { WorkoutPlan, WorkoutCategory, Equipment, Exercise } from '@/features/chat/types'

const genAI = new GoogleGenerativeAI(import.meta.env.VITE_GEMINI_API_KEY)
const model = genAI.getGenerativeModel({
  model: 'gemini-2.0-flash',
  generationConfig: { temperature: 0.3 },
})

const SYSTEM_PROMPT = `你是一個專業的個人重訓助理，負責根據使用者的訓練條件產出訓練菜單。

規則：
1. 只回覆與訓練相關的問題，拒絕其他話題
2. 產出菜單時，必須以合法 JSON 格式輸出，格式如下：
   {"category":"上肢","duration":45,"equipment":"有器材","exercises":[...]}
3. JSON 之後才能加上自然語言說明（鼓勵語或備注）
4. 動作替換時，只替換指定動作，其他保持不變，輸出完整更新後的菜單 JSON
5. weight 欄位必須輸出具體值（如 "60kg" 或 "徒手"），不可輸出 "自訂"`

export function extractWorkoutPlan(response: string): WorkoutPlan {
  const start = response.indexOf('{')
  if (start === -1) throw new Error('No JSON found in response')

  let depth = 0
  let end = -1
  for (let i = start; i < response.length; i++) {
    if (response[i] === '{') depth++
    if (response[i] === '}') depth--
    if (depth === 0) { end = i; break }
  }
  if (end === -1) throw new Error('Malformed JSON in response')

  const parsed = JSON.parse(response.slice(start, end + 1))
  if (!parsed.exercises || !Array.isArray(parsed.exercises)) {
    throw new Error('Invalid workout plan: missing exercises')
  }

  return parsed as WorkoutPlan
}

export async function generateMenu(
  category: WorkoutCategory,
  duration: number,
  equipment: Equipment,
): Promise<WorkoutPlan> {
  const userPrompt = `請根據以下條件產出訓練菜單：
- 部位/類型：${category}
- 時間：${duration} 分鐘
- 器材：${equipment}`

  const result = await model.generateContent(`${SYSTEM_PROMPT}\n\n${userPrompt}`)
  return extractWorkoutPlan(result.response.text())
}

export async function replaceExercise(
  currentPlan: WorkoutPlan,
  targetExerciseName: string,
): Promise<WorkoutPlan> {
  const userPrompt = `目前訓練菜單如下：
${JSON.stringify(currentPlan.exercises)}

使用者想替換「${targetExerciseName}」這個動作。
請提供一個替代動作，其他動作保持不變，輸出完整更新後的 JSON 菜單。`

  const result = await model.generateContent(`${SYSTEM_PROMPT}\n\n${userPrompt}`)
  return extractWorkoutPlan(result.response.text())
}
```

- [ ] **Step 4: 執行測試，確認全部通過**

```bash
npm run test:unit -- src/__tests__/services/gemini.spec.ts
```

Expected: 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add src/services/gemini.ts src/__tests__/services/gemini.spec.ts
git commit -m "feat: add gemini service with extractWorkoutPlan (TDD)"
```

---

## Task 6: Firebase Service

**Files:**
- Create: `src/services/firebase.ts`
- Create: `firestore.rules`

- [ ] **Step 1: 建立 `firestore.rules`**

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /workouts/{document=**} {
      allow read, write: if true;
    }
  }
}
```

- [ ] **Step 2: 建立 `src/services/firebase.ts`**

```typescript
// src/services/firebase.ts
import { initializeApp } from 'firebase/app'
import {
  getFirestore,
  collection,
  addDoc,
  getDocs,
  query,
  where,
  updateDoc,
  doc,
  Timestamp,
  orderBy,
} from 'firebase/firestore'
import type { WorkoutRecord, WorkoutPlan, WorkoutCategory } from '@/features/chat/types'

const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID,
}

const app = initializeApp(firebaseConfig)
const db = getFirestore(app)

export async function saveWorkout(plan: WorkoutPlan): Promise<WorkoutRecord> {
  const today = new Date().toISOString().split('T')[0]
  const record = {
    date: today,
    category: plan.category,
    duration: plan.duration,
    equipment: plan.equipment,
    name: today,
    exercises: plan.exercises,
    completed: false,
    createdAt: Timestamp.now(),
  }

  const docRef = await addDoc(collection(db, 'workouts'), record)
  return { id: docRef.id, ...record } as WorkoutRecord
}

export async function getWorkoutsByCategory(category: WorkoutCategory): Promise<WorkoutRecord[]> {
  const q = query(
    collection(db, 'workouts'),
    where('category', '==', category),
    orderBy('createdAt', 'desc'),
  )
  const snapshot = await getDocs(q)
  return snapshot.docs.map(d => ({ id: d.id, ...d.data() }) as WorkoutRecord)
}

export async function markWorkoutCompleted(id: string): Promise<void> {
  await updateDoc(doc(db, 'workouts', id), { completed: true })
}
```

- [ ] **Step 3: Commit**

```bash
git add src/services/firebase.ts firestore.rules
git commit -m "feat: add firebase service (saveWorkout, getWorkoutsByCategory, markCompleted)"
```

---

## Task 7: localStorage Service

**Files:**
- Create: `src/services/localStorage.ts`

- [ ] **Step 1: 建立 `src/services/localStorage.ts`**

```typescript
// src/services/localStorage.ts
import type { WorkoutRecord, WorkoutCategory } from '@/features/chat/types'

const KEY_PREFIX = 'pta_'

function key(category: WorkoutCategory): string {
  return `${KEY_PREFIX}${category}`
}

export function saveToLocal(record: WorkoutRecord): void {
  const existing = getFromLocal(record.category)
  const updated = [record, ...existing]
  localStorage.setItem(key(record.category), JSON.stringify(updated))
}

export function getFromLocal(category: WorkoutCategory): WorkoutRecord[] {
  const raw = localStorage.getItem(key(category))
  if (!raw) return []
  try {
    return JSON.parse(raw) as WorkoutRecord[]
  } catch {
    return []
  }
}

export function updateLocalCompleted(id: string, category: WorkoutCategory): void {
  const records = getFromLocal(category)
  const updated = records.map(r => r.id === id ? { ...r, completed: true } : r)
  localStorage.setItem(key(category), JSON.stringify(updated))
}
```

- [ ] **Step 2: Commit**

```bash
git add src/services/localStorage.ts
git commit -m "feat: add localStorage service for offline fallback"
```

---

## Task 8: Pinia Stores

**Files:**
- Create: `src/stores/chat.ts`
- Create: `src/stores/workouts.ts`

- [ ] **Step 1: 建立 `src/stores/chat.ts`**

```typescript
// src/stores/chat.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import type {
  Message,
  ConversationStep,
  WorkoutPlan,
  WorkoutCategory,
  Equipment,
} from '@/features/chat/types'

export const useChatStore = defineStore('chat', () => {
  const messages = ref<Message[]>([])
  const currentStep = ref<ConversationStep>('idle')

  // 步驟 1–3 的使用者選擇，呼叫 Gemini 時帶入
  const selectedCategory = ref<WorkoutCategory | null>(null)
  const selectedDuration = ref<number | null>(null)
  const selectedEquipment = ref<Equipment | null>(null)

  const currentWorkoutPlan = ref<WorkoutPlan | null>(null)
  const targetExerciseName = ref<string | null>(null)
  const isLoading = ref(false)

  function resetChat() {
    messages.value = []
    currentStep.value = 'idle'
    selectedCategory.value = null
    selectedDuration.value = null
    selectedEquipment.value = null
    currentWorkoutPlan.value = null
    targetExerciseName.value = null
    isLoading.value = false
  }

  return {
    messages,
    currentStep,
    selectedCategory,
    selectedDuration,
    selectedEquipment,
    currentWorkoutPlan,
    targetExerciseName,
    isLoading,
    resetChat,
  }
})
```

- [ ] **Step 2: 建立 `src/stores/workouts.ts`**

```typescript
// src/stores/workouts.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'
import type { WorkoutRecord, WorkoutCategory } from '@/features/chat/types'

export const useWorkoutsStore = defineStore('workouts', () => {
  const recordsByCategory = ref<Record<string, WorkoutRecord[]>>({})
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  return { recordsByCategory, isLoading, error }
})
```

- [ ] **Step 3: Commit**

```bash
git add src/stores/
git commit -m "feat: add chat and workouts pinia stores"
```

---

## Task 9: useChat Composable（TDD）

**Files:**
- Create: `src/features/chat/composables/useChat.ts`
- Create: `src/__tests__/features/chat/useChat.spec.ts`

- [ ] **Step 1: 建立測試檔**

```typescript
// src/__tests__/features/chat/useChat.spec.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useChat } from '@/features/chat/composables/useChat'

// Mock gemini service
vi.mock('@/services/gemini', () => ({
  generateMenu: vi.fn().mockResolvedValue({
    category: '上肢',
    duration: 45,
    equipment: '有器材',
    exercises: [{ name: '槓鈴臥推', sets: 4, reps: 10, weight: '60kg' }],
  }),
  replaceExercise: vi.fn().mockResolvedValue({
    category: '上肢',
    duration: 45,
    equipment: '有器材',
    exercises: [{ name: '啞鈴飛鳥', sets: 4, reps: 10, weight: '15kg' }],
  }),
}))

vi.mock('@/services/firebase', () => ({
  saveWorkout: vi.fn().mockResolvedValue({ id: 'test-id' }),
}))

vi.mock('@/services/localStorage', () => ({
  saveToLocal: vi.fn(),
}))

describe('useChat - 狀態機', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('初始狀態為 idle', () => {
    const { store } = useChat()
    expect(store.currentStep).toBe('idle')
  })

  it('正常流程：idle → ask_category → ask_duration → ask_equipment → show_menu', async () => {
    const { store, handleUserInput } = useChat()

    // 觸發對話開始
    await handleUserInput('start')
    expect(store.currentStep).toBe('ask_category')

    await handleUserInput('上肢')
    expect(store.currentStep).toBe('ask_duration')
    expect(store.selectedCategory).toBe('上肢')

    await handleUserInput('45 分鐘')
    expect(store.currentStep).toBe('ask_equipment')
    expect(store.selectedDuration).toBe(45)

    await handleUserInput('有器材')
    expect(store.currentStep).toBe('show_menu')
    expect(store.selectedEquipment).toBe('有器材')
    expect(store.currentWorkoutPlan).not.toBeNull()
  })

  it('伸展流程：ask_duration 後跳過 ask_equipment', async () => {
    const { store, handleUserInput } = useChat()

    await handleUserInput('start')
    await handleUserInput('伸展')
    await handleUserInput('30 分鐘')

    expect(store.currentStep).toBe('show_menu')
    expect(store.selectedEquipment).toBe('純徒手')
  })

  it('替換動作分支：show_menu → replace_action → show_menu', async () => {
    const { store, handleUserInput } = useChat()

    await handleUserInput('start')
    await handleUserInput('上肢')
    await handleUserInput('45 分鐘')
    await handleUserInput('有器材')

    expect(store.currentStep).toBe('show_menu')

    await handleUserInput('要替換')
    expect(store.currentStep).toBe('replace_action')

    await handleUserInput('槓鈴臥推')
    expect(store.currentStep).toBe('show_menu')
    expect(store.currentWorkoutPlan?.exercises[0].name).toBe('啞鈴飛鳥')
  })
})
```

- [ ] **Step 2: 執行測試，確認失敗**

```bash
npm run test:unit -- src/__tests__/features/chat/useChat.spec.ts
```

Expected: FAIL（useChat not found）

- [ ] **Step 3: 建立 `src/features/chat/composables/useChat.ts`**

```typescript
// src/features/chat/composables/useChat.ts
import { useChatStore } from '@/stores/chat'
import { generateMenu, replaceExercise } from '@/services/gemini'
import { saveWorkout } from '@/services/firebase'
import { saveToLocal } from '@/services/localStorage'
import type { Message } from '@/features/chat/types'

function createMessage(role: 'ai' | 'user', content: string): Message {
  return { id: crypto.randomUUID(), role, content, timestamp: new Date() }
}

function parseDuration(input: string): number {
  const match = input.match(/(\d+)/)
  return match ? parseInt(match[1]) : 30
}

export function useChat() {
  const store = useChatStore()

  async function handleUserInput(input: string): Promise<void> {
    const step = store.currentStep

    if (step === 'idle') {
      store.messages.push(createMessage('ai', '嗨！我是你的訓練助理。今天要練什麼？'))
      store.currentStep = 'ask_category'
      return
    }

    store.messages.push(createMessage('user', input))

    if (step === 'ask_category') {
      store.selectedCategory = input as typeof store.selectedCategory
      store.messages.push(createMessage('ai', '預計運動多久？'))
      store.currentStep = 'ask_duration'
      return
    }

    if (step === 'ask_duration') {
      store.selectedDuration = parseDuration(input)

      if (store.selectedCategory === '伸展') {
        store.selectedEquipment = '純徒手'
        store.currentStep = 'show_menu'
        await callGenerateMenu()
        return
      }

      store.messages.push(createMessage('ai', '需要器材（啞鈴/槓鈴/機器）還是純徒手？'))
      store.currentStep = 'ask_equipment'
      return
    }

    if (step === 'ask_equipment') {
      store.selectedEquipment = input as typeof store.selectedEquipment
      store.currentStep = 'show_menu'
      await callGenerateMenu()
      return
    }

    if (step === 'show_menu') {
      if (input === '要替換') {
        store.messages.push(createMessage('ai', '哪個動作需要替換？請輸入動作名稱。'))
        store.currentStep = 'replace_action'
        return
      }
      // 不替換 → 確認儲存
      store.messages.push(createMessage('ai', '菜單已準備好！要儲存今天的訓練計畫嗎？'))
      store.currentStep = 'confirm_save'
      return
    }

    if (step === 'replace_action') {
      store.targetExerciseName = input
      store.isLoading = true
      try {
        const updated = await replaceExercise(store.currentWorkoutPlan!, input)
        store.currentWorkoutPlan = updated
        store.messages.push(createMessage('ai', `已替換完成！更新後的菜單如上。需要再替換其他動作嗎？`))
        store.currentStep = 'show_menu'
      } finally {
        store.isLoading = false
      }
      return
    }

    if (step === 'confirm_save') {
      if (input === '儲存') {
        await handleSave()
        return
      }
      store.messages.push(createMessage('ai', '好的，沒有儲存。下次加油！'))
      store.currentStep = 'done'
      return
    }
  }

  async function callGenerateMenu() {
    store.isLoading = true
    try {
      const plan = await generateMenu(
        store.selectedCategory!,
        store.selectedDuration!,
        store.selectedEquipment!,
      )
      store.currentWorkoutPlan = plan
      store.messages.push(createMessage('ai', `菜單已產出！需要替換其中任何動作嗎？`))
    } finally {
      store.isLoading = false
    }
  }

  async function handleSave() {
    if (!store.currentWorkoutPlan) return
    try {
      const record = await saveWorkout(store.currentWorkoutPlan)
      saveToLocal(record)
      store.messages.push(createMessage('ai', '✅ 已儲存！可以在側邊欄查看記錄。'))
      store.currentStep = 'done'
    } catch {
      store.messages.push(createMessage('ai', '連線發生問題，請點擊重試。'))
    }
  }

  return { store, handleUserInput }
}
```

- [ ] **Step 4: 執行測試，確認全部通過**

```bash
npm run test:unit -- src/__tests__/features/chat/useChat.spec.ts
```

Expected: 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git add src/features/chat/composables/ src/__tests__/features/chat/
git commit -m "feat: add useChat composable with 5-step state machine (TDD)"
```

---

## Task 10: Chat UI 元件

**Files:**
- Create: `src/features/chat/components/MessageBubble.vue`
- Create: `src/features/chat/components/TypingIndicator.vue`
- Create: `src/features/chat/components/QuickReplyChips.vue`
- Create: `src/features/chat/components/InputBar.vue`

- [ ] **Step 1: 建立 `MessageBubble.vue`**

```vue
<!-- src/features/chat/components/MessageBubble.vue -->
<script setup lang="ts">
import type { Message } from '@/features/chat/types'

const props = defineProps<{ message: Message }>()
</script>

<template>
  <div
    class="flex mb-3"
    :class="props.message.role === 'user' ? 'justify-end' : 'justify-start'"
  >
    <div
      class="max-w-[80%] px-4 py-3 rounded-bubble text-sm leading-relaxed"
      :class="
        props.message.role === 'user'
          ? 'bg-brand-blue text-text-primary'
          : 'bg-bg-surface border border-border-base text-text-primary'
      "
    >
      {{ props.message.content }}
    </div>
  </div>
</template>
```

- [ ] **Step 2: 建立 `TypingIndicator.vue`**

```vue
<!-- src/features/chat/components/TypingIndicator.vue -->
<template>
  <div class="flex justify-start mb-3">
    <div class="bg-bg-surface border border-border-base rounded-bubble px-4 py-3 flex gap-1 items-center">
      <span
        v-for="i in 3"
        :key="i"
        class="w-2 h-2 bg-text-secondary rounded-full animate-bounce"
        :style="{ animationDelay: `${(i - 1) * 0.15}s` }"
      />
    </div>
  </div>
</template>
```

- [ ] **Step 3: 建立 `QuickReplyChips.vue`**

```vue
<!-- src/features/chat/components/QuickReplyChips.vue -->
<script setup lang="ts">
const props = defineProps<{ chips: string[] }>()
const emit = defineEmits<{ select: [value: string] }>()
</script>

<template>
  <div class="flex flex-wrap gap-2 mb-3 pl-2">
    <button
      v-for="chip in props.chips"
      :key="chip"
      class="px-4 py-2 rounded-bubble text-sm text-brand-gold border border-brand-gold bg-bg-surface active:scale-95 transition-transform"
      @click="emit('select', chip)"
    >
      {{ chip }}
    </button>
  </div>
</template>
```

- [ ] **Step 4: 建立 `InputBar.vue`**

```vue
<!-- src/features/chat/components/InputBar.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{ disabled?: boolean; placeholder?: string }>()
const emit = defineEmits<{ submit: [value: string] }>()

const text = ref('')

function handleSubmit() {
  const trimmed = text.value.trim()
  if (!trimmed) return
  emit('submit', trimmed)
  text.value = ''
}
</script>

<template>
  <div class="fixed bottom-0 left-0 right-0 h-[72px] bg-bg-base border-t border-border-base flex items-center px-4 gap-3">
    <textarea
      v-model="text"
      rows="1"
      :placeholder="props.placeholder ?? '輸入訊息...'"
      :disabled="props.disabled"
      class="flex-1 bg-bg-surface border border-border-base rounded-lg px-3 py-2 text-sm text-text-primary placeholder-text-secondary resize-none outline-none focus:border-brand-gold"
      @keydown.enter.prevent="handleSubmit"
    />
    <button
      :disabled="props.disabled || !text.trim()"
      class="text-brand-gold disabled:opacity-40 transition-opacity"
      @click="handleSubmit"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
        <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"/>
      </svg>
    </button>
  </div>
</template>
```

- [ ] **Step 5: Commit**

```bash
git add src/features/chat/components/
git commit -m "feat: add chat UI components (MessageBubble, TypingIndicator, QuickReplyChips, InputBar)"
```

---

## Task 11: Sidebar 元件

**Files:**
- Create: `src/features/sidebar/composables/useSidebar.ts`
- Create: `src/features/sidebar/components/SidebarNav.vue`

- [ ] **Step 1: 建立 `useSidebar.ts`**

```typescript
// src/features/sidebar/composables/useSidebar.ts
import { ref } from 'vue'

const isOpen = ref(false)

export function useSidebar() {
  function open() { isOpen.value = true }
  function close() { isOpen.value = false }
  function toggle() { isOpen.value = !isOpen.value }

  return { isOpen, open, close, toggle }
}
```

- [ ] **Step 2: 建立 `SidebarNav.vue`**

```vue
<!-- src/features/sidebar/components/SidebarNav.vue -->
<script setup lang="ts">
import { useRouter } from 'vue-router'
import { useSidebar } from '@/features/sidebar/composables/useSidebar'
import { useChatStore } from '@/stores/chat'

const categories = ['上肢', '下肢', '核心', '全身', '伸展']

const router = useRouter()
const { isOpen, close } = useSidebar()
const chatStore = useChatStore()

function goToCategory(category: string) {
  router.push(`/records/${category}`)
  close()
}

function startNewChat() {
  chatStore.resetChat()
  router.push('/chat')
  close()
}
</script>

<template>
  <!-- Backdrop -->
  <Transition name="fade">
    <div
      v-if="isOpen"
      class="fixed inset-0 bg-black/60 z-40 md:hidden"
      @click="close"
    />
  </Transition>

  <!-- Sidebar -->
  <Transition name="slide">
    <div
      v-if="isOpen"
      class="fixed left-0 top-0 h-full w-[280px] bg-bg-surface z-50 flex flex-col"
    >
      <!-- Header -->
      <div class="h-14 flex items-center justify-between px-4 border-b border-border-base">
        <span class="text-base font-medium text-text-primary">訓練記錄</span>
        <button class="text-text-secondary md:hidden" @click="close">✕</button>
      </div>

      <!-- New Chat Button -->
      <div class="p-4">
        <button
          class="w-full h-11 rounded-lg bg-brand-blue text-text-primary text-sm font-medium hover:bg-blue-800 transition-colors"
          @click="startNewChat"
        >
          + 新對話
        </button>
      </div>

      <div class="border-t border-border-base" />

      <!-- Category List -->
      <nav class="flex-1 py-2">
        <button
          v-for="cat in categories"
          :key="cat"
          class="w-full h-12 flex items-center px-4 text-sm text-text-secondary hover:bg-brand-blue hover:text-text-primary transition-colors"
          @click="goToCategory(cat)"
        >
          {{ cat }}訓練
        </button>
      </nav>

      <div class="border-t border-border-base" />
    </div>
  </Transition>
</template>

<style scoped>
.fade-enter-active, .fade-leave-active { transition: opacity 0.25s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

.slide-enter-active, .slide-leave-active { transition: transform 0.25s ease-out; }
.slide-enter-from, .slide-leave-to { transform: translateX(-100%); }
</style>
```

- [ ] **Step 3: Commit**

```bash
git add src/features/sidebar/
git commit -m "feat: add sidebar with category navigation and new chat"
```

---

## Task 12: ChatView（組裝主畫面）

**Files:**
- Modify: `src/views/ChatView.vue`

- [ ] **Step 1: 實作完整 `ChatView.vue`**

```vue
<!-- src/views/ChatView.vue -->
<script setup lang="ts">
import { ref, nextTick, computed, onMounted } from 'vue'
import { useChat } from '@/features/chat/composables/useChat'
import { useSidebar } from '@/features/sidebar/composables/useSidebar'
import MessageBubble from '@/features/chat/components/MessageBubble.vue'
import TypingIndicator from '@/features/chat/components/TypingIndicator.vue'
import QuickReplyChips from '@/features/chat/components/QuickReplyChips.vue'
import InputBar from '@/features/chat/components/InputBar.vue'
import SidebarNav from '@/features/sidebar/components/SidebarNav.vue'

const { store, handleUserInput } = useChat()
const { toggle } = useSidebar()

const messageList = ref<HTMLElement | null>(null)

// Quick reply chips 依目前步驟決定
const chips = computed(() => {
  switch (store.currentStep) {
    case 'ask_category': return ['上肢', '下肢', '核心', '全身', '伸展']
    case 'ask_duration': return ['30 分鐘', '45 分鐘', '60 分鐘', '90 分鐘']
    case 'ask_equipment': return ['有器材', '純徒手']
    case 'show_menu': return ['不用，就這樣', '要替換']
    case 'confirm_save': return ['儲存', '不用了']
    default: return []
  }
})

// replace_action 時提示輸入動作名稱
const inputPlaceholder = computed(() =>
  store.currentStep === 'replace_action' ? '輸入要替換的動作名稱' : '輸入訊息...'
)

async function scrollToBottom() {
  await nextTick()
  messageList.value?.scrollTo({ top: messageList.value.scrollHeight, behavior: 'smooth' })
}

async function onSubmit(text: string) {
  await handleUserInput(text)
  scrollToBottom()
}

async function onChipSelect(chip: string) {
  await handleUserInput(chip)
  scrollToBottom()
}

onMounted(async () => {
  if (store.currentStep === 'idle') {
    await handleUserInput('start')
    scrollToBottom()
  }
})
</script>

<template>
  <div class="flex flex-col h-screen bg-bg-base">
    <SidebarNav />

    <!-- Header -->
    <header class="h-14 flex items-center px-4 border-b border-border-base shrink-0">
      <button class="text-text-secondary mr-3" @click="toggle">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="currentColor" viewBox="0 0 24 24">
          <path d="M3 18h18v-2H3v2zm0-5h18v-2H3v2zm0-7v2h18V6H3z"/>
        </svg>
      </button>
      <span class="text-base font-bold text-text-primary">Private Train Assistant</span>
    </header>

    <!-- Message List -->
    <div
      ref="messageList"
      class="flex-1 overflow-y-auto px-4 pt-4 pb-[88px]"
    >
      <MessageBubble
        v-for="msg in store.messages"
        :key="msg.id"
        :message="msg"
      />
      <TypingIndicator v-if="store.isLoading" />

      <!-- Quick Reply Chips（顯示在最後一條 AI 訊息下方）-->
      <QuickReplyChips
        v-if="chips.length && !store.isLoading"
        :chips="chips"
        @select="onChipSelect"
      />
    </div>

    <InputBar
      :disabled="store.isLoading"
      :placeholder="inputPlaceholder"
      @submit="onSubmit"
    />
  </div>
</template>
```

- [ ] **Step 2: 在瀏覽器測試完整對話流程**

```bash
npm run dev
```

手動測試：
- 開啟 `http://localhost:5173/chat`
- 確認歡迎訊息出現 + 5 個 Quick reply chips
- 點擊「上肢」→ 出現時間詢問
- 點擊「45 分鐘」→ 出現器材詢問
- 點擊「有器材」→ AI 呼叫（需真實 API key）→ 出現菜單

- [ ] **Step 3: Commit**

```bash
git add src/views/ChatView.vue
git commit -m "feat: implement ChatView with full conversation UI"
```

---

## Task 13: useWorkouts Composable + Records UI

**Files:**
- Create: `src/features/records/composables/useWorkouts.ts`
- Create: `src/features/records/components/RecordCard.vue`
- Create: `src/features/records/components/SkeletonCard.vue`
- Create: `src/features/records/components/EmptyState.vue`

- [ ] **Step 1: 建立 `useWorkouts.ts`**

```typescript
// src/features/records/composables/useWorkouts.ts
import { useWorkoutsStore } from '@/stores/workouts'
import { getWorkoutsByCategory, markWorkoutCompleted } from '@/services/firebase'
import { getFromLocal, updateLocalCompleted } from '@/services/localStorage'
import type { WorkoutCategory } from '@/features/chat/types'

export function useWorkouts() {
  const store = useWorkoutsStore()

  async function fetchByCategory(category: WorkoutCategory) {
    store.isLoading = true
    store.error = null
    try {
      const records = await getWorkoutsByCategory(category)
      store.recordsByCategory[category] = records
    } catch {
      // Firestore 失敗 → fallback localStorage
      store.recordsByCategory[category] = getFromLocal(category)
      store.error = '載入失敗，顯示離線資料'
    } finally {
      store.isLoading = false
    }
  }

  async function markCompleted(id: string, category: WorkoutCategory) {
    try {
      await markWorkoutCompleted(id)
      updateLocalCompleted(id, category)
      // 更新 store 快取
      const records = store.recordsByCategory[category] ?? []
      const idx = records.findIndex(r => r.id === id)
      if (idx !== -1) records[idx] = { ...records[idx], completed: true }
    } catch {
      store.error = '標記失敗，請重試'
    }
  }

  return { store, fetchByCategory, markCompleted }
}
```

- [ ] **Step 2: 建立 `RecordCard.vue`**

```vue
<!-- src/features/records/components/RecordCard.vue -->
<script setup lang="ts">
import type { WorkoutRecord } from '@/features/chat/types'

const props = defineProps<{
  record: WorkoutRecord
}>()

const emit = defineEmits<{ markCompleted: [id: string] }>()
</script>

<template>
  <div class="bg-bg-surface border border-border-base rounded-card p-4 mb-3">
    <!-- Card Header -->
    <div class="flex items-center justify-between mb-2">
      <span class="text-base font-medium text-text-primary">{{ props.record.name }}</span>
      <div class="flex items-center gap-2">
        <span
          v-if="props.record.completed"
          class="text-xs text-brand-gold border border-brand-gold rounded-full px-2 py-0.5"
        >已完成</span>
        <span class="text-xs text-brand-gold bg-brand-blue rounded-full px-2.5 py-1">
          {{ props.record.duration }} 分鐘
        </span>
      </div>
    </div>

    <div class="border-t border-border-base my-2" />

    <!-- Exercise List -->
    <div class="space-y-1">
      <div
        v-for="ex in props.record.exercises"
        :key="ex.name"
        class="flex items-center justify-between text-sm"
      >
        <span class="text-text-primary">{{ ex.name }}</span>
        <div class="flex items-center gap-2">
          <span class="text-text-secondary">{{ ex.sets }}組 × {{ ex.reps }}下</span>
          <span class="text-brand-gold text-xs">{{ ex.weight }}</span>
        </div>
      </div>
    </div>

    <!-- Mark Complete Button -->
    <button
      v-if="!props.record.completed"
      class="mt-3 w-full h-9 text-sm text-brand-gold border-t border-border-base pt-3"
      @click="emit('markCompleted', props.record.id)"
    >
      標記完成
    </button>
  </div>
</template>
```

- [ ] **Step 3: 建立 `SkeletonCard.vue`**

```vue
<!-- src/features/records/components/SkeletonCard.vue -->
<template>
  <div class="bg-bg-surface border border-border-base rounded-card p-4 mb-3 animate-pulse">
    <div class="h-4 bg-border-base rounded w-1/3 mb-3" />
    <div class="border-t border-border-base my-2" />
    <div class="space-y-2">
      <div class="h-3 bg-border-base rounded w-full" />
      <div class="h-3 bg-border-base rounded w-4/5" />
      <div class="h-3 bg-border-base rounded w-3/5" />
    </div>
  </div>
</template>
```

- [ ] **Step 4: 建立 `EmptyState.vue`**

```vue
<!-- src/features/records/components/EmptyState.vue -->
<template>
  <div class="flex flex-col items-center justify-center h-full gap-3 text-text-secondary">
    <span class="text-4xl">🏋️</span>
    <p class="text-base">還沒有訓練記錄</p>
    <p class="text-sm">點擊右下角開始安排今天的菜單</p>
  </div>
</template>
```

- [ ] **Step 5: Commit**

```bash
git add src/features/records/
git commit -m "feat: add useWorkouts composable and records UI components"
```

---

## Task 14: RecordsView（組裝記錄頁面）

**Files:**
- Modify: `src/views/RecordsView.vue`

- [ ] **Step 1: 實作 `RecordsView.vue`**

```vue
<!-- src/views/RecordsView.vue -->
<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useWorkouts } from '@/features/records/composables/useWorkouts'
import RecordCard from '@/features/records/components/RecordCard.vue'
import SkeletonCard from '@/features/records/components/SkeletonCard.vue'
import EmptyState from '@/features/records/components/EmptyState.vue'
import type { WorkoutCategory } from '@/features/chat/types'

const route = useRoute()
const router = useRouter()
const { store, fetchByCategory, markCompleted } = useWorkouts()

const category = computed(() => route.params.category as WorkoutCategory)
const records = computed(() => store.recordsByCategory[category.value] ?? [])

onMounted(() => fetchByCategory(category.value))

async function onMarkCompleted(id: string) {
  await markCompleted(id, category.value)
}
</script>

<template>
  <div class="flex flex-col h-screen bg-bg-base">
    <!-- Header -->
    <header class="h-14 flex items-center px-4 border-b border-border-base shrink-0">
      <button class="text-text-secondary mr-3" @click="router.back()">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="currentColor" viewBox="0 0 24 24">
          <path d="M20 11H7.83l5.59-5.59L12 4l-8 8 8 8 1.41-1.41L7.83 13H20v-2z"/>
        </svg>
      </button>
      <span class="text-base font-bold text-text-primary">{{ category }}訓練</span>
    </header>

    <!-- Content -->
    <div class="flex-1 overflow-y-auto px-4 pt-4 pb-20">
      <!-- Loading -->
      <template v-if="store.isLoading">
        <SkeletonCard v-for="i in 3" :key="i" />
      </template>

      <!-- Error -->
      <div v-else-if="store.error" class="text-center text-text-secondary py-8">
        {{ store.error }}
        <button class="block mx-auto mt-2 text-brand-gold text-sm" @click="fetchByCategory(category)">
          重新整理
        </button>
      </div>

      <!-- Empty -->
      <EmptyState v-else-if="records.length === 0" />

      <!-- Records -->
      <template v-else>
        <RecordCard
          v-for="record in records"
          :key="record.id"
          :record="record"
          @mark-completed="onMarkCompleted"
        />
      </template>
    </div>

    <!-- FAB -->
    <button
      class="fixed right-4 bottom-4 w-14 h-14 rounded-full bg-brand-gold text-white text-2xl shadow-lg flex items-center justify-center"
      @click="router.push('/chat')"
    >
      +
    </button>
  </div>
</template>
```

- [ ] **Step 2: 手動測試記錄頁面**

```bash
npm run dev
```

- 完成一次對話並儲存記錄
- 開啟側邊欄 → 點擊「上肢訓練」
- 確認記錄卡片顯示正確
- 點擊「標記完成」→ 確認按鈕消失、「已完成」badge 出現

- [ ] **Step 3: Commit**

```bash
git add src/views/RecordsView.vue
git commit -m "feat: implement RecordsView with loading, empty, and error states"
```

---

## Task 15: 全端整合測試與 type-check

- [ ] **Step 1: 執行所有測試**

```bash
npm run test:unit
```

Expected: 全部 PASS

- [ ] **Step 2: Type check**

```bash
npm run type-check
```

Expected: 無 error

- [ ] **Step 3: 手動整合測試（完整流程）**

```bash
npm run dev
```

驗收清單：
1. 開啟 App → 歡迎訊息 + 5 個 Quick reply chips 顯示
2. 點擊「上肢」→「45 分鐘」→「有器材」→ Gemini 產出菜單
3. 點擊「不用，就這樣」→「儲存」→ 成功訊息
4. 開啟側邊欄 → 點擊「上肢訓練」→ 顯示剛儲存的記錄卡片
5. 點擊「標記完成」→ 卡片更新
6. 測試伸展流程（跳過器材步驟）
7. 測試替換動作流程

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "test: verify full integration flow passes"
```

---

## Task 16: Vercel 部署

**Files:**
- Create: `vercel.json`

- [ ] **Step 1: 建立 `vercel.json`（SPA 路由設定）**

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

- [ ] **Step 2: Build 確認**

```bash
npm run build
```

Expected: `dist/` 資料夾建立成功，無 error

- [ ] **Step 3: 部署至 Vercel**

```bash
npx vercel --prod
```

在 Vercel dashboard 設定環境變數：
- `VITE_GEMINI_API_KEY`
- `VITE_FIREBASE_API_KEY`
- `VITE_FIREBASE_AUTH_DOMAIN`
- `VITE_FIREBASE_PROJECT_ID`
- `VITE_FIREBASE_STORAGE_BUCKET`
- `VITE_FIREBASE_MESSAGING_SENDER_ID`
- `VITE_FIREBASE_APP_ID`

- [ ] **Step 4: 在手機瀏覽器（375px）測試**

驗收清單（PRD 成功標準）：
- [ ] 從開啟 App 到獲得菜單不超過 5 個對話來回
- [ ] 訓練打卡後有明顯視覺回饋
- [ ] 手機瀏覽器（375px）正常操作
- [ ] 可在 5 分鐘 Demo 中展示完整流程

- [ ] **Step 5: Commit**

```bash
git add vercel.json
git commit -m "chore: add vercel config for SPA routing"
git push
```
