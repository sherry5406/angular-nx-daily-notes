# Signal Forms Field State × Submit 實戰｜2026-09-12

> 今日主軸：Signal Forms 從「表單綁定與驗證」進一步進入 Field State、錯誤顯示與 Submit lifecycle，並把它放進 Nx Monorepo 的 Feature / Data Access 架構。

## ⭐ 今日最值得看的 Angular 技術

今天的重點是 **Signal Forms Field State + `submit()` + `FormRoot`**。

Signal Forms 建立在 Angular Signals 之上，`form()` 會依照資料模型建立 FieldTree。每個 field 都可以進一步讀取自己的狀態，例如 `value()`、`valid()`、`invalid()`、`errors()`、`touched()`、`dirty()`、`pending()`。

這讓表單 UI 不需要另外維護一套 `isEmailError`、`isSubmitting`、`isDirty` 等零散 state。

Signal Forms 需要 Angular 21 或更高版本；目前 Angular 官方 API 文件中，`FieldState` 與 `submit()` 已標示為 Angular 22 起 stable。對 Angular 21 專案來說，可以先理解並評估導入範圍；升到 Angular 22 時則更適合正式採用。

---

## 🔍 核心概念 / API

### 1. FieldTree 與 FieldState

建立表單：

```ts
import { signal } from '@angular/core';
import { form } from '@angular/forms/signals';

userModel = signal({
  name: '',
  email: '',
});

userForm = form(this.userModel);
```

FieldTree 可以透過 dot notation 存取：

```ts
userForm.email
```

而呼叫 field 後：

```ts
userForm.email()
```

就能取得 FieldState。

例如：

```ts
userForm.email().value();
userForm.email().valid();
userForm.email().invalid();
userForm.email().errors();
userForm.email().touched();
userForm.email().dirty();
userForm.email().pending();
```

可以把它想成：

```text
userForm.email
      ↓
FieldTree node
      ↓
userForm.email()
      ↓
FieldState
      ├── value()
      ├── valid()
      ├── invalid()
      ├── errors()
      ├── touched()
      ├── dirty()
      └── pending()
```

### 2. `touched()` 與 `dirty()`

`touched()` 代表使用者曾經 focus / blur 互動過。

`dirty()` 則代表使用者修改過欄位值，即使最後又改回原本的值，dirty 狀態仍然可以維持 true。

常見的錯誤顯示策略是：

```text
invalid + touched
       ↓
顯示錯誤
```

而不是一進頁面就把所有錯誤全部顯示出來。

### 3. Form-level State

Root form 本身也是 FieldTree 的一部分，因此可以直接：

```ts
userForm().valid();
userForm().invalid();
userForm().pending();
userForm().touched();
userForm().dirty();
```

這非常適合控制整個表單的 UI：

```html
<button [disabled]="userForm().invalid() || userForm().pending()">
  儲存
</button>
```

Field-level state 適合處理單一欄位錯誤；form-level state 適合處理整張表單的狀態。

---

# 💻 可以直接實作的完整範例

下面是一個簡單的登入表單：

