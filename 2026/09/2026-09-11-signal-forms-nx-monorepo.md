# Signal Forms 穩定版 × Nx 23.2 Agent-friendly CI｜2026-09-11

> 今日主軸：Angular Signal Forms 已進入穩定階段；同時 Nx 23.2 帶來更好的快取、較精簡的 CI 輸出，以及更適合 AI Agent 工作流的能力。今天把兩件事串起來：**Angular 表單開發模型 + Nx Monorepo 開發/CI 效率**。

---

## ⭐ 今日最值得看的 Angular 技術

今天最值得注意的 Angular 變化，不只是「又出了一個新版本」，而是 **Signal Forms 已經從實驗性方向走到穩定能力**。

Angular 官方 roadmap 現在把 Signal Forms 列為 completed，並說明它已穩定化，同時強化與 Reactive Forms 的互通，讓既有大型表單可以逐步遷移，而不是一次全部重寫。Signal Forms 需要 Angular v21 或以上。citeturn1search2turn1search3

這對企業 Angular 專案很重要：

```text
既有大型系統
   ↓
Reactive Forms 繼續維護
   ↓
新功能優先評估 Signal Forms
   ↓
逐步建立 Signals-first 的表單模式
```

因此今天不要把 Signal Forms 理解成「Reactive Forms 的淘汰品」，而是把它看成 Angular Signals 生態中更完整的一種表單建模方式。

---

# 🔍 核心概念 / API

## 1. `signal()` 是表單資料模型

先建立真正的資料模型：

```ts
interface UserFormModel {
  name: string;
  email: string;
  department: string;
}

userModel = signal<UserFormModel>({
  name: '',
  email: '',
  department: '',
});
```

這裡最重要的觀念是：

> **表單資料不是藏在一堆 control 裡，而是有一個明確的 signal model。**

---

## 2. `form()` 建立 FieldTree

```ts
userForm = form(this.userModel);
```

Angular 會根據資料模型建立 FieldTree：

```text
userForm
├── name
├── email
└── department
```

HTML 可以直接綁定：

```html
<input [formField]="userForm.name" />
<input [formField]="userForm.email" />
<input [formField]="userForm.department" />
```

官方文件把這種模式描述為：表單欄位會與資料模型自動同步，同時提供 type-safe field access。citeturn1search3

---

## 3. Validation 集中在 schema

Signal Forms 的另一個核心方向是把 validation 規則集中管理，而不是讓每個 HTML input 各自散落驗證邏輯。

例如：

```ts
import {
  email,
  form,
  FormField,
  required,
} from '@angular/forms/signals';

userForm = form(this.userModel, (schema) => {
  required(schema.name);
  required(schema.email);
  email(schema.email);
});
```

概念上變成：

```text
Model
  ↓
Form Tree
  ↓
Validation Schema
  ↓
Field State
  ↓
UI
```

這讓 validation 不需要散落在 component template 裡。

---

# 💻 可以直接實作的完整範例

下面是一個可以直接放進 Angular component 的最小使用方式：

```ts
import { Component, signal } from '@angular/core';
import {
  email,
  form,
  FormField,
  required,
} from '@angular/forms/signals';

interface UserFormModel {
  name: string;
  email: string;
  department: string;
}

@Component({
  selector: 'app-user-form',
  imports: [FormField],
  template: `
    <form (submit)="$event.preventDefault()">
      <label>
        姓名
        <input [formField]="userForm.name" />
      </label>

      <label>
        Email
        <input type="email" [formField]="userForm.email" />
      </label>

      <label>
        部門
        <input [formField]="userForm.department" />
      </label>

      <pre>{{ userModel() | json }}</pre>
    </form>
  `,
})
export class UserFormComponent {
  userModel = signal<UserFormModel>({
    name: '',
    email: '',
    department: '',
  });

  userForm = form(this.userModel, (schema) => {
    required(schema.name);
    required(schema.email);
    email(schema.email);
    required(schema.department);
  });
}
```

實務上要注意：如果 template 使用 `json` pipe，記得依專案設定匯入 `JsonPipe`；這裡的重點是 Signal Forms 的 model、FieldTree 與 validation 關係。

