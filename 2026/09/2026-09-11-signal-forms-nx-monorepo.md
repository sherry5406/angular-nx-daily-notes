# Signal Forms × Nx Monorepo 實戰｜2026-09-11

> 今日主軸：Angular 最新版本 × Angular 21 LTS × Signal Forms × Nx Monorepo

## ⭐ 今日最值得看的 Angular 技術

目前 Angular 22 是 Active 版本；Angular 21 已進入 LTS，支援到 2027 年 6 月。對現有 Angular 21 專案而言，今天最值得投入的不是為了升版而升版，而是先掌握 Signal Forms、Signals 與目前 Angular 的 model-driven 開發方式，再評估升級節奏。

Angular 官方目前把 Signal Forms 定位成建立在 Signals 上的表單狀態管理方案。它透過 writable signal 作為表單模型，再由 `form()` 建立對應的 FieldTree，HTML 使用 `[formField]` 綁定，因此 UI 與資料模型可以自動同步。Signal Forms 需要 Angular v21 或更高版本。

### 為什麼值得現在學？

如果 Angular 專案正在逐步採用 Signals，Signal Forms 的思路會比單純把 Reactive Forms API 換掉更重要：

```text
Form UI
  ↓
FieldTree
  ↓
Signal Form Model
  ↓
API DTO / Domain Model
```

核心觀念是：**表單資料模型成為單一來源，而表單狀態由 FieldTree 衍生出來。**

> 注意：既有 Reactive Forms 專案不需要為了 Signal Forms 強制重寫；Signal Forms 特別適合新的、以 Signals 為核心的應用。

---

## 🔍 核心概念 / API

### 1. `signal()`：表單資料模型

```ts
interface LoginData {
  email: string;
  password: string;
  rememberMe: boolean;
}

loginModel = signal<LoginData>({
  email: '',
  password: '',
  rememberMe: false,
});
```

這個 signal 就是表單資料的來源。

### 2. `form()`：建立 FieldTree

```ts
loginForm = form(this.loginModel);
```

`form()` 建立一棵 FieldTree：

```text
loginForm
├── email
├── password
└── rememberMe
```

因此可以直接使用：

```ts
loginForm.email
loginForm.password
```

### 3. `[formField]`：連接 HTML

```html
<input type="email" [formField]="loginForm.email" />
<input type="password" [formField]="loginForm.password" />
```

使用者輸入會同步更新 model；model 更新也會反映到 UI。

---

# 💻 可以直接實作的範例

```ts
import { Component, signal } from '@angular/core';
import { FormField, form } from '@angular/forms/signals';

interface LoginData {
  email: string;
  password: string;
}

@Component({
  selector: 'app-login',
  imports: [FormField],
  template: `
    <input type="email" placeholder="Email" [formField]="loginForm.email" />
    <input type="password" placeholder="Password" [formField]="loginForm.password" />
    <pre>{{ loginModel() | json }}</pre>
  `,
})
export class LoginComponent {
  loginModel = signal<LoginData>({
    email: '',
    password: '',
  });

  loginForm = form(this.loginModel);
}
```

實務上可以先把它放在獨立 feature 裡，不要一次改掉整個既有 Reactive Forms 架構。

---

# 🆕 Signal Forms 教學 — Day 1

## Day 1：建立第一個 Signal Form

今天只學三件事情：

```text
1. signal()  → 保存表單資料
2. form()    → 建立 FieldTree
3. formField → 綁定 HTML
```

### 練習

```ts
interface UserData {
  name: string;
  email: string;
  phone: string;
}

userModel = signal<UserData>({
  name: '',
  email: '',
  phone: '',
});

userForm = form(this.userModel);
```

HTML：

```html
<input [formField]="userForm.name" />
<input [formField]="userForm.email" />
<input [formField]="userForm.phone" />
```

### 今天要理解的重點

不要先背 API，先理解這個關係：

```text
userModel = signal(...)
        ↓
userForm = form(userModel)
        ↓
userForm.name / email / phone
        ↓
[formField]
        ↓
HTML
```

---

# 🅱️ Nx Monorepo 實戰

你的團隊是 Nx Monorepo，因此 Signal Forms 不建議直接散落在各個 Angular App 裡。

可以先採用這種結構：

```text
apps/
  eip-web/

libs/
  feature/
    user/
  ui/
    form-field/
  data-access/
    user/
```

例如：

```text
libs/feature/user
        ↓
使用 Signal Forms
        ↓
libs/ui/form-field
        ↓
共用 UI 元件
        ↓
libs/data-access/user
        ↓
API / DTO
```

### Nx Affected

當 Monorepo 中只有某個 feature 被修改時，可以利用 Nx Affected 只執行受影響的專案，而不是每次把整個 Monorepo 全部測試。

```bash
nx graph --affected
nx affected -t test
nx affected -t build
```

Nx 的價值不只是把專案放在同一個 Repository，而是讓工具知道專案之間的依賴關係。

---

# 🔄 Angular 21 ↔ 最新 Angular

| 項目 | Angular 21 | Angular 22 |
|---|---|---|
| 支援狀態 | LTS | Active |
| 發布 | 2025-11-19 | 2026-06-03 |
| LTS 結束 | 2027-06 | 2028-06 |
| Signal Forms | 可使用 | 可使用 |
| 適合策略 | 穩定維護 | 新功能優先 |

對企業 Angular 專案，我會建議：

```text
Angular 21
   ↓
先採用適合的 Signals / Signal Forms
   ↓
整理 Nx boundaries
   ↓
改善測試與 affected CI
   ↓
再安排 Angular 22 升級
```

而不是一次同時升版、重寫 Forms、重構 shared components。

---

# 🎯 今日實作 Checklist

- [ ] 建立一個最小 Signal Form
- [ ] 理解 `signal()` 是 form model
- [ ] 理解 `form()` 建立 FieldTree
- [ ] 使用 `[formField]` 綁定 HTML
- [ ] 不要急著改既有 Reactive Forms
- [ ] 在 Nx 中確認 Feature / UI / Data Access 的責任
- [ ] 執行 `nx graph --affected`
- [ ] 思考目前專案哪些表單適合逐步導入 Signal Forms

---

# 📌 今日一句話

> **Signal Forms 的核心不是「新的表單 API」，而是讓表單資料模型直接建立在 Angular Signals 上，再由 FieldTree 管理欄位狀態。**

---

# 🔗 官方來源

- Angular 版本與支援週期：https://angular.dev/reference/releases
- Angular Signal Forms：https://angular.dev/essentials/signal-forms
- Signal Forms Overview：https://angular.dev/guide/forms/signals/overview
- Signal Forms Model：https://angular.dev/guide/forms/signals/models
- Signal Forms Validation：https://angular.dev/guide/forms/signals/validation
- Signal Forms Field State：https://angular.dev/guide/forms/signals/field-state-management
- Signal Forms Submission：https://angular.dev/guide/forms/signals/form-submission
- Angular Signal Forms Tutorial：https://angular.dev/tutorials/signal-forms
- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Project Graph：https://nx.dev/docs/features/explore-graph
