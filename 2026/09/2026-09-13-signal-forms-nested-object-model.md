# Signal Forms Nested Object × Type-safe Form Model｜2026-09-13

> 今日主軸：Signal Forms 從單層欄位進入 **Nested Object + Form Model 設計**。重點不是把表單寫得更複雜，而是讓 Angular 的資料模型、FieldTree、validation 與 Nx feature architecture 對齊。

## ⭐ 今日最值得看的 Angular 技術

今天最值得學的是 **Signal Forms 如何處理巢狀資料模型（Nested Object）**。

前幾天我們已經完成：

```text
Day 1
signal() → form() → formField

Day 2
schema → required() / email()

Day 3
FieldState → errors() / touched() / dirty() / pending()
         → FormRoot / submit()
```

今天往下一層：如果實際 API DTO 不是扁平資料，而是：

```ts
{
  name: 'Sherry',
  email: 'sherry@example.com',
  address: {
    city: 'New Taipei City',
    district: 'Yonghe',
    zipCode: '234'
  }
}
```

Signal Forms 可以直接依照這個資料模型建立巢狀 FieldTree。

這個能力很適合企業 Angular 專案，因為真正的 API model 通常不會永遠只有 `name`、`email` 這種一層欄位。

---

## 🔍 核心概念 / API

### 1. Form Model 應該先反映真正的資料結構

先定義 TypeScript model：

```ts
interface AddressFormModel {
  city: string;
  district: string;
  zipCode: string;
}

interface UserFormModel {
  name: string;
  email: string;
  address: AddressFormModel;
}
```

再建立 signal：

```ts
userModel = signal<UserFormModel>({
  name: '',
  email: '',
  address: {
    city: '',
    district: '',
    zipCode: '',
  },
});
```

最後交給 `form()`：

```ts
userForm = form(this.userModel);
```

概念上會得到：

```text
userForm
├── name
├── email
└── address
    ├── city
    ├── district
    └── zipCode
```

也就是說，**資料模型的巢狀結構會自然反映在 FieldTree 上**。

---

### 2. 巢狀欄位可以直接用 dot notation 存取

例如：

```html
<input [formField]="userForm.address.city" />
<input [formField]="userForm.address.district" />
<input [formField]="userForm.address.zipCode" />
```

在 TypeScript 裡也可以讀取對應 FieldState：

```ts
userForm.address.city().value();
userForm.address.city().valid();
userForm.address.city().errors();
```

這比自己建立：

```ts
cityError = signal<string | null>(null);
districtError = signal<string | null>(null);
zipCodeError = signal<string | null>(null);
```

更容易維持一致性。

---

### 3. Validation Schema 也跟著資料模型走

可以直接對 nested path 套用 validator：

```ts
userForm = form(this.userModel, (schema) => {
  required(schema.name, { message: '請輸入姓名' });
  required(schema.email, { message: '請輸入 Email' });
  email(schema.email, { message: 'Email 格式不正確' });

  required(schema.address.city, { message: '請輸入城市' });
  required(schema.address.district, { message: '請輸入行政區' });
  required(schema.address.zipCode, { message: '請輸入郵遞區號' });
});
```

這裡的關鍵是：

```text
TypeScript Model
      ↓
Nested Object
      ↓
FieldTree
      ↓
Schema Path
      ↓
Validation
      ↓
Field State
```

所以不要先想「我要做幾個 input」，而是先想「這個 feature 的資料模型是什麼」。

---

## 💻 可以直接實作的完整範例

下面是一個可以放進 Angular component 的使用方式。

