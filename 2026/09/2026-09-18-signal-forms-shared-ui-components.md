# Signal Forms × Shared UI Components：建立可重用的 Field Error 與 Form Control｜2026-09-18

> 今日主軸：Signal Forms Day 9，從前面的單一 Feature 表單進入 Shared UI Components，學會讓表單狀態留在 Feature、讓 UI 元件只負責呈現與互動，避免把 FieldTree 綁死在共用元件裡。

---

## ⭐ 今日最值得看的 Angular 技術

今天最值得學的是 **Signal Forms 與可重用表單 UI 的 boundary 設計**。

前幾天已經完成：

```text
Day 1  signal() + form()
Day 2  validation
Day 3  Field State + Submit
Day 4  Nested Object
Day 5  Array / Dynamic Form
Day 6  API DTO ↔ Form Model
Day 7  httpResource
Day 8  RxJS interop
```

今天進入 Day 9：

```text
Feature Form
    ↓
FieldTree / FieldState
    ↓
Shared UI
    ↓
Input / Error / Hint / Loading
```

核心不是「把 Signal Forms 做成一個萬用表單元件」，而是建立清楚的責任邊界：

- **Feature**：擁有 Form Model、Form Schema、Submit 與商業規則。
- **Shared UI**：負責 label、input、hint、error、disabled 等視覺與互動。
- **Data Access**：負責 API、DTO 與 server error mapping。

這種切法特別適合 Nx Monorepo，因為可以把依賴方向固定下來，讓 AI coding agent 也比較容易理解 workspace 結構。

---

# 🔍 核心概念 / API

## 1. FieldTree 是 Feature State，不是 UI Component State

假設 Feature 有：

```ts
profileForm = form(profileModel, (schema) => {
  required(schema.name);
});
```

Template：

```html
<input [formField]="profileForm.name" />
```

這裡的：

```ts
profileForm.name
```

是 Feature 擁有的 FieldTree node。

不要讓 Shared UI Component 自己重新建立一份 Form Model：

```text
❌ Feature Form
     ↓
Shared Component 自己再建立 form()
```

而應該：

```text
✅ Feature
   ↓
FieldTree node
   ↓
Shared UI
```

Shared UI 只使用傳入的 field binding。

---

## 2. `FormField`：讓 UI 與 FieldTree 連接

Signal Forms 的 field directive 可以讓原生 input 與 FieldTree node 同步：

```html
<input [formField]="profileForm.name" />
```

這代表：

```text
input value
   ↕
FieldTree
   ↕
Signal Form model
```

因此 Shared UI 可以專注在 HTML / accessibility，而 Feature 保留 form schema 與資料模型。

---

## 3. Error UI 應該讀 Field State

一個共用錯誤元件真正需要的是「目前 field 的 state」，例如：

```text
invalid()
touched()
errors()
```

概念上：

```html
@if (field().touched() && field().invalid()) {
  @for (error of field().errors(); track error.kind) {
    <p>{{ error.message }}</p>
  }
}
```

不要在每個 Feature 都重新寫：

```ts
emailErrorMessage
passwordErrorMessage
nameErrorMessage
```

這會造成 validation state duplicated。

---

# 💻 可以直接實作的完整範例

以下示範一個簡單的共用 `FieldErrorComponent`。

## Shared UI

```ts
import { Component, input } from '@angular/core';

type FieldError = {
  kind: string;
  message?: string;
};

@Component({
  selector: 'app-field-error',
  standalone: true,
  template: `
    @if (showError()) {
      <div class="field-error" role="alert">
        @for (error of errors(); track error.kind) {
          <p>{{ error.message || error.kind }}</p>
        }
      </div>
    }
  `,
})
export class FieldErrorComponent {
  readonly invalid = input.required<boolean>();
  readonly touched = input.required<boolean>();
  readonly errors = input.required<readonly FieldError[]>();

  protected readonly showError = () =>
    this.touched() && this.invalid() && this.errors().length > 0;
}
```