---

# 🆕 Signal Forms 教學 — Day 2

昨天是 Day 1：

```text
signal()
   ↓
form()
   ↓
formField
```

今天進入 **Day 2：Form Model + FieldTree + 基本 Validation**。

## 今天一定要會的 4 個東西

```text
1. signal()     → 表單資料
2. form()       → FieldTree
3. formField    → HTML binding
4. required()   → 基本 validation
```

### 練習 1：新增 required

```ts
profile = signal({
  name: '',
  email: '',
});

profileForm = form(profile, (schema) => {
  required(schema.name);
  required(schema.email);
});
```

### 練習 2：加入 email validation

```ts
email(schema.email);
```

所以今天的心智模型是：

```text
interface / model
       ↓
signal(model)
       ↓
form(model, schema)
       ↓
required / email / ...
       ↓
formField
       ↓
HTML
```

### Day 2 的實作目標

不要急著學所有 validation API。

今天只要做到：

```text
輸入資料
  ↓
更新 signal model
  ↓
FieldTree 知道欄位狀態
  ↓
validation schema 判斷是否合法
```

下一階段再進入 error display、field state、submit，以及 nested / array forms。

---

# 🅱️ Nx Monorepo 實戰：Nx 23.2 值得注意什麼？

今天 Nx 的部分很適合跟你的 Angular Monorepo 工作流連在一起。

Nx 23.2 官方文章特別提到幾個值得注意的更新：

- `@nx/oxlint` 與 Oxfmt 支援
- 更精簡的 task output
- 更好的 cache 行為
- agent sandbox 下的 Nx 執行改善
- Angular 22.1 支援
- Vitest generator 支援改善
- `.nx/ci-config.yaml` 簡化 Nx Cloud CI 設定

citeturn1search0

---

## 1. AI Agent 工作流最值得看的：精簡輸出

Nx 23.2 把成功且已 cache 的 task output 壓縮成單行，失敗 task 才顯示完整輸出。

這對人類開發者有幫助，對 AI coding agent 更重要：

```text
以前
↓
大量成功 log
↓
Agent 必須讀很多 token

現在
↓
成功 task → 精簡
失敗 task → 完整
↓
Agent 更快找到真正需要處理的錯誤
```

Nx 官方在 23.2 release article 中展示了大量 cache hit 時的輸出縮減效果，目的之一就是降低 CI 與 agent 處理不必要 log 的成本。citeturn1search0

---

## 2. Agent Sandbox 與 Nx

如果你使用 Cursor、Claude Code 或其他 coding agent，Nx 23.2 也值得注意。

官方提到 `nx configure-ai-agents` 可以協助設定 agent sandbox 需要的 Nx socket 權限，讓 Nx daemon、plugin workers 等工具在受限 sandbox 環境中更容易正常運作。citeturn1search0

可以把它理解成：

```text
AI Agent
   ↓
修改程式
   ↓
執行 Nx
   ↓
Project Graph
   ↓
Affected / Cache / Test / Build
```

Nx 不只是「建置工具」，而逐漸成為 AI agent 理解 Monorepo 的結構化入口。

---

## 3. Angular 22.1 + Nx 23.2

如果你的團隊準備從 Angular 21 往 Angular 22 移動，Nx 23.2 已支援 Angular 22.1；使用 `@nx/angular` 的 workspace 可以透過 Nx migration 協助升級。citeturn1search0

常見流程：

```bash
npx nx migrate latest
```

先產生 migration，再檢查 migration plan，最後：

```bash
npx nx migrate --run-migrations
```

不要在大型企業 Monorepo 裡直接把：

```text
Angular 升版
+ Nx 升版
+ Forms 重構
+ shared library 重構
```

放在同一個 PR。

比較安全的是拆成可驗證的小步驟。

---

# 🧱 建議的 Angular × Nx 架構

如果要在 Nx 中導入 Signal Forms，可以考慮：

```text
apps/
└── eip-web/

libs/
├── feature/
│   └── user/
├── data-access/
│   └── user/
├── ui/
│   └── form-field/
└── util/
    └── validation/
```

