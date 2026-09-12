# Signal Forms Field State × Submit 實戰｜2026-09-12

> 今日主軸：Signal Forms Day 3，從 Validation 進一步進入 Field State、錯誤顯示與正式 Submit 流程，讓表單從「能綁定」進化到「能正確驗證與送 API」。

---

## ⭐ 今日最值得看的 Angular 技術

今天最值得學的是 **Signal Forms 的 Field State + `submit()` / `FormRoot`**。

Signal Forms 已經是 Angular 的穩定能力。Angular 官方目前把 Signal Forms 描述為以 Signals 管理表單狀態，並提供與 Reactive Forms 的互通，適合新表單直接採用，也適合既有大型表單逐步遷移。

今天的學習重點不是再增加更多 validator，而是把昨天的：

```text
signal()
   ↓
form()
   ↓
required() / email()
```

接成真正的使用者流程：

```text
使用者輸入
   ↓
Field State
   ↓
valid / invalid / touched / dirty / errors
   ↓
顯示錯誤
   ↓
submit()
   ↓
API
```

Angular 官方指出，FieldTree 的每個節點都提供 reactive state signals，包括 `valid()`、`invalid()`、`errors()`、`pending()`、`touched()`、`dirty()` 等。citeturn1search3turn1search4

---

# 🔍 核心概念 / API

## 1. `field()`：取得 Field State

昨天我們使用：

```ts
userForm.email
```

今天開始要理解：

```ts
userForm.email()
```

呼叫 FieldTree node 後，可以取得該欄位的 FieldState。

例如：

```ts
userForm.email().value();
userForm.email().valid();
userForm.email().invalid();
userForm.email().errors();
userForm.email().touched();
userForm.email().dirty();
```

可以把它理解成：

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

---

## 2. `errors()`：不要自己猜錯誤

Signal Forms 的 validation error 會透過：

```ts
field().errors()
```

取得。

例如：

```ts
@if (userForm.email().touched() && userForm.email().invalid()) {
  @for (error of userForm.email().errors(); track error.kind) {
    <p>{{ error.message }}</p>
  }
}
```

這比：

```ts
if (!email) {
  error = 'Email 必填';
}
```

更適合大型專案，因為 validation schema 與 UI 顯示可以保持分離。

Angular 官方也特別說明，不應依賴瀏覽器原生 `validity`、`:valid`、`:invalid` 或 `validationMessage` 來觀察 Signal Forms 狀態；應使用 Field State signals。citeturn1search0turn1search3

---

## 3. `touched()` 與 `dirty()` 不一樣

這兩個非常容易搞混。

### `touched()`

代表使用者有互動，例如 focus 後離開欄位。

```ts
userForm.email().touched()
```

### `dirty()`

代表使用者修改過值。

```ts
userForm.email().dirty()
```

所以常見的錯誤顯示策略是：

```text
invalid
  +
touched
  ↓
顯示錯誤
```

而不是一進頁面就把所有錯誤全部顯示出來。

---

# 💻 可以直接實作的完整範例