Feature 使用：

```html
<app-field-error
  [invalid]="profileForm.email().invalid()"
  [touched]="profileForm.email().touched()"
  [errors]="profileForm.email().errors()"
/>
```

這裡故意沒有把整棵 FieldTree 傳進 UI component。

Shared UI 只收到它真正需要的資訊：

```text
invalid
 touched
 errors
```

好處是 UI library 不需要知道 Feature 的 Form Model 長什麼樣子。

---

# 🧩 更進一步：共用 Input Component

如果要做真正的 reusable input，可以把「輸入框 UI」與「FieldTree binding」保持清楚。

Feature：

```html
<app-text-field
  label="Email"
  [field]="profileForm.email"
/>
```

但在設計這種元件時，必須先確認目前 Angular 版本的 Signal Forms directive 與 component boundary 能否直接承接你的型別。大型 Nx workspace 建議先做一個最小 proof-of-concept，再決定是否抽成 shared abstraction。

不要為了追求「所有表單欄位都只有一個 component」而建立過度泛型的 API。

---

# 🧠 Signal Forms 課程 Day 9

今天正式進入 Shared UI：

```text
Day 8
Signal Forms
    ↕
RxJS interop

Day 9
Signal Forms
    ↓
FieldTree
    ↓
Shared UI Components
```

目前課程進度：

| Day | 主題 |
|---|---|
| 1 | `signal()` + `form()` 基礎 |
| 2 | FormField / Validation |
| 3 | Field State / Submit lifecycle |
| 4 | Nested Object |
| 5 | Array / Dynamic Form |
| 6 | API DTO ↔ Form Model |
| 7 | Signal Forms + `httpResource` |
| 8 | Signal Forms + RxJS |
| **9** | **Shared UI Components ← 今天** |

下一步可以進入 **Day 10：Nx Feature / Data Access / UI Architecture**，把今天的 Shared UI boundary 放進完整 Monorepo。

---

# 🧪 測試策略

Shared UI 最重要的是「狀態進來後，UI 是否正確呈現」。

例如：

```text
invalid=false
   ↓
不顯示 error
```

```text
invalid=true
+touched=true
   ↓
顯示 error
```

```text
errors=[...]
   ↓
依序顯示 message
```

因此測試可以集中在：

1. error 顯示條件
2. accessibility，例如 `role="alert"`
3. 多個 validation error
4. 沒有 error 時不產生多餘 DOM

而真正的 validation schema 應在 Feature / domain logic 層另外測試。

---

# 🅱️ Nx Monorepo 實戰

推薦結構：

```text
libs/
├── feature/
│   └── profile/
│       └── edit-profile/
│
├── ui/
│   └── form/
│       ├── field-error/
│       └── text-field/
│
├── data-access/
│   └── profile/
│       └── profile-api.ts
│
└── util/
    └── form/
```

依賴方向：

```text
feature
  ↓
ui

feature
  ↓
data-access

feature
  ↓
util
```

避免：

```text
ui
  ↓
feature
```

否則 Shared UI 反而依賴 Feature，最後很容易形成 circular dependency。

### `nx graph`

當你懷疑 boundary 有問題時，可以使用：

```bash
pnpm nx graph
```

確認 Project Graph。

CI 則可以搭配：

```bash
pnpm nx affected -t lint test build
```

只驗證這次變更真正影響的 projects。

---

# 🤖 Nx × AI Agent 實戰

今天的 architecture 特別適合 AI Agent。

例如給 Agent 一個明確規則：

```text
libs/feature/*
  ↓ 可以依賴
libs/ui/*
libs/data-access/*
libs/util/*

libs/ui/*
  ↓ 不可以依賴
libs/feature/*
```

Agent 修改 `field-error` 後，可以讓 Nx 幫忙找到 affected projects：

