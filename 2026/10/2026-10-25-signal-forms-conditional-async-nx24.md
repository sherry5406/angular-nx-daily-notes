# Signal Forms 條件式 × 非同步驗證 × Nx 24 Task-based Affected｜2026-10-25

> 日期：2026-10-25  
> 課程進度：Signal Forms Day 12 — Conditional / Async Validation  
> 今日重點：先用同步規則擋掉無效輸入，再用 validateHttp() 驗證伺服器狀態；搭配 Nx 24 的 task-based affected 減少不必要的 CI 工作。

---

## 1. 今日最值得學習的 Angular 技術

今天的主軸是 **Signal Forms 的條件式驗證與非同步驗證**。

實務表單常遇到兩種規則：

1. **條件式驗證**：只有選擇「公司帳號」時，公司名稱才必填。
2. **非同步驗證**：使用者名稱格式正確後，才呼叫 API 確認是否已被註冊。

Signal Forms 讓驗證規則集中在 schema，並將結果反映在 FieldTree 的狀態訊號中。

### 今天要掌握的 API

| API | 用途 |
|---|---|
| form() | 將 model signal 與表單 schema 建立關聯 |
| required()、email()、minLength() | 宣告同步驗證規則 |
| applyWhen() | 條件成立時才套用一組 schema 規則 |
| debounce() | 延遲 UI 變更寫入 form model，減少每次按鍵都觸發後續工作 |
| validateHttp() | 使用 HTTP API 執行非同步驗證 |
| pending() | 顯示非同步驗證進行中 |
| errors()、invalid()、touched() | 呈現欄位驗證狀態與錯誤 |
| submit() | 依 Signal Forms 的提交流程執行動作 |

**重要觀念：** 非同步驗證會等同步驗證通過後才執行。當欄位值改變時，進行中的驗證請求會自動取消，避免舊值的回應覆蓋新值的驗證結果。

