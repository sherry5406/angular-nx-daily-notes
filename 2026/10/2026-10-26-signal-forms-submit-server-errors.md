# Signal Forms 提交流程 × Server Error Mapping｜2026-10-26

> 日期：2026-10-26  
> 課程進度：Signal Forms Day 13 — Submit Lifecycle / Server Error Mapping  
> 今日主軸：讓表單提交、非同步驗證、欄位錯誤與 Nx CI 工作選擇形成一條可靠的交付流程。

---

## 1. 今日最值得學習的 Angular 技術

今天延續昨天的 Conditional / Async Validation，往下一步學習：**驗證通過之後，如何安全地提交資料，並把後端錯誤正確顯示在欄位上。**

真實表單通常不只需要檢查輸入格式，還要處理：

- 提交時自動顯示欄位錯誤。
- 非同步驗證尚未完成時，不要過早送出。
- 避免使用者連續點擊造成重複提交。
- 後端回覆「Email 已註冊」時，將錯誤顯示在 Email 欄位。
- 網路或伺服器暫時失敗時，顯示可理解的表單層級錯誤。
- 使用者修正欄位後，讓過期的伺服器提交錯誤自動清除。

Angular Signal Forms 的 `submit()`、`FormRoot`、`submitting()` 與提交錯誤的 `fieldTree` 機制，可以把這些工作放進一致的提交生命週期。

### 今日核心 API

| API | 用途 |
|---|---|
| `form()` 的 `submission.action` | 定義表單驗證成功後要執行的非同步提交動作 |
| `FormRoot` / `formRoot` | 接管表單 submit 事件、阻止瀏覽器預設導頁並呼叫 Signal Forms 提交流程 |
| `submit()` | 需要手動控制時，直接啟動提交流程 |
| `submitting()` | 顯示提交中狀態、避免重複點擊 |
| `ignoreValidators: 'none'` | 等待所有驗證完成，包含仍在執行的非同步驗證 |
| `fieldTree` | 將後端錯誤指定到特定欄位 |
| `onInvalid` | 驗證失敗時執行額外處理，例如聚焦第一個錯誤欄位 |
| `errors()` | 讀取欄位或表單層級的錯誤 |

官方說明：Signal Forms 的 `submit()` 會先標記可互動欄位為 touched，再檢查驗證；驗證失敗時不執行 action。action 執行期間 `submitting()` 為 true，完成或回傳錯誤後會恢復。citeturn3view0

---

## 2. 核心概念：把「驗證」與「提交」分開

### 2.1 驗證成功不代表 API 一定成功

前端驗證能檢查必填、Email 格式與使用者名稱是否可用，但後端仍可能拒絕請求，例如：

- 另一位使用者剛好搶先註冊相同 Email。
- 權限或商業規則不允許這次操作。
- 網路中斷或伺服器暫時無法回應。

所以要分清楚兩種錯誤：

| 錯誤類型 | 例子 | 適合的處理方式 |
|---|---|---|
| 驗證錯誤 | Email 格式錯誤、使用者名稱已被使用 | 由 schema 驗證規則產生 |
| 提交錯誤 | Email 已被其他請求註冊、伺服器暫時失敗 | 由 `submission.action` 回傳錯誤 |

### 2.2 `FormRoot` 與 `submission.action` 怎麼合作？

最常見的寫法如下：

~~~ts
const registrationForm = form(
  model,
  (path) => {
    required(path.email);
    email(path.email);
  },
  {
    submission: {
      action: async (field) => {
        // 只有驗證流程允許提交時才執行。
        await saveToServer(field().value());
      },
      ignoreValidators: 'none',
    },
  },
);
~~~

HTML 綁定：

~~~html
<form [formRoot]="registrationForm">
  <!-- 欄位 -->
  <button type="submit">建立帳號</button>
</form>
~~~

使用 `FormRoot` 時，不需要另外手動呼叫 `preventDefault()`，也不需要在每個 click handler 中自行管理 submitting 狀態。

### 2.3 為什麼今天特別使用 `ignoreValidators: 'none'`？

Signal Forms 預設的提交策略是 `'pending'`：只要沒有已確認的驗證錯誤，即使非同步驗證還在執行，也可以繼續提交。