```text
AI Agent
   ↓
修改 libs/ui/form/field-error
   ↓
Nx Project Graph
   ↓
Affected Projects
   ↓
lint / test / build
```

Nx 23 也持續強化 AI agent 與 monorepo workflow，例如 agentic migrations、task sandboxing，以及 `nx configure-ai-agents`。citeturn0search14turn0search10

對團隊來說，重點不是「讓 AI 隨便改整個 repo」，而是：

```text
Architecture Boundary
        ↓
AI 可理解的 scope
        ↓
Affected validation
        ↓
Automated Gate
```

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-18**，Angular 官方支援狀態為：

| 版本 | 狀態 | 發布日期 | LTS 結束 |
|---|---|---|---|
| Angular 22 | Active | 2026-06-03 | 2028-06 |
| Angular 21 | LTS | 2025-11-19 | 2027-06 |
| Angular 20 | LTS | 2025-05-28 | 2026-11-28 |

Angular 官方也已調整 release cadence：從 Angular 22 起，major release 改為一年一次。citeturn0search9

如果你的企業專案目前在 Angular 21：

```text
Angular 21 LTS
   ↓
持續開發 Feature
   ↓
Shared UI 邊界穩定
   ↓
再評估 Angular 22
```

不要為了今天的 Shared UI 重構就同時進行 major upgrade。

另外，Angular 22 的 Node.js / TypeScript 相容版本與 Angular 21 不同，升級前要先檢查官方 compatibility matrix。citeturn0search12turn0search16

---

# 📦 Nx 23.2 今日注意事項

截至 2026-09-18，Nx 官方支援資訊顯示：

- Nx 23：Current
- Nx 22：LTS
- Nx 21：LTS

Nx 23.2 發布於 2026-09-02，包含 `@nx/oxlint`、Oxfmt、TUI 改善，以及 Angular Rspack、Bun 等多項修正。citeturn0search10turn0search11

升級 Nx 時，維持 `nx` 與 `@nx/*` 版本一致，並優先使用：

```bash
pnpm nx migrate latest
```

Nx 官方也建議使用 `nx migrate` 管理 major upgrade。citeturn0search11

---

# 🎯 今日實作 Checklist

- [ ] 理解 FieldTree 屬於 Feature state
- [ ] 不在 Shared UI 重建 `form()`
- [ ] 使用 `[formField]` 連接 FieldTree
- [ ] 建立共用 `FieldErrorComponent`
- [ ] 使用 `invalid()` / `touched()` / `errors()` 控制錯誤顯示
- [ ] 保留 accessibility，例如 `role="alert"`
- [ ] Feature 與 UI library 保持單向依賴
- [ ] 使用 `nx graph` 檢查 Project Graph
- [ ] 使用 `nx affected -t lint test build`
- [ ] 確認 Angular 21 / 22 migration 時機
- [ ] 確認 Nx 版本與 `@nx/*` 一致

---

# 📌 今日一句話

> **好的 Shared UI 不是把表單邏輯藏起來，而是讓 Feature 保有狀態與規則，UI 元件只負責把狀態可靠地呈現出來。**

---

# 🔗 官方來源

- Angular Signal Forms：https://angular.dev/guide/forms/signals/overview
- Angular Signal Forms Models：https://angular.dev/guide/forms/signals/models
- Angular Signal Forms Field State：https://angular.dev/guide/forms/signals/field-state-management
- Angular Signal Forms Validation：https://angular.dev/guide/forms/signals/validation
- Angular RxJS Interop：https://angular.dev/ecosystem/rxjs-interop
- Angular Releases：https://angular.dev/reference/releases
- Angular Version Compatibility：https://angular.dev/reference/versions
- Nx Changelog：https://nx.dev/changelog
- Nx Release Schedule：https://nx.dev/docs/reference/releases
- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Project Graph：https://nx.dev/features/manage-application-dependencies
- Nx 23 Release：https://nx.dev/blog/nx-23-release
