# Signal Forms API DTO × Domain Model｜2026-09-15

> Signal Forms Day 6：從 UI Form Model 走到 API DTO，建立可維護的資料邊界。

## 1. ⭐ 今日最值得看的 Angular 技術

今天的主軸不是再學一個 validator，而是把 Signal Forms 真正接進前端的 API 架構：**Form Model 與 API DTO 分離**。

Signal Forms 的 `form()` 以 `WritableSignal<TModel>` 作為 source of truth，FieldTree 的結構會跟著 model 型別走；這讓表單本身非常適合當 UI model，但不代表它就應該直接等同於後端 DTO。Angular 官方也明確指出，`form()` 建立的 FieldTree 會直接反映並更新傳入的 model。citeturn0search15turn0search13

實務上建議把資料拆成三層：

```text
UI Form Model
      ↓ mapFormToCreateDto()
API Request DTO
      ↓ HTTP
Backend
      ↓ mapResponseToFormModel()
UI Form Model
```

這樣可以避免：

- 後端欄位命名綁死在 HTML。
- UI 暫存欄位直接送進 API。
- API 回傳格式改變時，整個表單元件一起壞掉。
- `Date`、number、optional field 等型別轉換散落在 component。

---

## 2. 🔍 核心概念 / API

### `form()`：FieldTree 與 model 綁定

```ts
const profileModel = signal({
  name: '',
  email: '',
});

const profileForm = form(profileModel);
```

`form()` 的 model 是表單資料來源；修改 FieldTree 對應的 value，也會同步修改原本的 signal model。citeturn0search15

### `apply()`：重用巢狀 schema

如果 DTO 對應的 UI model 有巢狀物件，可以把驗證規則抽成 reusable schema，再用 `apply()` 套用。Angular 官方目前把 `apply()` 標示為 v22 stable。citeturn0search16

```ts
const addressSchema = schema<AddressModel>((address) => {
  required(address.city);
  required(address.zipCode);
});

const profileForm = form(profileModel, (profile) => {
  apply(profile.address, addressSchema);
});
```

### `submit()`：在 API boundary 執行 action

Signal Forms 的 `submit()` 會先處理 touched 與 validation，再執行 action；action 執行期間可以透過 submission state 判斷正在送出。若 action 回傳錯誤，也可以把錯誤導回對應 field。citeturn0search20

因此建議把 API mapping 放在 submit action 前後，而不是把 `HttpClient` 呼叫與欄位轉換全部塞在 template/component event handler 裡。

---

## 3. 💻 可以直接實作的完整範例

下面是一個「會員資料編輯」範例。重點是 **Form Model ≠ API DTO**。

### Step 1：定義 API DTO

```ts
export interface UpdateProfileRequest {
  display_name: string;
  email: string;
  city_code: string;
}

export interface ProfileResponse {
  id: string;
  display_name: string;
  email: string;
  city_code: string;
}
```

後端使用 snake_case 並不需要污染 Angular template。

### Step 2：定義 UI Form Model

```ts
export interface ProfileFormModel {
  displayName: string;
  email: string;
  city: string;
}
```

### Step 3：建立 mapping function

```ts
export function toUpdateProfileDto(
  model: ProfileFormModel,
): UpdateProfileRequest {
  return {
    display_name: model.displayName.trim(),
    email: model.email.trim(),
    city_code: model.city,
  };
}

export function toProfileFormModel(
  response: ProfileResponse,
): ProfileFormModel {
  return {
    displayName: response.display_name,
    email: response.email,
    city: response.city_code,
  };
}
```

這個 mapping layer 的價值在於：API contract 改動時，只需要修改 mapper，而不是到處搜尋 `display_name`。

### Step 4：建立 API service

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ProfileApi {
  private readonly http = inject(HttpClient);

  updateProfile(
    id: string,
    body: UpdateProfileRequest,
  ): Observable<ProfileResponse> {
    return this.http.put<ProfileResponse>(
      `/api/profiles/${id}`,
      body,
    );
  }
}
```

### Step 5：Signal Form component

```ts
import { Component, inject, signal } from '@angular/core';
import {
  email,
  form,
  FormField,
  required,
  submit,
} from '@angular/forms/signals';

