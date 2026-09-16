# Signal Forms × httpResource：Reactive API 表單｜2026-09-16

> Signal Forms Day 7：在 Day 6 的「Form Model ≠ API DTO」基礎上，進一步把 API 讀取、提交與 reactive state 整合到 Angular Signals 架構。

## 1. ⭐ 今日最值得看的 Angular 技術

今天的主軸是 **Signal Forms + `httpResource` 的資料流設計**。

前一天我們把資料邊界拆成：

```text
Form Model
   ↓ mapper
Request DTO
   ↓ HttpClient / API
Backend
   ↓ mapper
Form Model
```

今天再往前一步：當 API 資料本身是 reactive state 時，可以用 Angular 的 Resource API 觀察 request 狀態，讓 UI 不必另外維護一堆 `loading`、`error`、`data` signals。

Angular 的 `httpResource` 建立在 Resource API 之上，會依 reactive request 變化執行 HTTP request，並提供 value / loading / error 等狀態。實務上適合「讀取型 API」；表單提交仍應明確使用 submission action 或 `HttpClient` mutation flow，而不要把 POST 當成自動 reactive resource。

---

## 2. 🔍 核心概念 / API

### `httpResource()`

概念範例：

```ts
userId = signal('42');

userResource = httpResource<User>(() => `/api/users/${this.userId()}`);
```

request 依賴 `userId()`。當 signal 改變時，resource 會重新取得資料。

常用狀態概念：

```ts
userResource.value()
userResource.isLoading()
userResource.error()
```

因此 template 可以直接反映 API lifecycle：

```html
@if (userResource.isLoading()) {
  <p>載入中...</p>
}

@if (userResource.error()) {
  <p>載入失敗</p>
}

@if (userResource.value(); as user) {
  <p>{{ user.name }}</p>
}
```

### Signal Forms 的角色

Resource 負責「外部資料」，Signal Form 負責「使用者編輯中的資料」。兩者不要混成同一個 state。

```text
API Response
    ↓
mapResponseToFormModel()
    ↓
Form Model signal
    ↓
form()
    ↓
User editing
    ↓
mapFormToRequestDto()
    ↓
Submit API
```

這個邊界非常重要：API response 可以重新取得，但不應在使用者正在編輯時，無條件覆蓋整個 Form Model。

---

## 3. 💻 可以直接實作的完整範例

以下建立「編輯會員資料」：先用 `httpResource` 讀取 API，再把 Response DTO mapping 成 Form Model。

### Step 1：DTO 與 Form Model 分離

```ts
export interface ProfileResponseDto {
  id: string;
  display_name: string;
  email: string;
}

export interface ProfileFormModel {
  displayName: string;
  email: string;
}

export interface UpdateProfileRequestDto {
  display_name: string;
  email: string;
}
```

### Step 2：Mapper

```ts
export function toProfileFormModel(
  dto: ProfileResponseDto,
): ProfileFormModel {
  return {
    displayName: dto.display_name,
    email: dto.email,
  };
}

export function toUpdateProfileRequest(
  model: ProfileFormModel,
): UpdateProfileRequestDto {
  return {
    display_name: model.displayName,
    email: model.email,
  };
}
```

### Step 3：Resource + Form