```ts
import { Component, signal } from '@angular/core';
import {
  email,
  form,
  FormField,
  required,
} from '@angular/forms/signals';

interface AddressFormModel {
  city: string;
  district: string;
  zipCode: string;
}

interface UserFormModel {
  name: string;
  email: string;
  address: AddressFormModel;
}

@Component({
  selector: 'app-user-profile-form',
  imports: [FormField],
  template: `
    <form (submit)="$event.preventDefault()">
      <h2>基本資料</h2>

      <label>
        姓名
        <input [formField]="userForm.name" />
      </label>

      @if (userForm.name().touched() && userForm.name().invalid()) {
        @for (error of userForm.name().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        Email
        <input type="email" [formField]="userForm.email" />
      </label>

      @if (userForm.email().touched() && userForm.email().invalid()) {
        @for (error of userForm.email().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <h2>地址</h2>

      <label>
        城市
        <input [formField]="userForm.address.city" />
      </label>

      @if (
        userForm.address.city().touched() &&
        userForm.address.city().invalid()
      ) {
        @for (error of userForm.address.city().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        行政區
        <input [formField]="userForm.address.district" />
      </label>

      <label>
        郵遞區號
        <input [formField]="userForm.address.zipCode" />
      </label>

      <hr />

      <h3>目前 Model</h3>
      <pre>{{ userModel() | json }}</pre>
    </form>
  `,
})
export class UserProfileFormComponent {
  userModel = signal<UserFormModel>({
    name: '',
    email: '',
    address: {
      city: '',
      district: '',
      zipCode: '',
    },
  });

  userForm = form(this.userModel, (schema) => {
    required(schema.name, { message: '請輸入姓名' });

    required(schema.email, { message: '請輸入 Email' });
    email(schema.email, { message: 'Email 格式不正確' });

    required(schema.address.city, { message: '請輸入城市' });
    required(schema.address.district, { message: '請輸入行政區' });
    required(schema.address.zipCode, { message: '請輸入郵遞區號' });
  });
}
```

> 如果範例使用 `json` pipe，請依你的 Angular component 設定匯入 `JsonPipe`；這裡主要示範的是 nested Signal Form 的資料結構。

### 實作後你應該看到的關係

```text
userModel()
    │
    ├── name
    ├── email
    └── address
         ├── city
         ├── district
         └── zipCode
              │
              ▼
          userForm.address.zipCode
              │
              ▼
          FieldState
              ├── value()
              ├── valid()
              ├── invalid()
              ├── errors()
              ├── touched()
              └── dirty()
```

這就是今天最重要的觀念：**Model、FormTree 與 UI 是同一個資料結構的不同視角。**

---

# 🆕 Signal Forms 持續課程 — Day 4

延續 9/12 的課程，今天進入：

## Day 4：Nested Object + Form Model 設計

前面已經知道如何處理：

```text
單一 field
↓
FieldState
↓
整張 form
```

今天開始處理：

```text
單一 field
↓
Nested field
↓
Nested object
↓
完整 Form Model
```

### 今天一定要會

#### ① Model 先定義資料結構

```ts
interface Profile {
  name: string;
  address: {
    city: string;
    zipCode: string;
  };
}
```

#### ② `signal()` 保存表單資料

```ts
profile = signal<Profile>({
  name: '',
  address: {
    city: '',
    zipCode: '',
  },
});
```

#### ③ `form()` 建立 FieldTree

```ts
profileForm = form(this.profile);
```

#### ④ nested field 直接綁定

```html
<input [formField]="profileForm.address.city" />
```

#### ⑤ nested field 直接驗證

```ts
required(schema.address.city);
```

### 為什麼這很重要？

因為實務 API 常常長這樣：

```json
{
  "customer": {
    "name": "Sherry",
    "contact": {
      "email": "sherry@example.com"
    },
    "address": {
      "city": "New Taipei City",
      "district": "Yonghe"
    }
  }
}
```

如果表單模型一開始就跟 API domain model 對齊，後續 DTO mapping、validation 與 UI binding 都會比較清楚。

但也不要因此把「後端 DTO」直接等同「所有 UI Form Model」。如果 UI 需要額外的欄位、顯示狀態或暫存資料，建議仍然建立獨立的 Form Model，再在 data-access 層做 mapping。

---

# 🧩 Form Model ≠ API DTO

這是今天另一個非常實用的觀念。

假設後端 DTO：

```ts
interface UpdateUserDto {
  name: string;
  email: string;
  address: {
    city: string;
    district: string;
    zipCode: string;
  };
}
```

UI 可能需要：

```ts
interface UserFormModel extends UpdateUserDto {
  confirmEmail: string;
  isAddressExpanded: boolean;
}
```

這時不要硬把 UI-only 欄位送到 API。

推薦：

```text
UI
 ↓
UserFormModel
 ↓
Form / Validation
 ↓
Mapper
 ↓
UpdateUserDto
 ↓
API
```

例如：

```ts
function toUpdateUserDto(model: UserFormModel): UpdateUserDto {
  return {
    name: model.name,
    email: model.email,
    address: model.address,
  };
}
```

這種分層在 Nx Monorepo 裡尤其重要，因為它可以避免 feature UI model 反向污染 data-access 的 API contract。

---

# 🅱️ Nx Monorepo 實戰

Nested Signal Forms 放進 Nx Monorepo 後，可以把責任切成：

```text
apps/
└── web/

libs/
├── feature/
│   └── profile/
│       └── edit-profile/
├── data-access/
│   └── user/
│       ├── user-api.ts
│       └── user.models.ts
└── ui/
    └── form/
        └── field-error/
```

建議依賴方向：

```text
feature/profile
      ↓
data-access/user

feature/profile
      ↓
ui/form
```

而不要讓：

```text
ui/form
   ↓
feature/profile
```

形成反向依賴。

### 一個實用的 Feature 分工

```text
edit-profile feature
├── UserFormModel
├── Signal Form schema
├── Submit flow
└── UI interaction

user data-access
├── UpdateUserDto
├── API service
└── DTO mapping

shared form UI
└── Error message / loading UI
```

這樣當 API contract 改變時，不需要整個 Monorepo 的 UI library 跟著一起改。

---

# ⚡ Nx Affected：Nested Form 修改後怎麼驗證？

假設你只修改：

```text
libs/feature/profile/edit-profile
```

CI 不一定需要把整個 workspace 的所有 task 全部重新跑一次。

可以使用：

```bash
nx affected -t lint test build
```

Nx 會依 Git 變更與 Project Graph 找出受影響的 projects，再執行指定 targets。

也可以查看 affected graph：

```bash
nx graph --affected
```

這在 AI Agent 工作流中特別有價值：

```text
AI Agent 修改
     ↓
Nx Project Graph
     ↓
Affected Projects
     ↓
lint / test / build
     ↓
Remote Cache
```

Agent 不需要猜「改了這個 component 會影響誰」，Nx 可以用 workspace graph 做依賴分析。

---

# 🤖 AI Agent × Nested Form 的實務工作流

如果你使用 Cursor / Claude Code 類型的 coding agent，可以把任務拆成：

```text
1. Agent 先讀 Feature Project
2. 找到 Form Model
3. 找到 Data Access DTO
4. 找到 shared UI
5. 修改 Nested Form
6. 執行 nx affected -t lint test build
7. 根據失敗 task 修正
8. 再次驗證
```

這比直接對 agent 說：

> 「幫我把會員表單改好」

更容易得到可驗證的結果。

對 Monorepo 而言，**Project Graph + 明確 library boundary + affected validation** 本身就是 AI Agent 的重要上下文。

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-13**，Angular 官方版本資訊顯示：

| Angular | 狀態 | 支援資訊 |
|---|---|---|
| Angular 22 | Active | 2026-06-03 發布，Active 至 2027-06，LTS 至 2028-06 |
| Angular 21 | LTS | 2025-11-19 發布，LTS 至 2027-06 |

Angular 官方目前採每年 major release 的節奏；Angular 22 是目前 Active major，而 Angular 21 已進入 LTS。

今天的 Nested Signal Forms 在 Angular 21+ 的 Signal Forms 能力範圍內；同時官方目前的 Signal Forms API 文件把許多 API 標示為 Angular 22 起 stable，例如 `required()`、`email()`、`minLength()` 與 `FormRoot`。

因此如果你目前是 Angular 21：

```text
Angular 21 LTS
   ↓
可以持續學習 Signal Forms
   ↓
先在新 feature 建立邊界
   ↓
測試 nested model / validation / submit
   ↓
再規劃 Angular 22 migration
```

不要因為要使用一個新 API，就把整個 Angular 21 專案一次升級。企業專案更適合先確認 Angular、Nx、TypeScript、Node 與測試工具的相容性，再使用官方 migration。

Nx 官方也建議使用最新 Nx 搭配目前仍受支援的 Angular 版本；升級時應使用 Nx migration，而不是只手動修改 package version。

另外，Nx 23 已移除部分長期 deprecated API。例如 Module Federation 的 Angular import path 已從 `@nx/angular/module-federation` 遷移到 `@nx/module-federation/angular`。如果你的 workspace 有 Module Federation，升級 Nx 23 時要特別檢查 migration 結果。

---

# 🎯 今日實作 Checklist

- [ ] 建立一個有 nested object 的 `signal()` model
- [ ] 使用 `form()` 建立 nested FieldTree
- [ ] 使用 `formField` 綁定 `form.address.city`
- [ ] 對 nested field 使用 `required()`
- [ ] 從 nested FieldState 讀取 `errors()`
- [ ] 理解 Form Model 與 API DTO 不一定相同
- [ ] 建立 `toDto()` mapper
- [ ] 將 Feature、Data Access、UI 拆成 Nx libraries
- [ ] 用 `nx affected -t lint test build` 驗證變更
- [ ] 用 `nx graph --affected` 查看受影響範圍
- [ ] 檢查 Angular 21 / 22 與目前 Nx 的版本相容性

---

# 📌 今日一句話

> **好的 Signal Form 不是把 input 綁起來而已，而是先建立正確的資料模型，再讓 FieldTree、Validation、UI 與 API mapping 都圍繞這個模型工作。**

---

# 🔗 官方來源

### Angular

- Angular 版本與支援週期：https://angular.dev/reference/releases
- Angular 版本相容性：https://angular.dev/reference/versions
- Signal Forms Overview：https://angular.dev/guide/forms/signals/overview
- Signal Forms Validation：https://angular.dev/guide/forms/signals/validation
- Signal Forms Field State：https://angular.dev/guide/forms/signals/field-state-management
- Signal Forms Essentials：https://angular.dev/essentials/signal-forms
- `required()` API：https://angular.dev/api/forms/signals/required
- `email()` API：https://angular.dev/api/forms/signals/email
- `minLength()` API：https://angular.dev/api/forms/signals/minLength

### Nx

- Nx Changelog：https://nx.dev/changelog
- Nx × Angular Version Matrix：https://nx.dev/docs/kb/angular-nx-version-matrix
- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Project Graph：https://nx.dev/docs/features/explore-graph
- Nx Angular Migrations：https://nx.dev/docs/technologies/angular/migrations

---

## 📚 Signal Forms 課程目前進度

```text
Day 1  signal() + form() + FormField
   ↓
Day 2  Model + FieldTree + 基本 Validation
   ↓
Day 3  FieldState + Submit lifecycle
   ↓
Day 4  Nested Object + Form Model ← 今天
   ↓
Day 5  Array / Dynamic Form
   ↓
Day 6  API DTO ↔ Signal Form
   ↓
Day 7  Signal Forms + httpResource
   ↓
Day 8  Signal Forms + RxJS
   ↓
Day 9  Shared UI Components
   ↓
Day 10 Nx Feature / Data Access / UI Architecture
```