下面是一個可以直接拿來理解 Signal Forms 的登入表單：

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
        <input
          type="email"
          [formField]="loginForm.email"
        />
      </label>

      @if (loginForm.email().touched() && loginForm.email().invalid()) {
        @for (error of loginForm.email().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        Password
        <input
          type="password"
          [formField]="loginForm.password"
        />
      </label>

      @if (loginForm.password().touched() && loginForm.password().invalid()) {
        @for (error of loginForm.password().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <button
        type="submit"
        [disabled]="loginForm().invalid() || loginForm().pending()"
      >
        登入
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
    required(schema.email, {
      message: 'Email 為必填欄位',
    });

    email(schema.email, {
      message: '請輸入正確的 Email 格式',
    });

    required(schema.password, {
      message: '密碼為必填欄位',
    });
  });
}
```

這裡先把「錯誤顯示」做好，再進入 submit。

---

# 🚀 Signal Forms 的 Submit 流程

Angular 官方提供 `submit()` 來處理完整 submission lifecycle。

流程可以理解成：

```text
submit
  ↓
標記 interactive fields 為 touched
  ↓
檢查 validation
  ↓
如果 invalid → 停止
  ↓
執行 action
  ↓
submitting() = true
  ↓
API 完成
  ↓
成功 / 回傳 submission errors
```

官方文件指出，`submit()` 會先處理互動欄位的 touched 狀態，再檢查 validation；只有驗證通過才執行 action。action 執行期間可以用 `submitting()` 判斷是否正在送出。citeturn1search2

---

## `FormRoot`：最推薦的表單入口

常見寫法：

```ts
imports: [FormField, FormRoot]
```

Template：

```html
<form [formRoot]="loginForm">
  ...
  <button type="submit">登入</button>
</form>
```

`FormRoot` 會幫忙：

1. 設定 `novalidate`
2. 阻止 browser 預設 submit 行為
3. 觸發 Signal Forms 的 submit flow

所以不要再自己寫一大串：

```ts
(event) => {
  event.preventDefault();
  ...
}
```

讓 Signal Forms 管理 submission lifecycle 會更乾淨。citeturn1search2

---

# 🧩 `submitting()`：避免重複送出

假設 API 需要 2 秒：

```text
使用者按登入
      ↓
API request
      ↓
2 秒等待
```

如果沒有處理 submitting，使用者可能連按 5 次。

Signal Forms 可以直接使用：

```ts
loginForm().submitting()
```

例如：

```html
<button
  type="submit"
  [disabled]="loginForm().submitting()"
>
  @if (loginForm().submitting()) {
    登入中...
  } @else {
    登入
  }
</button>
```

官方 API 也明確說明，同一個 FieldTree 正在 submission 時，不允許併發 submission；後續 submit 會直接回傳 `false`。citeturn1search5

---

# 🌐 API Submission 實戰

接下來把 API 接進來：

```ts
loginForm = form(
  this.loginModel,
  (schema) => {
    required(schema.email, {
      message: 'Email 為必填欄位',
    });

    email(schema.email, {
      message: 'Email 格式錯誤',
    });

    required(schema.password, {
      message: '密碼為必填欄位',
    });
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

重點是：

```ts
field().value()
```

拿到的是目前表單資料。

而 API 錯誤也可以透過 submission error 回到 FieldTree，而不是自己另外建立一套 error state。Angular 官方的 submission API 支援把 submission errors 整合回指定 field。citeturn1search2turn1search5

---

# 🧠 Signal Forms Day 3

昨天 Day 2：

```text
signal()
   ↓
form()
   ↓
required()
   ↓
email()
```

今天 Day 3：

```text
FieldTree
   ↓
FieldState
   ↓
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

### 今天一定要會的 API

| API | 用途 |
|---|---|
| `field()` | 取得 FieldState |
| `value()` | 取得目前值 |
| `valid()` | 是否通過驗證 |
| `invalid()` | 是否有驗證錯誤 |
| `errors()` | 取得錯誤集合 |
| `touched()` | 是否與使用者互動 |
| `dirty()` | 是否修改過 |
| `pending()` | async validation 是否進行中 |
| `submitting()` | 是否正在送出 |
| `submit()` | 執行 submission flow |
| `FormRoot` | 將 HTML form 接上 Signal Forms submission |

---

# 🅱️ Nx Monorepo 實戰

Signal Forms 進入實務後，不建議所有表單邏輯都塞在 Angular component。

可以先建立：

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

責任可以分成：

```text
feature/auth/login
      ↓
表單 model + form schema + page flow

ui/form/field-error
      ↓
共用錯誤顯示 UI

data-access/auth
      ↓
API request / DTO

util/validation
      ↓
真正跨 feature 共用的 validation logic
```

### 一個重要原則

不要因為 Signal Forms 很方便，就把整棵 `FieldTree` 放進 global state。

比較好的方式是：

```text
Page / Feature
   ↓
Signal Form
   ↓
Data Access
   ↓
API
```

表單本身通常屬於 feature-level state，而不是整個 application 的 global state。

---

# 🤖 Nx × AI Agent 實戰

目前 Nx 23 是 current release，而 Nx 官方的 release policy 顯示 Nx 23 於 2026-06-16 發布；Nx 22、21 目前處於 LTS。Nx 版本建議使用 `nx migrate` 維持在支援版本。citeturn0search0

Nx 23 release 也持續強化 agentic migration、task sandboxing、效能與 target configuration，這讓 Nx 越來越適合成為 AI coding agent 操作 Monorepo 的結構化入口。citeturn0search2turn0search4

今天可以用這個方式理解：

```text
AI Agent
   ↓
修改 feature/auth/login
   ↓
nx affected
   ↓
只找真正受影響的 project
   ↓
test / lint / build
   ↓
Cache
   ↓
快速得到驗證結果
```

對大型 Angular Monorepo，這比讓 Agent 每次執行整個 workspace 更有效率。

---

# 🔄 Angular 21 ↔ 最新 Angular

目前 Angular 官方支援表顯示：

| 版本 | 狀態 | 發布 |
|---|---|---|
| Angular 22 | Active | 2026-06-03 |
| Angular 21 | LTS | 2025-11-19 |
| Angular 20 | LTS | 2025-05-28 |

Angular 22 是目前 Active major，Angular 21 已進入 LTS。Angular 官方目前的 major release 支援週期通常為 24 個月。citeturn0search1

對目前仍在 Angular 21 的企業專案，我會建議：

```text
Angular 21 LTS
     ↓
先導入穩定的 Signal Forms
     ↓
建立新的表單開發模式
     ↓
確認 Nx / Node / TypeScript 相容性
     ↓
再安排 Angular 22 migration
```

不要因為 Angular 22 是 Active，就一次把所有 Angular 21 表單全部重寫。

Angular 官方也已把 Signal Forms 列為 stable，並強調與 Reactive Forms 的互通，可以採取 progressive migration。citeturn1search1

---

# 🧪 今日實作

今天建議直接完成一個 Login Form：

### Step 1：建立 model

```ts
loginModel = signal({
  email: '',
  password: '',
});
```

### Step 2：建立 schema

```ts
loginForm = form(this.loginModel, (schema) => {
  required(schema.email);
  email(schema.email);
  required(schema.password);
});
```

### Step 3：顯示 Field State

```html
@if (loginForm.email().touched() && loginForm.email().invalid()) {
  @for (error of loginForm.email().errors(); track error.kind) {
    <p>{{ error.message }}</p>
  }
}
```

### Step 4：接 FormRoot

```html
<form [formRoot]="loginForm">
```

### Step 5：處理 submission

```ts
{
  submission: {
    action: async (field) => {
      const data = field().value();
      await this.api.login(data);
    },
  },
}
```

完成後你應該能回答：

> 「Signal Forms 的 validation 結果到底放在哪裡？」

答案是：

```text
FieldState
  ↓
errors()
valid()
invalid()
pending()
```

---

# 🎯 今日實作 Checklist

- [ ] 理解 `form.email` 與 `form.email()` 的差異
- [ ] 會讀取 `value()`
- [ ] 會讀取 `valid()` / `invalid()`
- [ ] 會讀取 `errors()`
- [ ] 理解 `touched()` 與 `dirty()` 的差異
- [ ] 理解 `pending()`
- [ ] 使用 `FormRoot`
- [ ] 使用 `submit()` / submission action
- [ ] 使用 `submitting()` 防止重複送出
- [ ] 把 API 呼叫放在 data-access
- [ ] 在 Nx 中保持 Feature / UI / Data Access 邊界

---

# 📌 今日一句話

> **Signal Forms 的價值不只是在「用 signal 做表單」，而是讓 value、validation、interaction state、submission lifecycle 全部成為同一套 reactive FieldTree 模型。**

---

# 🔗 官方來源

- Angular Signal Forms：https://angular.dev/essentials/signal-forms
- Signal Forms Validation：https://angular.dev/guide/forms/signals/validation
- Signal Forms Field State：https://angular.dev/guide/forms/signals/field-state-management
- Signal Forms Form Submission：https://angular.dev/guide/forms/signals/form-submission
- Signal Forms Cross-field Logic：https://angular.dev/guide/forms/signals/cross-field-logic
- Signal Forms Testing：https://angular.dev/guide/forms/signals/testing
- Angular Roadmap：https://angular.dev/roadmap
- Angular Releases：https://angular.dev/reference/releases
- Nx Releases：https://nx.dev/docs/reference/releases
- Nx Changelog：https://nx.dev/changelog
- Nx 23 Release：https://nx.dev/blog/nx-23-release
- Nx / Angular Version Matrix：https://nx.dev/docs/kb/angular-nx-version-matrix