```ts
import { Component, effect, signal } from '@angular/core';
import {
  FormField,
  FormRoot,
  email,
  form,
  required,
} from '@angular/forms/signals';
import { httpResource } from '@angular/common/http';

@Component({
  selector: 'app-profile-edit',
  imports: [FormField, FormRoot],
  template: `
    @if (profileResource.isLoading()) {
      <p>載入中...</p>
    }

    @if (profileResource.error()) {
      <p>會員資料載入失敗</p>
    }

    <form [formRoot]="profileForm">
      <label>
        顯示名稱
        <input [formField]="profileForm.displayName" />
      </label>

      @if (
        profileForm.displayName().touched() &&
        profileForm.displayName().invalid()
      ) {
        @for (error of profileForm.displayName().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        Email
        <input type="email" [formField]="profileForm.email" />
      </label>

      <button
        type="submit"
        [disabled]="
          profileForm().invalid() ||
          profileForm().pending() ||
          profileForm().submitting()
        "
      >
        @if (profileForm().submitting()) {
          儲存中...
        } @else {
          儲存
        }
      </button>
    </form>
  `,
})
export class ProfileEditComponent {
  readonly profileId = signal('42');

  readonly profileResource = httpResource<ProfileResponseDto>(
    () => `/api/profiles/${this.profileId()}`,
  );

  readonly profileModel = signal<ProfileFormModel>({
    displayName: '',
    email: '',
  });

  readonly profileForm = form(this.profileModel, (schema) => {
    required(schema.displayName, {
      message: '顯示名稱為必填',
    });

    required(schema.email, {
      message: 'Email 為必填',
    });

    email(schema.email, {
      message: 'Email 格式不正確',
    });
  });

  constructor() {
    effect(() => {
      const dto = this.profileResource.value();

      if (dto && !this.profileForm().dirty()) {
        this.profileModel.set(toProfileFormModel(dto));
      }
    });
  }
}
```

> 實務上如果表單初始化、重新載入、取消編輯等流程更複雜，建議把「server data → form model」流程抽到 feature service / facade，而不是讓 component 承擔全部 orchestration。

---

## 4. 🚀 Submit：讀取 Resource 與寫入 API 要分開

一個常見錯誤是看到 `httpResource` 很方便，就想讓所有 GET / POST / PUT 都走同一種模式。

比較好的責任分工是：

```text
GET /profiles/:id
        ↓
   httpResource
        ↓
   server state

PUT /profiles/:id
        ↑
Form submit action
        ↑
   Form Model
        ↑
      UI
```

更新 API 可以由 data-access service 提供：

```ts
@Injectable({ providedIn: 'root' })
export class ProfileApi {
  private readonly http = inject(HttpClient);

  update(id: string, request: UpdateProfileRequestDto) {
    return this.http.put(`/api/profiles/${id}`, request);
  }
}
```

Form submission 負責：

1. 取得目前 Form Model。
2. `toUpdateProfileRequest()`。
3. 呼叫 API。
4. 將 server error 映射回表單需要的位置。
5. 成功後再決定是否重新取得 resource。

這樣「server state」與「draft state」不會互相污染。

---

## 5. 🧠 Signal Forms 課程進度：Day 7

前六天：

```text
Day 1  signal() + form()
   ↓
Day 2  Validation
   ↓
Day 3  Field State + Submit
   ↓
Day 4  Nested Object
   ↓
Day 5  Array / Dynamic Form
   ↓
Day 6  API DTO ↔ Form Model
```

今天：

```text
Day 7
Signal Forms
     +
httpResource
     ↓
Server State / Draft State 分離
```

今天一定要理解：

| 東西 | 責任 |
|---|---|
| `httpResource` | reactive API 讀取 |
| Form Model signal | 使用者正在編輯的 draft |
| `form()` | 建立 Signal Forms FieldTree |
| Mapper | API DTO ↔ Form Model |
| Submit action / API service | mutation / 寫入後端 |

下一階段 **Day 8：Signal Forms + RxJS**，會比較 `Observable`、Signals、Resource 三種 reactive data flow 的責任。

---

## 6. 🅱️ Nx Monorepo 實戰

建議把今天的程式拆成：

```text
libs/
├── feature/profile/
│   └── profile-edit/
│       └── profile-edit.component.ts
│
├── data-access/profile/
│   ├── profile-api.ts
│   ├── profile.models.ts
│   └── profile.mappers.ts
│
└── ui/form/
    └── field-error/
```

依賴方向：

```text
feature/profile
      ↓
data-access/profile
      ↓
HTTP API
```

共用 UI 則由 feature 使用：

```text
feature/profile ──→ ui/form
```

不要反過來讓 `data-access` 依賴 feature component，否則容易形成循環依賴。

### Nx boundary

如果 workspace 有設定 `enforce-module-boundaries`，可以把 `feature`、`data-access`、`ui` 分成不同 tag，再限制允許的 import 方向。

例如：

```text
feature → data-access  ✅
feature → ui           ✅
data-access → ui        ❌
ui → feature            ❌
```

這會讓 AI Agent 在大型 Monorepo 中更容易理解哪些 library 可以互相依賴。

---

## 7. 🤖 Nx 23.2 × AI Agent：今天值得注意的更新

Nx 23.2 在 2026-09-02 發布，帶來 Oxc toolchain 整合、Oxlint / Oxfmt 支援、較精簡的成功任務輸出，以及更好的 cache / sandbox 體驗。

對 AI coding workflow 特別有價值的是 **failure-only terminal output**：成功或 cache hit 的 task 可以縮成一行，失敗才輸出完整內容，可減少 agent 需要閱讀的終端輸出量。

可以把 CI / Agent workflow 設計成：

```bash
nx affected -t lint test build
```

先只驗證受影響的 project，再讓 agent 根據真正的 failure output 修正。

Nx 23.2 也加入 `@nx/oxlint` plugin；但 Angular template / HTML 相關 lint 規則仍可能需要 ESLint，因此不要把 ESLint 一次全部移除。

---

## 8. 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-16**，Angular 官方支援表列：

| 版本 | 狀態 | LTS 結束 |
|---|---|---|
| Angular 22 | Active | 2028-06 |
| Angular 21 | LTS | 2027-06 |
| Angular 20 | LTS | 2026-11-28 |

今天與 Signal Forms 最相關的差異是：`FormRoot` 等 Signal Forms API 在 Angular 22 API 文件中已標示為 stable。若目前專案是 Angular 21，不要因為文章範例就直接全面升級；應先確認 workspace、Nx、TypeScript、builder、測試工具與第三方套件的相容性。

Nx 官方版本矩陣也建議目前仍受支援的 Angular 版本搭配最新 Nx，以取得修正與新功能；升級時優先使用 `nx migrate`，不要直接手動大量改 Nx 套件版本。

---

## 9. 🎯 今日實作 Checklist

- [ ] 建立 `httpResource()` GET request
- [ ] 理解 `value()` / `isLoading()` / `error()`
- [ ] 分離 server state 與 Form Model draft state
- [ ] 建立 Response DTO
- [ ] 建立 Form Model
- [ ] 建立 Request DTO
- [ ] 寫 `toResponse → FormModel` mapper
- [ ] 寫 `FormModel → RequestDTO` mapper
- [ ] 不要用 `httpResource` 取代所有 mutation flow
- [ ] 將 API 放入 Nx `data-access`
- [ ] 將表單放入 Nx `feature`
- [ ] 檢查 `enforce-module-boundaries`
- [ ] 執行 `nx affected -t lint test build`

---

## 10. 📌 今日一句話

> **Resource 管 server state，Signal Form 管使用者的 draft state；兩者之間用明確的 mapper 與 feature boundary 連接，才是大型 Angular × Nx 專案容易維護的資料流。**

---

## 11. 🔗 官方來源

- Angular Versioning / Releases：https://angular.dev/reference/releases
- Angular `httpResource` API：https://angular.dev/api/common/http/httpResource
- Angular Resource API：https://angular.dev/guide/signals/resource
- Angular Signal Forms：https://angular.dev/guide/forms/signals/overview
- Angular Signal Forms Form Submission：https://angular.dev/guide/forms/signals/form-submission
- Angular `FormRoot` API：https://angular.dev/api/forms/signals/FormRoot
- Nx Changelog：https://nx.dev/changelog
- Nx 23.2 Release：https://nx.dev/blog/nx-23-2-release
- Nx Angular Version Matrix：https://nx.dev/docs/kb/angular-nx-version-matrix
- Nx Angular Migrations：https://nx.dev/docs/technologies/angular/migrations
