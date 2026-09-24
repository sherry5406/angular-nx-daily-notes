# Signal Forms 測試策略 × Nx Feature Boundary｜2026-09-24

> 今日主軸：Signal Forms Day 10。從「會使用表單」進一步走到「怎麼測、怎麼拆、怎麼讓 AI Agent 不容易改壞」。同時把 Angular 最新的 Resource / `httpResource` 思維放進企業前端架構。

## ⭐ 今日最值得看的 Angular 技術

今天最值得看的不是單一表單 API，而是 **Signal-based state + async Resource + 可測試的 Feature Boundary**。

Angular 官方目前把 `resource` 視為 stable API（v22 起），Resource 會用 signals 表達非同步資料的 `value`、`status`、`error`、`isLoading` 等狀態；`httpResource` 則是在 `HttpClient` 上提供 reactive wrapper。citeturn1search0turn1search2turn1search15

這個方向很適合搭配我們前面 9 天的 Signal Forms 學習：

```text
使用者輸入
   ↓
Signal Form
   ↓
Form Model
   ↓
Feature
   ↓
Data Access
   ↓
httpResource / HttpClient
   ↓
API
```

重點不是「全部改成 resource」，而是先分清楚：

- **Form State**：使用者正在編輯什麼
- **Server State**：後端目前回傳什麼
- **UI State**：畫面是否 loading、錯誤、disabled
- **Domain / DTO**：前後端交換的資料格式

只要這四種責任沒有混在一起，Angular 專案就比較容易維護，也比較容易測試。

---

## 🔍 核心概念 / API

### 1. Resource 是 Server State，不是 Form State

Angular `Resource` 會把非同步資料的狀態包成 signals，例如：

```ts
resource.value()
resource.status()
resource.error()
resource.isLoading()
```

官方目前的 Resource status 包含：

```text
idle
loading
reloading
resolved
error
local
```

其中 `reloading` 很值得注意：重新載入時，原本的 value 可以繼續存在，不一定要整個畫面清空。citeturn1search7turn1search4

所以畫面可以做到：

```text
第一次載入
  → Skeleton

已有資料，再次查詢
  → 保留舊資料 + 顯示 Loading

API 失敗
  → 保留合理的 UI 狀態 + 顯示錯誤
```

---

### 2. `httpResource()` 適合讀取，不要拿來取代所有 API

Angular 官方明確把 `resource` 定位為非同步 dependency，而 `resource` 本身特別適合 read operation；mutation 不應該因為方便就全部塞進 Resource。citeturn1search2turn1search10

簡單理解：

```text
GET /users/123
       ↓
httpResource / resource
       ↓
Server State
```

而：

```text
POST /users
PUT /users/123
DELETE /users/123
       ↓
HttpClient / Data Access service
       ↓
Mutation
```

這個界線可以讓程式比較容易理解。

---

### 3. `FormRoot` 把 FieldTree 接到 HTML form

Signal Forms 的 `FormRoot` 是 stable since Angular 22，它會把 `FieldTree` 綁定到 `<form>`，並處理 submit event。citeturn1search17

```html
<form [formRoot]="userForm">
  ...
</form>
```

這也代表我們前面學到的：

```text
form()
 ↓
FieldTree
 ↓
FormField
 ↓
FormRoot
```

已經可以組成完整表單架構。

---

# 💻 可以直接實作的完整範例

今天做一個「使用者編輯頁」。

需求：

1. API 先取得使用者
2. 顯示姓名與 Email
3. 使用者可以修改 Form
4. 表單驗證失敗不能送出
5. 儲存後呼叫 API
6. Resource 負責 Server State
7. Signal Form 負責 Form State

---

## Step 1：Model

```ts
export interface UserDto {
  id: string;
  name: string;
  email: string;
}

export interface UserFormModel {
  name: string;
  email: string;
}

export interface UpdateUserRequest {
  name: string;
  email: string;
}
```

注意：

```text
UserDto
   ≠
UserFormModel
   ≠
UpdateUserRequest
```

不要因為欄位剛好一樣，就讓所有地方共用同一個 interface。

---

## Step 2：Data Access

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class UserApi {
  private readonly http = inject(HttpClient);

  getUser(id: string) {
    return this.http.get<UserDto>(`/api/users/${id}`);
  }

  updateUser(id: string, request: UpdateUserRequest) {
    return this.http.put<UserDto>(`/api/users/${id}`, request);
  }
}
```

這個 service 的責任只有 API。

不要把：

```text
validation
form()
UI loading message
navigation
```

全部塞進 Data Access。

---

## Step 3：Feature Component

```ts
import {
  Component,
  effect,
  inject,
  signal,
} from '@angular/core';
import {
  email,
  form,
  FormField,
  FormRoot,
  required,
} from '@angular/forms/signals';