官方文件：
- [Signal Forms Validation](https://angular.dev/guide/forms/signals/validation)
- [Signal Forms Async Operations](https://angular.dev/guide/forms/signals/async-operations)
- [Signal Forms Schemas](https://angular.dev/guide/forms/signals/schemas)

---

## 2. 核心概念：把驗證分成三個階段

### 2.1 同步驗證先擋掉明顯無效的輸入

~~~ts
required(path.username, {
  message: '請輸入使用者名稱',
});

minLength(path.username, 3, {
  message: '使用者名稱至少需要 3 個字元',
});
~~~

當使用者輸入空字串或只有 1～2 個字元時，這些規則就會失敗，不需要浪費 API 請求。

### 2.2 通過同步驗證後，再執行 HTTP 驗證

~~~ts
validateHttp(path.username, {
  request: ({value}) => {
    const username = value().trim();

    return username
      ? '/api/users/availability?username=' + encodeURIComponent(username)
      : undefined;
  },
  onSuccess: (response: {available: boolean}) =>
    response.available
      ? null
      : {
          kind: 'usernameTaken',
          message: '這個使用者名稱已被使用',
        },
  onError: () => ({
    kind: 'availabilityCheckFailed',
    message: '目前無法確認使用者名稱，請稍後再試',
  }),
});
~~~

重點：

- request 回傳 undefined 時，略過這次請求。
- onSuccess 將 API 回應轉成驗證結果；回傳 null 或 undefined 表示有效。
- onError 處理網路或 HTTP 失敗。**無法連線不等於使用者名稱可用**，不可默默當成驗證成功。

### 2.3 將狀態反映在 UI

~~~html
@if (registrationForm.username().pending()) {
  <p role="status">正在確認使用者名稱……</p>
}

@if (
  registrationForm.username().touched() &&
  registrationForm.username().invalid()
) {
  @for (error of registrationForm.username().errors(); track $index) {
    <p class="error" role="alert">{{ error.message }}</p>
  }
}
~~~

pending() 是訊號，直接在 template 中讀取即可。不要再建立另一個 boolean 並在每個 HTTP callback 手動切換，否則會有兩份需要同步的狀態。

---

## 3. 可以直接實作的完整範例

需求：

1. 使用者名稱必填，至少 3 個字元。
2. 通過同步驗證後，呼叫 API 確認使用者名稱是否可用。
3. Email 必填且格式正確。
4. 選擇「公司帳號」後，公司名稱才必填。
5. 驗證進行中或表單無效時，不允許提交。
6. API 驗證失敗與註冊失敗都要有清楚提示。

### 3.1 型別與 API Service

建立 registration-api.service.ts：

~~~ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export type AccountType = 'personal' | 'company';

export interface RegistrationModel {
  username: string;
  email: string;
  accountType: AccountType;
  companyName: string;
}

export interface UsernameAvailabilityResponse {
  available: boolean;
}

@Injectable({ providedIn: 'root' })
export class RegistrationApi {
  private readonly http = inject(HttpClient);

  register(payload: RegistrationModel): Promise<void> {
    return firstValueFrom(
      this.http.post<void>('/api/registrations', payload),
    );
  }
}
~~~

### 3.2 表單元件

建立 registration.component.ts：

~~~ts
import { Component, inject, signal } from '@angular/core';
import {
  applyWhen,
  debounce,
  email,
  form,
  FormField,
  minLength,
  required,
  submit,
  validateHttp,
} from '@angular/forms/signals';

import {
  RegistrationApi,
  RegistrationModel,
  UsernameAvailabilityResponse,
} from './registration-api.service';

@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [FormField],
  templateUrl: './registration.component.html',
  styles: [
    'form { display: grid; gap: 0.75rem; max-width: 28rem; } label { display: grid; gap: 0.25rem; } .error { color: #b42318; margin: 0; }',
  ],
})
export class RegistrationComponent {
  private readonly registrationApi = inject(RegistrationApi);

  readonly model = signal<RegistrationModel>({
    username: '',
    email: '',
    accountType: 'personal',
    companyName: '',
  });

  readonly submitMessage = signal('');

  readonly registrationForm = form(this.model, (path) => {
    required(path.username, {
      message: '請輸入使用者名稱',
    });

    minLength(path.username, 3, {
      message: '使用者名稱至少需要 3 個字元',
    });

    // UI 停止變動一段時間後，才將值寫入 model。
    debounce(path.username, 350);

    validateHttp(path.username, {
      request: ({value}) => {
        const username = value().trim();

        return username.length >= 3
          ? '/api/users/availability?username=' +
              encodeURIComponent(username)
          : undefined;
      },
      onSuccess: (response: UsernameAvailabilityResponse) =>
        response.available
          ? null
          : {
              kind: 'usernameTaken',
              message: '這個使用者名稱已被使用',
            },
      onError: () => ({
        kind: 'availabilityCheckFailed',
        message: '目前無法確認使用者名稱，請稍後再試',
      }),
    });

    required(path.email, {
      message: '請輸入 Email',
    });

    email(path.email, {
      message: 'Email 格式不正確',
    });

    // 只有公司帳號才套用公司名稱必填規則。
    applyWhen(
      path,
      ({valueOf}) => valueOf(path.accountType) === 'company',
      (companyPath) => {
        required(companyPath.companyName, {
          message: '公司帳號必須填寫公司名稱',
        });
      },
    );
  });

  onSubmit(event: Event): void {
    event.preventDefault();
    this.submitMessage.set('');

    submit(this.registrationForm, {
      action: async () => {
        try {
          await this.registrationApi.register(this.model());
          this.submitMessage.set('註冊成功！');
        } catch {
          this.submitMessage.set('註冊失敗，請稍後再試。');
        }
      },
    });
  }
}
~~~

### 3.3 表單 Template

建立 registration.component.html：

~~~html
<form novalidate (submit)="onSubmit($event)">
  <h2>會員註冊</h2>

  <label>
    使用者名稱
    <input
      [formField]="registrationForm.username"
      autocomplete="username"
    />
  </label>

  @if (registrationForm.username().pending()) {
    <p role="status">正在確認使用者名稱……</p>
  }

  @if (
    registrationForm.username().touched() &&
    registrationForm.username().invalid()
  ) {
    @for (error of registrationForm.username().errors(); track $index) {
      <p class="error" role="alert">{{ error.message }}</p>
    }
  }

  <label>
    Email
    <input
      type="email"
      [formField]="registrationForm.email"
      autocomplete="email"
    />
  </label>

  @if (
    registrationForm.email().touched() &&
    registrationForm.email().invalid()
  ) {
    @for (error of registrationForm.email().errors(); track $index) {
      <p class="error" role="alert">{{ error.message }}</p>
    }
  }

  <label>
    帳號類型
    <select [formField]="registrationForm.accountType">
      <option value="personal">個人帳號</option>
      <option value="company">公司帳號</option>
    </select>
  </label>

  @if (model().accountType === 'company') {
    <label>
      公司名稱
      <input [formField]="registrationForm.companyName" />
    </label>

    @if (
      registrationForm.companyName().touched() &&
      registrationForm.companyName().invalid()
    ) {
      @for (
        error of registrationForm.companyName().errors();
        track $index
      ) {
        <p class="error" role="alert">{{ error.message }}</p>
      }
    }
  }

  <button
    type="submit"
    [disabled]="
      registrationForm().invalid() ||
      registrationForm().pending()
    "
  >
    建立帳號
  </button>

  @if (submitMessage()) {
    <p role="status">{{ submitMessage() }}</p>
  }
</form>
~~~

### 3.4 啟用 HttpClient

在 app.config.ts 設定：

~~~ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};
~~~