| 設定 | 行為 | 常見用途 |
|---|---|---|
| `'pending'`（預設） | 驗證沒有失敗就可提交，即使仍有 pending validator | 不要求等待所有背景驗證的流程 |
| `'none'` | 所有驗證都必須完成且通過，才提交 | 註冊、建立唯一帳號等流程 |
| `'all'` | 不因驗證狀態而阻擋提交 | 儲存草稿等允許暫存無效資料的情境 |

對「先確認 Email / 使用者名稱，再建立帳號」這種流程，`'none'` 通常更符合預期。注意：這不代表後端可以省略驗證；後端仍必須負責最終一致性與安全性。citeturn3view1

---

## 3. 可以直接實作的完整範例

我們做一個簡化的「會員註冊」表單：

1. 使用者名稱必填，至少 3 個字元。
2. 使用者名稱格式通過後，呼叫 API 檢查是否可用。
3. Email 必填且格式正確。
4. 提交前等待所有非同步驗證完成。
5. 若後端回傳 HTTP 409，將「Email 已註冊」放到 Email 欄位。
6. 若發生其他錯誤，顯示表單層級錯誤。
7. 提交中停用按鈕，避免重複送出。

> 以下假設後端提供 `GET /api/users/availability?username=...` 與 `POST /api/registrations`。請依實際 API 契約調整 URL 與 DTO。

### 3.1 元件 TypeScript

建立 `registration.component.ts`：

~~~ts
import { Component, inject, signal } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import {
  email,
  form,
  FormField,
  FormRoot,
  minLength,
  required,
  validateHttp,
} from '@angular/forms/signals';
import { firstValueFrom } from 'rxjs';

interface RegistrationModel {
  username: string;
  email: string;
}

interface UsernameAvailabilityResponse {
  available: boolean;
}

interface RegistrationResponse {
  id: string;
}

@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [FormField, FormRoot],
  template: `
    <form [formRoot]="registrationForm">
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

      @if (registrationForm().errors().length > 0) {
        @for (error of registrationForm().errors(); track $index) {
          <p class="error" role="alert">{{ error.message }}</p>
        }
      }

      <button
        type="submit"
        [disabled]="registrationForm().submitting()"
      >
        @if (registrationForm().submitting()) {
          送出中……
        } @else {
          建立帳號
        }
      </button>

      @if (successMessage()) {
        <p role="status">{{ successMessage() }}</p>
      }
    </form>
  `,
  styles: [`
    form { display: grid; gap: 0.75rem; max-width: 28rem; }
    label { display: grid; gap: 0.25rem; }
    .error { color: #b42318; margin: 0; }
  `],
})
export class RegistrationComponent {
  private readonly http = inject(HttpClient);

  readonly model = signal<RegistrationModel>({
    username: '',
    email: '',
  });

  readonly successMessage = signal('');