@Component({
  selector: 'app-user-edit',
  imports: [FormField, FormRoot],
  template: `
    @if (userApiLoading()) {
      <p>載入中...</p>
    }

    <form [formRoot]="userForm">
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

      <button
        type="submit"
        [disabled]="userForm().invalid() || userForm().submitting()"
      >
        @if (userForm().submitting()) {
          儲存中...
        } @else {
          儲存
        }
      </button>
    </form>
  `,
})
export class UserEditComponent {
  private readonly api = inject(UserApi);

  readonly userId = signal('123');
  readonly userApiLoading = signal(false);

  readonly model = signal<UserFormModel>({
    name: '',
    email: '',
  });

  readonly userForm = form(this.model, (schema) => {
    required(schema.name, {
      message: '姓名為必填欄位',
    });

    required(schema.email, {
      message: 'Email 為必填欄位',
    });

    email(schema.email, {
      message: 'Email 格式不正確',
    });
  });

  loadUser() {
    this.userApiLoading.set(true);

    this.api.getUser(this.userId()).subscribe({
      next: (user) => {
        this.model.set({
          name: user.name,
          email: user.email,
        });
      },
      error: () => {
        this.userApiLoading.set(false);
      },
      complete: () => {
        this.userApiLoading.set(false);
      },
    });
  }
}
```

這裡故意把「表單」與「API」分開：

```text
UserApi
  ↓
只處理 HTTP

UserEditComponent
  ↓
表單、畫面、使用者互動
```

大型專案再進一步把 API 讀取改成 `httpResource()`，可以讓 Server State 的 loading / error / value 更一致。

---

# 🧪 Day 10：Signal Forms 測試策略

前 9 天我們學的是「怎麼做」。

今天開始學「怎麼確定它沒有壞」。

建議測試分成三層：

```text
        E2E
         ↑
   Feature Integration
         ↑
   Form / Domain Unit
```

不要所有東西都靠 E2E。

---

## ① Validation Unit Test

先測規則，而不是測瀏覽器。

例如：

```ts
it('email 不符合格式時應該 invalid', () => {
  // 建立 form model
  // 套用 email validation
  // 更新 email
  // 驗證 invalid state
});
```

測試重點：

```text
輸入資料
   ↓
Validation
   ↓
Expected state
```

---

## ② Component Test

Component test 測：

```text
Input
 ↓
FieldTree
 ↓
Validation message
 ↓
Button disabled
```

例如：

```ts
it('Email 錯誤時，儲存按鈕應該 disabled', () => {
  // render component
  // input invalid email
  // expect button disabled
});
```

這比直接測 private method 更有價值。

---

## ③ E2E Test

E2E 不需要測每一個 validation rule。

只需要確認完整流程：

```text
進入頁面
 ↓
取得 User
 ↓
修改 Email
 ↓
儲存
 ↓
API 成功
 ↓
畫面更新
```

這樣測試責任比較清楚。

---

# 🧠 Signal Forms 課程進度

目前課程進度：

```text
Day 1  signal() + form()
Day 2  Validation
Day 3  Field State + Submit
Day 4  Nested Object
Day 5  Array / Dynamic Form
Day 6  API DTO ↔ Form Model
Day 7  httpResource
Day 8  RxJS Interop
Day 9  Shared UI Components
Day 10 Testing + Architecture ← 今天
```

今天要建立一個很重要的觀念：

> **Signal Forms 不是單純的 UI 元件，而是 Feature State 的一部分。**

因此測試也應該圍繞 Feature boundary 設計。

---

# 🅱️ Nx Monorepo 實戰

推薦架構：

```text
apps/
└── web/

libs/
├── feature/
│   └── user-edit/
├── data-access/
│   └── user/
├── ui/
│   └── form-error/
└── util/
    └── validation/
```

依賴方向：

```text
feature/user-edit
      ↓
 data-access/user
      ↓
     API

feature/user-edit
      ↓
      ui
```

不要讓：

```text
ui
 ↓
data-access
```

形成奇怪的反向依賴。

---

## Nx Boundary 的價值

如果 AI Agent 修改：

```text
libs/feature/user-edit
```

我們希望它知道：

```text
可以依賴
  ↓