請確認應用程式的 bootstrap 使用了這份 appConfig。

### 3.5 後端 API 契約

**GET /api/users/availability?username=...**

可用時回傳：

~~~json
{
  "available": true
}
~~~

已被使用時回傳：

~~~json
{
  "available": false
}
~~~

**POST /api/registrations** 接收：

~~~json
{
  "username": "sherry_dev",
  "email": "sherry@example.com",
  "accountType": "company",
  "companyName": "Example Co."
}
~~~

這份範例包含可實作的元件、驗證 schema、HTTP Service 與 HttpClient 設定；實際執行時仍需要有符合契約的後端 API。

### 3.6 實作時要特別注意

- debounce(path.username, 350) 是延遲 UI 變更寫入 model，不是通用的 RxJS debounceTime() 替代品。
- validateHttp() 已處理欄位值變更時的過期請求取消；一般 HTTP 驗證不需要再自行包一層 switchMap()。
- pending() 不代表驗證成功；送出前要同時考慮 invalid() 與 pending()。
- 前端驗證只改善使用者體驗，後端仍必須重新驗證使用者名稱唯一性、Email、帳號類型與公司名稱。
- 若 API 回傳 500 或發生網路錯誤，應讓使用者知道目前無法完成驗證，不要將錯誤當成「名稱可用」。

---

## 4. Signal Forms 持續課程：Day 12

前一階段已學習 Cross-field Logic，包含 valueOf()、stateOf() 與 fieldTreeOf()。今天進入 **Conditional / Async Validation**，不重複介紹 signal()、form() 或基本欄位綁定。

| 學習項目 | 本日目標 |
|---|---|
| 條件式規則 | 用 applyWhen() 控制規則何時生效 |
| 同步與非同步順序 | 先驗證格式，再呼叫 API |
| 非同步狀態 | 用 pending() 顯示等待狀態 |
| 錯誤處理 | 用 onError() 將請求失敗轉成使用者可理解的訊息 |
| 降低請求量 | 用 debounce() 避免每個按鍵都更新 model |
| 提交流程 | 用 submit() 管理提交動作 |

### 練習題