```ts
import { Component, signal } from '@angular/core';
import {
  email,
  form,
  FormField,
  FormRoot,
  required,
} from '@angular/forms/signals';

interface LoginModel {
  email: string;
  password: string;
}

@Component({
  selector: 'app-login',
  imports: [FormField, FormRoot],
  template: `
    <form [formRoot]="loginForm">
      <label>
        Email
        <input type="email" [formField]="loginForm.email" />
      </label>

      @if (loginForm.email().touched() && loginForm.email().invalid()) {
        @for (error of loginForm.email().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        Password
        <input type="password" [formField]="loginForm.password" />
      </label>

      @if (loginForm.password().touched() && loginForm.password().invalid()) {
        @for (error of loginForm.password().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <button
        type="submit"
        [disabled]="loginForm().invalid() || loginForm().pending() || loginForm().submitting()"
      >
        @if (loginForm().submitting()) {
          登入中...
        } @else {
          登入
        }
      </button>
    </form>
  `,
})
export class LoginComponent {
  loginModel = signal<LoginModel>({
    email: '',
    password: '',
  });

  loginForm = form(this.loginModel, (schema) => {
    required(schema.email, { message: 'Email 為必填欄位' });
    email(schema.email, { message: '請輸入正確的 Email 格式' });
    required(schema.password, { message: '密碼為必填欄位' });
  });
}
```

這個範例有三個重要觀念：

```text
Field State
   ↓
錯誤顯示

Form State
   ↓
控制 Submit button

FormRoot
   ↓
接管 HTML form submission
```

---

# 🚀 Signal Forms Submit Lifecycle

Signal Forms 的 `submit()` 會處理完整的 submission lifecycle：

```text
使用者 Submit
      ↓
標記 interactive fields 為 touched
      ↓
檢查 validation
      ↓
Invalid？ ── Yes → 停止 submission
      │
      No
      ↓
執行 action
      ↓
submitting() = true
      ↓
API request
      ↓
成功 / 回傳 submission errors
      ↓
submitting() = false
```

因此你不需要自己維護：

```ts
isSubmitting = signal(false);
```

如果使用 `FormRoot`，也不需要自己處理 `event.preventDefault()`。

## `FormRoot`

Template：

```html
<form [formRoot]="loginForm">
  ...
  <button type="submit">登入</button>
</form>
```

Component：

```ts
@Component({
  imports: [FormField, FormRoot],
})
```

`FormRoot` 會設定 `novalidate`、阻止瀏覽器預設 submit 行為，並觸發 Signal Forms 的 submit flow。

---

# 🌐 API Submission 實戰

如果 API 需要呼叫後端，可以在 `form()` 的 submission option 定義 action：

```ts
loginForm = form(
  this.loginModel,
  (schema) => {
    required(schema.email);
    email(schema.email);
    required(schema.password);
  },
  {
    submission: {
      action: async (field) => {
        const data = field().value();
        const result = await this.loginApi.login(data);

        if (!result.ok) {
          return {
            kind: 'serverError',
            message: '登入失敗，請稍後再試',
          };
        }
      },
    },
  },
);
```

其中 `field().value()` 就是目前提交的表單資料。

如果後端回傳的是特定欄位錯誤，也可以指定 `fieldTree`：

```ts
return {
  kind: 'emailTaken',
  message: '這個 Email 已經註冊',
  fieldTree: field.email,
};
```

這樣 server error 可以進入該欄位的 `errors()`，不用另外建立一套 server error state。

---

# 🧠 Signal Forms 課程進度

今天延續前面的課程，不重新開始。

### 前一階段

```text
Day 1
signal()
  ↓
form()
  ↓
FormField
```

```text
Day 2
required()
email()
validation schema
```

### 今天 Day 3

```text
FieldTree
   ↓
FieldState
   ↓
value()
valid()
invalid()
errors()
touched()
dirty()
pending()
   ↓
FormRoot
   ↓
submit()
   ↓
submitting()
   ↓
API
```

### 今天一定要會

| API | 用途 |
|---|---|
| `field()` | 取得 FieldState |
| `value()` | 取得欄位值 |
| `valid()` | 驗證是否通過 |
| `invalid()` | 是否有錯誤 |
| `errors()` | 取得錯誤集合 |
| `touched()` | 是否互動過 |
| `dirty()` | 是否修改過 |
| `pending()` | async validation 是否進行中 |
| `submitting()` | 是否正在提交 |
| `FormRoot` | 將 HTML form 接到 Signal Forms |
| `submit()` | 執行 submission lifecycle |

下一階段進入 **Day 4：Nested Object + Form Model 設計**，再逐步進入 array / dynamic form。

---

# 🅱️ Nx Monorepo 實戰

Signal Forms 進入企業專案後，不建議把所有表單與 API 邏輯塞在同一個 component。

可以採用：

```text
libs/
├── feature/
│   └── auth/
│       └── login/
├── data-access/
│   └── auth/
│       └── auth-api.ts
├── ui/
│   └── form/
│       └── field-error/
└── util/
    └── validation/
```

責任：

```text
feature/auth/login
      ↓
Form Model + Form Schema + Page Flow

ui/form/field-error
      ↓
共用錯誤顯示

data-access/auth
      ↓
API / DTO

util/validation
      ↓
跨 feature 的共用規則
```

### 不要把 FieldTree 當 Global State

表單通常屬於 feature-level state：

```text
Page
 ↓
Feature
 ↓
Signal Form
 ↓
Data Access
 ↓
API
```

而不是把整棵 FieldTree 放進 Global Store。這樣可以降低 feature 之間的耦合，也比較符合 Nx library boundary 的思路。

---

# 🤖 Nx × AI Agent 實戰

對 AI coding agent 而言，Nx 的 Project Graph 與 affected workflow 很有價值。

例如 Agent 修改：

```text
libs/feature/auth/login
```

完成後可以讓 CI 驗證：

```bash
nx affected -t lint test build
```

概念上：

```text
AI Agent
   ↓
修改 Feature
   ↓
Nx Project Graph
   ↓
Affected Projects
   ↓
lint / test / build
   ↓
Cache
```

這比每次都重新執行整個 Monorepo 更適合大型 workspace。

Nx 版本升級則建議使用官方 migration 流程，例如 `nx migrate`，不要直接手動大量修改 Nx 套件版本。

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 2026-09-12，Angular 22 是目前 Active major，而 Angular 21 處於 LTS。

| 版本 | 狀態 | 建議 |
|---|---|---|
| Angular 22 | Active | 新功能與新專案優先 |
| Angular 21 | LTS | 企業既有專案穩定維護 |

Signal Forms 官方文件目前要求 Angular 21+；`FieldState` 與 `submit()` API 在 Angular 22 起標示為 stable。

如果專案目前是 Angular 21，可以採取：

```text
Angular 21
   ↓
先理解 Signal Forms
   ↓
選擇新 feature 試用
   ↓
確認 shared component / validation 架構
   ↓
再規劃 Angular 22 升級
```

不要把 Angular 升版、Reactive Forms 全面重寫、Shared Component 重構、Nx 架構重構全部塞進同一個 migration。

---

# 🎯 今日實作 Checklist

- [ ] 建立一個最小 Signal Form
- [ ] 練習 `field().value()`
- [ ] 練習 `valid()` / `invalid()`
- [ ] 練習 `errors()` 顯示 validation message
- [ ] 分清楚 `touched()` 與 `dirty()`
- [ ] 使用 `form().invalid()` 控制 Submit button
- [ ] 使用 `FormRoot`
- [ ] 理解 `submit()` lifecycle
- [ ] 理解 `submitting()`
- [ ] 把 API 放在 Nx `data-access`
- [ ] 用 `nx affected -t lint test build` 驗證受影響的 projects

---

# 📌 今日一句話

> **Signal Forms 真正有價值的地方，不只是把 input 綁到 signal，而是讓「值、驗證、互動狀態、提交狀態、Server Error」都進入同一套 reactive Field State。**

---

# 🔗 官方來源

- Angular Signal Forms Overview：https://angular.dev/guide/forms/signals/overview
- Angular Field State Management：https://angular.dev/guide/forms/signals/field-state-management
- Angular Form Submission：https://angular.dev/guide/forms/signals/form-submission
- Angular Signal Forms Validation：https://angular.dev/guide/forms/signals/validation
- Angular Signal Forms Models：https://angular.dev/guide/forms/signals/models
- Angular `FieldState` API：https://angular.dev/api/forms/signals/FieldState
- Angular `submit()` API：https://angular.dev/api/forms/signals/submit
- Angular Signal Forms Tutorial：https://angular.dev/tutorials/signal-forms
- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Migrate：https://nx.dev/docs/features/major-version-upgrade