@Component({
  selector: 'app-profile-form',
  imports: [FormField],
  template: `
    <form>
      <label>
        顯示名稱
        <input [formField]="profileForm.displayName" />
      </label>

      <label>
        Email
        <input type="email" [formField]="profileForm.email" />
      </label>

      <label>
        城市代碼
        <input [formField]="profileForm.city" />
      </label>

      <button
        type="button"
        [disabled]="profileForm().invalid || profileForm().submitting"
        (click)="save()"
      >
        @if (profileForm().submitting) {
          儲存中...
        } @else {
          儲存
        }
      </button>
    </form>
  `,
})
export class ProfileFormComponent {
  private readonly api = inject(ProfileApi);

  readonly profileId = 'user-001';

  readonly profileModel = signal<ProfileFormModel>({
    displayName: '',
    email: '',
    city: '',
  });

  readonly profileForm = form(this.profileModel, (path) => {
    required(path.displayName);
    required(path.email);
    email(path.email);
    required(path.city);
  });

  async save(): Promise<void> {
    const success = await submit(this.profileForm, async (field) => {
      const request = toUpdateProfileDto(field().value());

      try {
        await this.api
          .updateProfile(this.profileId, request)
          .toPromise();

        return;
      } catch {
        return {
          kind: 'serverError',
          message: '儲存失敗，請稍後再試',
        };
      }
    });

    if (success) {
      console.log('Profile updated');
    }
  }
}
```

> 實際專案若已全面採用 `firstValueFrom()`，可將 Observable 轉成 Promise；也可以把 API layer 改成 Signal-first 的 resource/data-access 架構。這裡的重點是 mapper 與 submit boundary，而不是特定 RxJS 寫法。

### Step 6：從 API response 初始化 Form Model

```ts
this.api.getProfile(this.profileId).subscribe((response) => {
  this.profileModel.set(toProfileFormModel(response));
});
```

這裡有一個非常重要的觀念：**不要把 API response 直接塞進 UI form，只要透過 mapper 建立 UI model。**

---

## 4. 🆕 Signal Forms 持續課程：Day 6 — API DTO ↔ Signal Form

前面課程已經完成：

1. `signal()` + `form()` 基礎
2. `formField` / Form Model / FieldTree
3. Field State / Submit lifecycle
4. Nested Object + Form Model
5. Array / Dynamic Form

### Day 6 的核心

今天開始建立真正適合團隊專案的 boundary：

```text
Backend DTO
    ↓ mapper
Form Model
    ↓ form()
FieldTree
    ↓ FormField
Angular Template
```

### 為什麼不要 DTO 直接當 Form Model？

例如後端：

```ts
interface UserDto {
  first_name: string;
  last_name: string;
  birth_date: string;
}
```

UI 可能需要：

```ts
interface UserFormModel {
  firstName: string;
  lastName: string;
  birthDate: Date | null;
  confirmEmail: string;
}
```

`confirmEmail` 根本不是 API 欄位；`birth_date` 也可能需要從 ISO string 轉成 UI 使用的 `Date`。

因此 Form Model 是 **UI 行為模型**，DTO 是 **API contract**。

### 今日練習

把你目前專案任一個表單找出來，回答三個問題：

1. 哪些欄位只存在 UI？
2. 哪些欄位名稱與 API 不同？
3. 哪些欄位需要型別轉換？

如果答案不是「沒有」，就值得建立 mapper。

---

## 5. 🅱️ Nx Monorepo 實戰

在 Nx Monorepo 裡，建議把 API DTO、mapper、UI form 分開管理，而不是全部放在 feature component。

推薦結構：

```text
apps/
└── eip-web/

libs/
├── profile/
│   ├── feature-profile-form/
│   ├── data-access-profile/
│   │   ├── profile.api.ts
│   │   ├── profile.dto.ts
│   │   └── profile.mapper.ts
│   └── ui-profile-form/
└── shared/
    └── ui/
```

### 建議依賴方向

```text
feature
   ↓
data-access
   ↓
API DTO / mapper
```

UI component 則盡量只知道 Form Model：

```text
feature-profile-form
        ↓
ProfileFormModel
        ↓