1. 將帳號類型改為 personal 時，公司名稱不再必填。
2. 將使用者名稱改成少於 3 個字元時，確認不會送出 availability API。
3. 快速輸入多個使用者名稱，確認舊值的驗證結果不會覆蓋最新值。
4. 暫時讓 availability API 回傳錯誤，確認畫面會顯示無法驗證，而不是顯示名稱可用。
5. 只有在驗證完成且表單有效時，才允許進入註冊流程。

下一課建議延伸 **Signal Forms 的驗證測試與 API 錯誤對應**，練習將 schema 測試與元件 DOM 測試分開。

官方延伸閱讀：[Signal Forms Testing](https://angular.dev/guide/forms/signals/testing)。

---

## 5. Nx Monorepo 實戰：Nx 24 的 task-based affected

截至 2026-10-25，Nx 官方已發布 **Nx 24.0（2026-10-21）**。Task-based affected 不只判斷哪些 project 受影響，也能進一步判斷哪些 task 的輸入真的受影響。

### 5.1 建議的專案分層

~~~text
apps/
└── portal/

libs/
└── registration/
    ├── feature/       # 表單頁面與使用者互動
    ├── data-access/   # RegistrationApi、HTTP 與 DTO
    └── ui/            # 共用欄位、錯誤訊息與表單控制項
~~~

責任分工：

- feature：組合 Signal Forms schema、處理頁面流程。
- data-access：管理 HTTP 呼叫、DTO 與 API 錯誤轉換。
- ui：重用表單欄位與錯誤顯示；不要在共用 UI 元件內複製業務驗證規則。

正式專案可以將 RegistrationApi 搬到 libs/registration/data-access，讓 feature 元件只依賴公開的 data-access API。

### 5.2 檢查專案圖與可執行任務

~~~bash
npx nx show projects
npx nx graph

npx nx affected -t lint test build \
  --base=origin/main \
  --head=HEAD \
  --explain
~~~

--explain 可協助理解任務為何被選中。實際能執行哪些 target，取決於 workspace 的 project 設定與 plugins。

### 5.3 Nx 24 升級注意事項

1. **Task-based affected**：新 workspace 預設採用更精準的 task 選擇；升級的既有 workspace 可能保留 affected.legacy: true。先確認團隊理解新的選擇結果，再移除 legacy 設定。
2. **已棄用 executors 移除**：Nx 23 已棄用且有 inferred task 替代方案的部分 executors，在 Nx 24 已移除。升級前可先評估並執行 npx nx g infer-targets。
3. **持續任務的 readiness**：可設定依賴任務等待 dev server 準備完成，再執行 E2E 測試。
4. **Angular 22.2 相容性**：官方指出，使用 Cypress component testing 或 Angular webpack Module Federation 的 workspace，可能遇到 Angular 22.2 所帶入 webpack 版本的相容問題；升級前務必檢查官方 migration 說明。

建議升級流程：

~~~bash
npx nx migrate latest
npx nx migrate --run-migrations

npx nx affected -t lint test build \
  --base=origin/main \
  --head=HEAD \
  --explain
~~~

不要只因為 nx migrate 成功就直接合併。還要檢查 build、unit test、E2E、Module Federation、Cypress component testing 與 CI cache。

官方來源：
- [Nx 24.0 Changelog](https://nx.dev/changelog)
- [Nx 23.3 Release](https://nx.dev/blog/nx-23-3-release)
- [Nx Angular / Nx Version Matrix](https://nx.dev/docs/kb/angular-nx-version-matrix)

---

## 6. Angular 21 與最新 Angular 的差異／升級注意事項

以下依 **2026-10-25** Angular 官方版本資料整理：

| 項目 | Angular 21 | Angular 22 |
|---|---|---|
| 支援狀態 | LTS | Active |
| Node.js | ^20.19.0、^22.12.0 或 ^24.0.0 | ^22.22.3、^24.15.0 或 ^26.0.0 |
| TypeScript | >=5.9.0 <6.0.0 | >=6.0.0 <6.1.0 |
| RxJS | ^6.5.3 或 ^7.4.0 | ^6.5.3 或 ^7.4.0 |
| Signal Forms API | 不應直接假設與 v22 相同的穩定性承諾 | 官方 API 文件將相關 API 標示為自 v22.0 起 stable |

### 升級建議

1. **先檢查 Node 與 TypeScript**：Angular 22 的 TypeScript 版本區間已進入 6.0.x，不能假設 Angular 21 的 TypeScript 5.9 設定可原封不動沿用。
2. **逐一確認 Angular 套件版本**：@angular/core、@angular/common、@angular/forms、CLI 與相關套件應遵循官方升級指南。
3. **先確認 Nx 相容矩陣**：不要只升級 Angular 而忽略 nx、@nx/angular 與其他 @nx/* 套件的版本配對。
4. **一次升一個主要版本**：使用官方 Update Guide，完成 migration、測試與建置後再進行下一步。
5. **Signal Forms 的穩定性要分版本判斷**：若專案仍是 Angular 21，先查該版本對應文件與 API 狀態；不要直接將 v22 文件中的 stable 標示套用到 v21。

官方來源：
- [Angular Version Compatibility](https://angular.dev/reference/versions)
- [Angular Versioning and Releases](https://angular.dev/reference/releases)
- [Angular Update Guide：v21 → v22](https://angular.dev/update-guide?from=21.0&to=22.0)

---

## 7. 今日實作 Checklist

### Angular / Signal Forms

- [ ] 將同步驗證與非同步驗證分開。
- [ ] 使用 applyWhen() 讓公司名稱只在公司帳號情境下必填。
- [ ] 使用 validateHttp() 檢查使用者名稱。
- [ ] 使用 pending() 呈現等待狀態。
- [ ] 用 onError() 顯示可理解的 API 失敗訊息。
- [ ] 使用 debounce() 降低連續輸入造成的 model 更新與驗證請求。
- [ ] 確認提交時不會在表單無效或驗證仍進行中時執行註冊。
- [ ] 確認後端仍會重新驗證所有重要業務規則。

### Nx / 升級

- [ ] 查看 nx graph，確認 feature、data-access、ui 的依賴方向。
- [ ] 確認 workspace 是否使用 Nx 24 已移除的 executors 或 config helpers。
- [ ] 執行 nx affected -t lint test build --explain，理解任務選擇結果。
- [ ] 檢查 Angular、Nx、TypeScript、Node.js 版本相容性。
- [ ] 若使用 Angular 22.2，確認 Cypress component testing 與 webpack Module Federation 相容性。
- [ ] 升級前先建立分支，並完成 build、unit test、E2E 與 CI 驗證。

---

## 8. 今日一句話

> **把規則放進 schema，把非同步狀態交給 Signal Forms，把真正受影響的任務交給 Nx；讓程式碼負責明確的事，而不是讓團隊手動同步一堆狀態。**

---

## 9. 官方來源

### Angular

- [Angular Signal Forms：Validation](https://angular.dev/guide/forms/signals/validation)
- [Angular Signal Forms：Async Operations](https://angular.dev/guide/forms/signals/async-operations)
- [Angular Signal Forms：Schemas](https://angular.dev/guide/forms/signals/schemas)
- [Angular Signal Forms：Testing](https://angular.dev/guide/forms/signals/testing)
- [Angular Version Compatibility](https://angular.dev/reference/versions)
- [Angular Versioning and Releases](https://angular.dev/reference/releases)
- [Angular Update Guide](https://angular.dev/update-guide)

### Nx

- [Nx 24.0 Changelog](https://nx.dev/changelog)
- [Nx 23.3 Release](https://nx.dev/blog/nx-23-3-release)
- [Nx Release Schedule and Support Policy](https://nx.dev/docs/reference/releases)
- [Nx Angular / Nx Version Matrix](https://nx.dev/docs/kb/angular-nx-version-matrix)

---

> 本文版本資訊以 2026-10-25 可查閱的官方文件為準。升級前請再次核對相容矩陣與 migration 指引。