  readonly registrationForm = form(
    this.model,
    (path) => {
      required(path.username, {
        message: '請輸入使用者名稱',
      });

      minLength(path.username, 3, {
        message: '使用者名稱至少需要 3 個字元',
      });

      validateHttp(path.username, {
        request: ({ value }) => {
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
    },
    {
      submission: {
        // 確保非同步驗證也已完成且通過。
        ignoreValidators: 'none',

        action: async (field) => {
          this.successMessage.set('');

          try {
            await firstValueFrom(
              this.http.post<RegistrationResponse>(
                '/api/registrations',
                field().value(),
              ),
            );

            this.successMessage.set('註冊成功！');
          } catch (error) {
            if (
              error instanceof HttpErrorResponse &&
              error.status === 409
            ) {
              // 將後端的 Email 衝突錯誤指向 Email 欄位。
              return {
                kind: 'emailTaken',
                message: '此 Email 已經註冊，請改用其他 Email。',
                fieldTree: field.email,
              };
            }

            // 沒有 fieldTree 時，錯誤會顯示在提交的表單節點。
            return {
              kind: 'submitFailed',
              message: '目前無法完成註冊，請稍後再試。',
            };
          }
        },

        onInvalid: (field) => {
          // 驗證失敗時，聚焦第一個錯誤欄位。
          const firstError = field().errorSummary()[0];
          firstError?.fieldTree().focusBoundControl();
        },
      },
    },
  );
}
~~~

### 3.2 啟用 HttpClient

在應用程式的 `app.config.ts` 設定：

~~~ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};
~~~

確認啟動應用程式時使用了這份設定。若專案已經呼叫 `provideHttpClient()`，不需要重複加入。

### 3.3 後端 API 契約

`GET /api/users/availability?username=...`

可用時：

~~~json
{
  "available": true
}
~~~

已被使用時：

~~~json
{
  "available": false
}
~~~

`POST /api/registrations` 接收：

~~~json
{
  "username": "sherry_dev",
  "email": "sherry@example.com"
}
~~~

成功時回傳註冊結果；若 Email 已存在，建議回傳 HTTP `409 Conflict`，讓前端能區分業務衝突與一般伺服器錯誤。

### 3.4 這個範例如何運作？

1. 使用者輸入使用者名稱，先跑 `required()`、`minLength()`。
2. 同步規則通過後，才執行 `validateHttp()`。
3. `pending()` 反映正在檢查的狀態；欄位值改變時，Signal Forms 會取消過期的非同步驗證請求。
4. 使用者送出表單，`FormRoot` 啟動提交流程。
5. `ignoreValidators: 'none'` 要求所有驗證完成且通過。
6. `submitting()` 讓按鈕顯示「送出中」，避免重複點擊。
7. 後端回傳 409 時，透過 `fieldTree: field.email` 把錯誤放在 Email 欄位。
8. 其他錯誤則回傳一般提交錯誤，顯示在表單層級。
9. 使用者修改有提交錯誤的欄位後，該欄位的提交錯誤會自動清除；下一次提交再重新向伺服器確認。

Signal Forms 官方文件指出，提交錯誤會與欄位的 `errors()` 一起呈現，並在使用者修改該欄位時自動清除。citeturn3view1

### 3.5 實作注意事項

- 前端「檢查可用」與實際註冊之間存在時間差；後端資料庫仍應以唯一索引或交易保證 Email 唯一性。
- 不要把所有 HTTP 錯誤都轉成「Email 已註冊」；只有明確的 409 衝突才做欄位映射。
- `submitting()` 是 Signal Forms 提供的狀態，不需要再自行維護一個重複的 `isSubmitting` boolean。
- 範例中的 `successMessage` 用於顯示成功結果；真實產品可在成功後導頁、清空表單或更新列表。
- 如果要將後端回傳的多個欄位錯誤映射到表單，請先用白名單將後端欄位名稱對應到明確的 FieldTree，避免任意字串索引造成型別與執行期問題。

---

## 4. Signal Forms 持續課程：Day 13

前一課 Day 12 已完成 Conditional / Async Validation。今天不重新講解基礎，而是把驗證流程接到真正的提交流程。

| 學習項目 | Day 13 目標 |
|---|---|
| `FormRoot` | 讓表單提交事件交給 Signal Forms 管理 |
| `submission.action` | 將目前表單值送往後端 |
| `ignoreValidators: 'none'` | 等待所有非同步驗證完成 |
| `submitting()` | 顯示送出中並避免重複提交 |
| `fieldTree` | 將伺服器業務錯誤映射到特定欄位 |
| `onInvalid` | 驗證失敗時聚焦第一個錯誤 |
| `errors()` | 同時顯示驗證錯誤與提交錯誤 |

### 今日練習

1. 讓註冊 API 回傳 409，確認 Email 欄位出現後端錯誤。
2. 修改 Email，確認舊的提交錯誤自動清除。
3. 人為延遲使用者名稱驗證 API，確認表單會等待非同步驗證。
4. 讓註冊 API 回傳 500，確認顯示一般提交錯誤，而不是 Email 衝突。
5. 連續點擊提交按鈕，確認不會送出重複請求。

**下一課建議：** Day 14 — Signal Forms 隔離測試與元件測試：測試 schema、伺服器錯誤映射、pending/submitting 狀態與 DOM 行為。

---

## 5. Nx Monorepo 實戰：Task-based Affected × CI 精準執行

### 5.1 今日官方更新

截至 2026-10-26：

- **Nx 24.0** 已於 2026-10-21 發布，包含 task-based affected、持續任務的 `waitFor` / `readyWhen`，以及 Node.js app 工作流程更新。
- **Nx Cloud Ultracache 與 Dynamic Task Packing** 已於 2026-10-22 開放。Ultracache 依 CI 任務實際讀取的檔案計算快取；Dynamic Task Packing 會依任務過去的 CPU / 記憶體使用情況分配工作，減少 worker 閒置。

來源：[Nx Changelog](https://nx.dev/changelog)、[Nx affected 官方指南](https://nx.dev/docs/features/ci-features/affected)。citeturn2view1turn1view3

### 5.2 Task-based Affected 是什麼？

傳統 project-based affected 的思考方式是：「這個專案受到影響，所以把它的多個任務都跑一次。」

Task-based affected 更精細地問：「這次改動會影響哪些任務的輸入？」

例如只修改 `libs/registration/ui/src/lib/email-field.spec.ts`：

- 可能需要執行該 UI library 的 `test`。
- 若這份 spec 不屬於 production build 的輸入，就不必因此重跑該 library 的 `build`。
- 只有真正受影響的相依任務才需要執行。

### 5.3 Angular + Nx 的建議分層

~~~text
apps/
  admin/                         # 頁面、路由、組裝 feature
libs/
  registration/
    feature/                     # 頁面流程與表單提交
    data-access/                 # HttpClient、DTO、API mapping
    ui/                          # 可重用欄位與錯誤顯示元件
    util/                        # 純函式與共用型別
~~~

建議責任：

- `feature`：負責表單 schema、提交流程、畫面狀態。
- `data-access`：負責 API 呼叫、DTO 與錯誤回應轉換。
- `ui`：負責欄位呈現、錯誤訊息、載入狀態。
- `util`：放置不依賴 Angular UI 的純邏輯。

這樣修改 API mapping 時，Nx 可以依實際依賴與任務輸入選擇需要重跑的工作，而不是每次都建置整個 workspace。

### 5.4 CI 指令實戰

在完整 Git 歷史已取得、且 base ref 可用的 CI 環境中，可以執行：

~~~bash
pnpm nx affected -t lint test build --base=origin/main --head=HEAD
~~~

先在本機確認受影響的專案與任務：

~~~bash
pnpm nx show projects --affected --base=origin/main --head=HEAD
pnpm nx affected -t test --base=origin/main --head=HEAD
~~~

> 若 CI 使用淺層 clone，請先確保比較基準 commit 已抓取。否則 Nx 可能無法正確計算變更範圍。

### 5.5 Nx 24 升級特別注意

Nx 24 升級既有 workspace 時，migration 會設定 `affected.legacy: true`，保留原本 project-based 行為。不要以為升級完成就已經切換到 task-based affected。

建議流程：

1. 先執行 `pnpm nx migrate latest`，檢查產生的 migration。
2. 依官方流程安裝相依套件並執行 migrations。
3. 檢查 `nx.json` 中的 `affected.legacy`。
4. 檢查各 target 的 `inputs`、`outputs` 與 cache 設定，確認任務輸入描述正確。
5. 在 CI 先比較舊行為與 task-based affected 選出的任務。
6. 確認測試、建置與 E2E 都沒有漏跑，再移除 legacy 設定或設為 `false`。

Nx 24 也移除了 Nx 23 已棄用的部分 executors、generators 與設定 helper；升級前務必閱讀 changelog，不要只更新 package version。citeturn3view2

### 5.6 Ultracache 與 Dynamic Task Packing

- **Ultracache：** 根據 CI 任務實際讀取的檔案建立更精準的 cache key，降低手動宣告 inputs 不完整造成的問題。
- **Dynamic Task Packing：** 根據任務資源使用情況分配工作，減少固定 worker 配置造成的閒置。
- **先後順序：** 先確保 affected 選對任務，再確認 cache inputs 正確，最後才評估分散式 worker 的資源分配。

這些功能不代表可以省略測試或錯誤的依賴宣告；請透過 CI timeline、cache hit 與任務結果驗證實際效果。既有 Nx Cloud workspace 需依官方說明確認 Ultracache 與 Dynamic Task Packing 的啟用狀態。citeturn2view1

---

## 6. Angular 21 與最新 Angular：版本及升級注意事項

截至 **2026-10-26**，Angular 官方版本支援表將 Angular 22 列為 Active，Angular 21 列為 LTS。官方版本相容性表目前列出的 Node.js / TypeScript 範圍如下；升級前仍應依實際使用的 minor 版本再次核對。citeturn2view2turn1view2

| 項目 | Angular 21.x | Angular 22.0.x（官方相容性表） |
|---|---|---|
| 支援狀態 | LTS | Active |
| Node.js | `^20.19.0 || ^22.12.0 || ^24.0.0` | `^22.22.3 || ^24.15.0 || ^26.0.0` |
| TypeScript | `>=5.9.0 <6.0.0` | `>=6.0.0 <6.1.0` |
| RxJS | `^6.5.3 || ^7.4.0` | `^6.5.3 || ^7.4.0` |

### 升級檢查清單

1. 先確認目前 `@angular/core`、`@angular/cli`、TypeScript、Node.js 與 Nx 的實際版本。
2. Angular core 與 CLI 的 major 版本應一致。
3. 不要只升級 Angular；同時檢查 Nx Angular plugin 的支援矩陣。
4. Angular 22.2 的支援與相容性要依官方對應版本確認；不要直接把 22.0.x 的表格當成所有 22.2 專案的完整承諾。
5. Nx 24 changelog 指出，Angular 22.2 migration 會略過使用 Cypress component testing 或 webpack Module Federation 的 Angular workspace，因相關 webpack 版本存在相容性問題。這類專案應先閱讀 Nx migration 說明，再安排升級。citeturn3view2
6. 升級後依序執行 lint、unit test、build 與 E2E；不要只以「安裝成功」判定升級完成。

官方文件：

- [Angular Version Compatibility](https://angular.dev/reference/versions)
- [Angular Versioning and Releases](https://angular.dev/reference/releases)
- [Angular Update Guide](https://angular.dev/update-guide)
- [Nx Changelog](https://nx.dev/changelog)
- [Nx Angular / Nx Version Matrix](https://nx.dev/docs/kb/angular-nx-version-matrix)

---

## 7. 今日實作 Checklist

### Signal Forms

- [ ] 使用 `FormRoot` 綁定表單。
- [ ] 將 API 提交放在 `submission.action`。
- [ ] 對需要等待的非同步驗證設定 `ignoreValidators: 'none'`。
- [ ] 使用 `submitting()` 顯示送出中狀態並避免重複提交。
- [ ] 將 HTTP 409 的 Email 衝突映射到 `field.email`。
- [ ] 將其他提交失敗呈現在表單層級。
- [ ] 驗證失敗時聚焦第一個錯誤欄位。
- [ ] 確認後端仍會執行唯一性與業務規則驗證。

### Nx Monorepo

- [ ] 確認 workspace 的 Nx 版本與 migration 狀態。
- [ ] 確認 `affected.legacy` 是否仍啟用。
- [ ] 檢查 build / test / e2e 的 task inputs。
- [ ] 用 `nx affected` 檢查實際被選中的任務。
- [ ] 確認 CI 能取得 base commit。
- [ ] 閱讀 Nx 24 breaking changes。
- [ ] 在 CI 中確認快取命中與任務結果，再評估 Ultracache / Dynamic Task Packing。

---

## 8. 今日一句話

**可靠的表單不只要知道輸入是否正確，也要把提交中、伺服器拒絕與成功狀態清楚地交還給使用者；可靠的 Monorepo CI 也一樣，精準執行比盲目執行更多任務更重要。**

---

## 9. 官方來源

### Angular

- [Angular Signal Forms — Form Submission](https://angular.dev/guide/forms/signals/form-submission)
- [Angular Signal Forms — Async Operations](https://angular.dev/guide/forms/signals/async-operations)
- [Angular Signal Forms — Validation](https://angular.dev/guide/forms/signals/validation)
- [Angular Version Compatibility](https://angular.dev/reference/versions)
- [Angular Versioning and Releases](https://angular.dev/reference/releases)
- [Angular Update Guide](https://angular.dev/update-guide)

### Nx

- [Nx Changelog](https://nx.dev/changelog)
- [Run Only Tasks Affected by a PR](https://nx.dev/docs/features/ci-features/affected)
- [Nx Release Schedule and Support Policy](https://nx.dev/docs/reference/releases)
- [Nx Angular / Nx Version Matrix](https://nx.dev/docs/kb/angular-nx-version-matrix)

---

> 本文版本資訊查閱日期：2026-10-26。範例中的 API endpoint 為示範契約，實際專案請依後端規格調整。