責任分工：

```text
feature
  ↓
負責頁面流程 / user interaction

ui
  ↓
共用視覺元件 / Form UI

data-access
  ↓
API / DTO / server communication

util
  ↓
純函式 / validation helper / 共用工具
```

Signal Form model 通常可以放在 feature 或 data-access 的適當邊界，不要因為「共用」就把所有表單塞進一個巨大 shared library。

---

# 🔄 Angular 21 ↔ 最新 Angular

目前官方支援資訊顯示：Angular 22 是 Active，Angular 21 是 LTS。Angular 22.0 於 2026-06-03 發布，Angular 官方目前文件網站已進入 22.1 系列；Angular 的 major release 支援週期現在通常為 24 個月。citeturn0search0turn0search6

版本相容性也要一起看。官方 compatibility table 顯示 Angular 22 與 Angular 21 對 Node.js、TypeScript、RxJS 有不同的版本要求，因此企業專案升級不能只改 `@angular/core`。citeturn0search2

建議升級順序：

```text
1. 確認 Angular / Nx 版本
        ↓
2. 確認 Node / TypeScript / RxJS
        ↓
3. 跑 migration
        ↓
4. lint / typecheck
        ↓
5. unit test
        ↓
6. build
        ↓
7. SSR / E2E
        ↓
8. production-like 驗收
```

Angular 官方也建議使用 `ng update` 與 Update Guide 來處理 major upgrade。citeturn0search12

---

# 🧪 今日實作：把 Nx 變成 AI Agent 的「導航系統」

今天可以在自己的 Nx workspace 做一個小實驗：

### Step 1：看 Project Graph

```bash
nx graph
```

### Step 2：看 affected

```bash
nx graph --affected
```

### Step 3：只跑受影響的測試

```bash
nx affected -t test
```

### Step 4：觀察 cache

```bash
nx affected -t test --skip-nx-cache
```

再重新執行：

```bash
nx affected -t test
```

觀察哪些 task 被 cache 命中。

### Step 5：觀察 agent-friendly output

如果使用 Nx 23.2，注意 CI / 非互動模式下成功 task 的輸出是否更精簡。

這個實驗的重點不是記住指令，而是建立：

```text
Project Graph
      ↓
Affected
      ↓
Task Graph
      ↓
Cache
      ↓
AI Agent / CI
```

這就是 Nx 在 AI-assisted development 時真正有價值的地方。

---

# 🎯 今日實作 Checklist

- [ ] 確認目前 Angular 版本
- [ ] 確認目前 Nx 版本
- [ ] 確認 Node / TypeScript / RxJS compatibility
- [ ] 建立一個最小 Signal Form
- [ ] 使用 `signal()` 建立 model
- [ ] 使用 `form()` 建立 FieldTree
- [ ] 使用 `[formField]` 綁定 HTML
- [ ] 使用 `required()` / `email()` 建立基本 validation
- [ ] 執行 `nx graph`
- [ ] 執行 `nx graph --affected`
- [ ] 執行 `nx affected -t test`
- [ ] 觀察 Nx cache hit
- [ ] 如果使用 Nx 23.2，觀察 agent-friendly output

---

# 📌 今日一句話

> **Angular 的 Signal Forms 正在從「新 API」變成穩定的 Signals-first 表單方案，而 Nx 正在從 Monorepo build tool 進一步變成 AI Agent 理解與操作大型程式碼庫的結構化入口。**

---

# 🔗 官方來源

### Angular

- Angular：https://angular.dev/
- Angular Releases：https://angular.dev/reference/releases
- Angular Version Compatibility：https://angular.dev/reference/versions
- Angular Signal Forms Overview：https://angular.dev/guide/forms/signals/overview
- Angular Roadmap：https://angular.dev/roadmap
- Angular Update Guide：https://angular.dev/update

### Nx

- Nx：https://nx.dev/
- Nx with Angular：https://nx.dev/docs/technologies/angular/introduction
- Nx 23.2 Release：https://nx.dev/blog/nx-23-2-release
- Nx 2026 Roadmap：https://nx.dev/blog/nx-2026-roadmap