ui-profile-form
```

這會讓 API contract 的變動不需要直接影響 shared UI。

### CI：只測受影響的 project

Nx 的 affected workflow 可以根據 PR 的變更與 project graph，只執行受到影響的 tasks。citeturn1view2

例如：

```bash
nx affected -t lint test build
```

如果今天只改 `profile.mapper.ts`，沒有必要重新執行整個 Monorepo 的所有 application test。

### Nx 23.2 值得注意的新能力

Nx 官方 changelog 顯示 Nx 23.2 於 2026-09-02 發布，包含：

- `@nx/oxlint` plugin
- `nx format` 可偵測 Prettier / Oxfmt
- 成功與 cached task 的 terminal output 更精簡，失敗才展開完整輸出
- TUI status bar 與搜尋
- Nx cache 改進，能跨 worktree / clone / sandbox 共用
- Angular 22.1 與更廣泛的 Vitest 支援

對 AI coding workflow 特別有價值的是「成功 task 少輸出、失敗才輸出」，因為可以減少 agent 需要處理的 terminal token。citeturn1view1

---

## 6. 🔄 Angular 21 ↔ 最新 Angular：升級注意事項

截至 **2026-09-15**，Angular 官方支援表顯示：

| 版本 | 狀態 | LTS 結束 |
|---|---|---|
| Angular 22 | Active | 2028-06 |
| Angular 21 | LTS | 2027-06 |
| Angular 20 | LTS | 2026-11-28 |

Angular 22.1 已於 2026-07-27 那週進入 release cycle；官方目前文件版本為 22.1.6。citeturn1view0

### 對 Angular 21 專案的實際建議

如果你的專案目前是 Angular 21：

**不需要為了追版本而立刻升級。** Angular 21 仍是 LTS。

但如果準備採用今天的 Signal Forms API，值得注意：Angular 官方目前將 `form()`、`apply()`、`applyEach()`、`FormRoot` 等 API 標示為 **stable since v22.0**；Signal Forms 本身則要求 Angular v21+。citeturn0search13turn0search15turn0search16turn0search11turn0search19

因此團隊可以採取：

```text
Angular 21
├─ 可以開始學習 / 試用 Signal Forms
└─ 既有 Reactive Forms 不需要全部重寫

Angular 22
├─ Signal Forms stable API 更完整
├─ 適合新表單採用
└─ 升級前先跑 migration + CI
```

Angular 官方建議 major upgrade 使用 `ng update`；跨多個 major 時應逐版升級，不要一次跳過多個 major。citeturn1view0

---

## 7. 🎯 今日實作 Checklist

- [ ] 找出專案中一個 API 表單
- [ ] 建立獨立的 Form Model
- [ ] 建立 Request DTO
- [ ] 建立 Response DTO
- [ ] 建立 `toDto()` mapper
- [ ] 建立 `toFormModel()` mapper
- [ ] 使用 `form()` 建立 Signal Form
- [ ] 將 validation 集中在 schema
- [ ] 在 `submit()` boundary 呼叫 data-access
- [ ] 不讓 UI component 直接依賴 snake_case DTO
- [ ] Nx 中把 mapper / DTO 放進 data-access library
- [ ] CI 使用 `nx affected -t lint test build`

---

## 8. 📌 今日一句話

> **Signal Forms 解決的是表單狀態；Mapper 解決的是資料邊界。兩者一起使用，才會讓大型 Angular Monorepo 的表單真正可維護。**

---

## 9. 🔗 官方來源

- Angular Signal Forms Overview：<https://angular.dev/guide/forms/signals/overview>
- Angular Signal Forms Models：<https://angular.dev/guide/forms/signals/models>
- Angular Signal Forms Validation：<https://angular.dev/guide/forms/signals/validation>
- Angular Signal Forms Form Submission：<https://angular.dev/guide/forms/signals/form-submission>
- Angular `form()` API：<https://angular.dev/api/forms/signals/form>
- Angular `apply()` API：<https://angular.dev/api/forms/signals/apply>
- Angular `applyEach()` API：<https://angular.dev/api/forms/signals/applyEach>
- Angular Versioning / Releases：<https://angular.dev/reference/releases>
- Nx Changelog：<https://nx.dev/changelog>
- Nx Affected：<https://nx.dev/docs/features/ci-features/affected>