UI
Data Access
Util

不應該直接依賴
  ↓
另一個不相關 Feature 的內部 implementation
```

這就是 Nx Module Boundary 對 AI coding 很有價值的原因。

---

## `nx affected`

PR 驗證可以使用：

```bash
nx affected -t lint test build
```

概念是：

```text
Git diff
  ↓
Nx Project Graph
  ↓
Affected Projects
  ↓
lint
  ↓
test
  ↓
build
```

大型 Monorepo 不需要每次把全部專案重新驗證。

---

# 🤖 Nx × AI Agent

Nx 23 已經非常明確地朝 AI-agent friendly workflow 發展。

Nx 23.2 在 2026-09-02 發布，加入 Oxlint / Oxfmt、成功 task 更精簡的 terminal output、TUI status bar，以及讓 Nx 在 agent sandbox 中執行並共享 cache 的能力。官方表示成功 task 的輸出大幅縮減，可降低 AI agent token 使用量。citeturn0search5turn0search6

因此今天可以把流程想成：

```text
AI Agent
   ↓
修改 Feature
   ↓
Nx Project Graph
   ↓
Affected
   ↓
lint / test / build
   ↓
Cache
   ↓
PASS / FAIL
```

這比單純叫 AI「幫我改 Angular」更可靠。

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-24**，Angular 官方支援版本如下：

| 版本 | 狀態 | LTS 結束 |
|---|---|---|
| Angular 22 | Active | 2028-06 |
| Angular 21 | LTS | 2027-06 |
| Angular 20 | LTS | 2026-11-28 |

Angular 22.2 的時間點落在 2026 年 9 月；Angular 官方也已調整版本節奏，從 Angular 22 起改為一年一個 major，搭配多個 minor releases。citeturn1search13

如果你的專案目前是 Angular 21，不需要因為 Angular 22 Active 就立刻全面升級。

建議：

```text
Angular 21
   ↓
維持 LTS
   ↓
先吸收 Angular 22 API
   ↓
確認 Nx 相容版本
   ↓
安排 migration
   ↓
一次升級 + 一次驗證
```

Nx 官方目前的版本矩陣顯示，Angular 21.2 可搭配 Nx 22.6+，Angular 22.1 則建議使用 Nx 最新版本，且至少需要 Nx 23.2。citeturn0search13

所以不要只升：

```bash
pnpm update @angular/core
```

而應該一起檢查：

```text
Angular
Angular CLI
@nx/angular
nx
Node
TypeScript
```

並保持 Nx 與 `@nx/*` 套件版本一致。Nx 官方也建議使用 `nx migrate` 管理升級。citeturn0search7turn0search14

---

# 🎯 今日實作 Checklist

- [ ] 分清楚 Form State 與 Server State
- [ ] 理解 `Resource` 的 status lifecycle
- [ ] 理解 `httpResource()` 的用途
- [ ] 知道 mutation 不應該硬塞進 Resource
- [ ] 使用 `FormRoot`
- [ ] 為 Signal Form validation 寫測試
- [ ] 為 Field UI 寫 component test
- [ ] 為完整使用者流程寫 E2E
- [ ] 把 Feature / Data Access / UI 分開
- [ ] 使用 Nx Module Boundary
- [ ] 使用 `nx affected -t lint test build`
- [ ] 檢查 Angular / Nx / Node / TypeScript 版本相容性
- [ ] 了解 Angular 21 LTS 與 Angular 22 Active 的差異

---

# 📌 今日一句話

> **好的 Angular 架構不是把所有新 API 都用上，而是讓 Form State、Server State、UI State 與 API Boundary 各自負責自己的事情。**

---

# 🔗 官方來源

- Angular Versioning & Releases：https://angular.dev/reference/releases
- Angular Resource API：https://angular.dev/api/core/Resource
- Angular `resource()` API：https://angular.dev/api/core/resource
- Angular Async Reactivity：https://angular.dev/guide/signals/resource
- Angular `httpResource()`：https://angular.dev/guide/http/http-resource
- Angular Signal Forms `FormRoot`：https://angular.dev/api/forms/signals/FormRoot
- Angular RxJS Interop：https://angular.dev/ecosystem/rxjs-interop
- Nx Changelog：https://nx.dev/changelog
- Nx 23.2 Release：https://nx.dev/blog/nx-23-2-release
- Nx Angular / Nx Version Matrix：https://nx.dev/docs/kb/angular-nx-version-matrix
- Nx Release & Support Policy：https://nx.dev/docs/reference/releases
