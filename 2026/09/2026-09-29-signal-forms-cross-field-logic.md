# Signal Forms Cross-field Logic × `valueOf()`｜2026-09-29

> 今日主軸：Signal Forms Day 11，從「測試表單」進一步進入跨欄位邏輯。學會讓一個欄位依賴另一個欄位的值或狀態，而不是在 component 裡塞滿 `effect()` 與手動 if/else。

---

## ⭐ 今日最值得看的 Angular 技術

今天最值得學的是 **Signal Forms 的 Cross-field Logic**。

實務表單很少只有「Email 必填」這種單欄位規則，常見需求反而是：

- 確認密碼必須等於密碼
- 選擇「公司帳號」後，公司名稱才必填
- 選擇「宅配」後，地址才必填
- 結束日期不能早於開始日期
- 啟用折扣後，優惠碼才需要驗證

這些都是「一個 field 依賴另一個 field」的情境。

Signal Forms 官方提供 field context，可以在規則中使用 `valueOf()` 讀取其他欄位的值，也可以使用 `stateOf()` 讀取其他欄位狀態，使用 `fieldTreeOf()` 取得對應的 FieldTree。這比把跨欄位邏輯散落在 component event handler 裡更容易維護。

可以先記住這個模型：

```text
                    Form Model
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
       password     confirmPassword   accountType
          │             │             │
          └──────┬──────┘             │
                 ↓                    ↓
          Cross-field rule      Conditional rule
                 │                    │
                 └─────────┬──────────┘
                           ↓
                     Field State
```

---

# 🔍 核心概念 / API

## 1. `valueOf()`：讀另一個欄位的值

假設模型：

```ts
interface RegisterModel {
  password: string;
  confirmPassword: string;
}
```

在 `confirmPassword` 的 validation rule 裡，可以直接讀取 `password`：

```ts
validate(schema.confirmPassword, ({valueOf}) => {
  if (valueOf(schema.password) !== valueOf(schema.confirmPassword)) {
    return {
      kind: 'passwordMismatch',
      message: '兩次輸入的密碼不一致',
    };
  }

  return null;
});
```

重點不是「怎麼比字串」，而是：

```text
schema.confirmPassword
        ↓
validate()
        ↓
valueOf(schema.password)
        ↓
建立 reactive dependency
```

當 `password` 改變時，相關 cross-field rule 可以重新評估。Angular 官方的 Cross-field Logic 指南就是以 field context 與 `valueOf()`、`stateOf()`、`fieldTreeOf()` 作為主要 API。

---

## 2. `stateOf()`：依賴另一個欄位的狀態

有時候我們不是想知道另一欄的值，而是想知道它目前是不是 valid、pending、touched 等。

例如：

```ts
stateOf(schema.email).valid()
```

可以用來建立依賴其他 field state 的 UI 或 validation logic。

簡單理解：

```text
valueOf()
→ 我想知道「值」

stateOf()
→ 我想知道「狀態」

fieldTreeOf()
→ 我想取得「FieldTree」
```

這三個 API 是今天最重要的記憶點。

---

## 3. `fieldTreeOf()`：需要指定錯誤目標時使用

當 business rule 的錯誤應該指向另一個 field，可以取得 FieldTree：

```ts
fieldTreeOf(schema.email)
```

這在 submission error 或複雜 cross-field rule 特別有用。

例如後端告訴我們 Email 與公司帳號組合不可用，可以把錯誤指定給 Email field，而不是在 component 自己維護：

```ts
return {
  kind: 'accountConflict',
  message: '此 Email 與公司帳號組合不可用',
  fieldTree: fieldTreeOf(schema.email),
};
```

實務上要把「錯誤屬於哪個欄位」一起設計，UI 才不需要猜。

---

# 💻 可以直接實作的完整範例

下面做一個「會員註冊」表單：

需求：

1. Email 必填且格式正確
2. 密碼至少 8 碼
3. 確認密碼必須與密碼一致
4. 選擇公司帳號後，公司名稱才必填
5. Submit 時避免重複送出

```ts
import {Component, signal} from '@angular/core';
import {
  email,
  form,
  FormField,
  FormRoot,
  minLength,
  required,
  validate,
} from '@angular/forms/signals';

interface RegisterModel {
  email: string;
  password: string;
  confirmPassword: string;
  accountType: 'personal' | 'company';
  companyName: string;
}

@Component({
  selector: 'app-register',
  imports: [FormField, FormRoot],
  template: `
    <form [formRoot]="registerForm">
      <label>
        Email
        <input
          type="email"
          [formField]="registerForm.email"
        />
      </label>

      @if (registerForm.email().touched() && registerForm.email().invalid()) {
        @for (error of registerForm.email().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        密碼
        <input
          type="password"
          [formField]="registerForm.password"
        />
      </label>

      <label>
        確認密碼
        <input
          type="password"
          [formField]="registerForm.confirmPassword"
        />
      </label>

      @if (
        registerForm.confirmPassword().touched() &&
        registerForm.confirmPassword().invalid()
      ) {
        @for (
          error of registerForm.confirmPassword().errors();
          track error.kind
        ) {
          <p>{{ error.message }}</p>
        }
      }

      <label>
        帳號類型
        <select [formField]="registerForm.accountType">
          <option value="personal">個人</option>
          <option value="company">公司</option>
        </select>
      </label>

      @if (registerForm.accountType().value() === 'company') {
        <label>
          公司名稱
          <input [formField]="registerForm.companyName" />
        </label>

        @if (
          registerForm.companyName().touched() &&
          registerForm.companyName().invalid()
        ) {
          @for (
            error of registerForm.companyName().errors();
            track error.kind
          ) {
            <p>{{ error.message }}</p>
          }
        }
      }

      <button
        type="submit"
        [disabled]="registerForm().submitting()"
      >
        @if (registerForm().submitting()) {
          建立中...
        } @else {
          建立帳號
        }
      </button>
    </form>
  `,
})
export class RegisterComponent {
  registerModel = signal<RegisterModel>({
    email: '',
    password: '',
    confirmPassword: '',
    accountType: 'personal',
    companyName: '',
  });

  registerForm = form(
    this.registerModel,
    (schema) => {
      required(schema.email, {
        message: 'Email 為必填欄位',
      });

      email(schema.email, {
        message: '請輸入正確的 Email 格式',
      });

      required(schema.password, {
        message: '密碼為必填欄位',
      });

      minLength(schema.password, 8, {
        message: '密碼至少需要 8 碼',
      });

      required(schema.confirmPassword, {
        message: '請再次輸入密碼',
      });

      validate(schema.confirmPassword, ({valueOf}) => {
        if (valueOf(schema.password) !== valueOf(schema.confirmPassword)) {
          return {
            kind: 'passwordMismatch',
            message: '兩次輸入的密碼不一致',
          };
        }

        return null;
      });

      required(schema.companyName, {
        message: '公司帳號必須填寫公司名稱',
        when: ({valueOf}) => valueOf(schema.accountType) === 'company',
      });
    },
    {
      submission: {
        action: async (field) => {
          const payload = field().value();

          console.log('送出 API：', payload);

          // 實務上這裡呼叫 data-access service。
          await new Promise((resolve) => setTimeout(resolve, 800));
        },
      },
    },
  );
}
```

### 這個範例最重要的地方

不是 HTML，而是 schema：

```ts
validate(schema.confirmPassword, ({valueOf}) => {
  if (valueOf(schema.password) !== valueOf(schema.confirmPassword)) {
    return {
      kind: 'passwordMismatch',
      message: '兩次輸入的密碼不一致',
    };
  }

  return null;
});
```

以及 conditional validation：

```ts
required(schema.companyName, {
  when: ({valueOf}) => valueOf(schema.accountType) === 'company',
});
```

這代表：

```text
accountType = personal
        ↓
companyName 不需要驗證

accountType = company
        ↓
companyName required
```

Angular 官方 Validation 文件也支援使用 `when` 建立條件式 validation。

---

# 🧠 為什麼不要用 `effect()` 做跨欄位驗證？

很多 Angular 開發者第一時間可能會寫：

```ts
effect(() => {
  if (this.password() !== this.confirmPassword()) {
    this.passwordError.set('密碼不一致');
  }
});
```

這種方式不是不能做，而是它把「validation rule」拆成另一套 state management。

最後很容易變成：

```text
Form State
   ↓
Signal

Validation State
   ↓
另一個 Signal

Server Error
   ↓
第三個 Signal
```

Signal Forms 的 schema approach 則可以讓：

```text
Form Model
    ↓
Validation Schema
    ↓
Field State
    ↓
errors()
```

全部留在同一套表單模型裡。

官方文件也指出，Signal Forms validation rules 會在值變更時自動執行，錯誤透過 field state signals 暴露給 UI。

---

# 🔁 Cross-field Logic 的三個層次

可以用這張表記：

| 情境 | 建議 API | 白話理解 |
|---|---|---|
| 讀另一欄的值 | `valueOf()` | 「另一欄現在填什麼？」 |
| 讀另一欄的狀態 | `stateOf()` | 「另一欄現在 valid 嗎？」 |
| 取得另一欄 FieldTree | `fieldTreeOf()` | 「我要把錯誤指到另一欄」 |

例如：

```text
password
   │
   ├── valueOf() ─────→ 比對 confirmPassword
   │
   └── stateOf() ─────→ 判斷 password 是否 valid

email
   │
   └── fieldTreeOf() ─→ 指定 server error
```

---

# 🚀 Submit 與 Cross-field Validation

Cross-field validation 通過後，才應該進 API。

Signal Forms 的 `submit()` lifecycle 是：

```text
使用者 Submit
      ↓
interactive fields → touched
      ↓
validation
      ↓
有錯誤？
 ├─ Yes → 停止
 └─ No
      ↓
submission action
      ↓
submitting() = true
      ↓
API
      ↓
成功 / submission error
```

Angular 官方文件指出，`submit()` 會先標記 interactive fields 為 touched，再檢查 validation；驗證通過才執行 action，執行期間 `submitting()` 會是 `true`。

而 `FormRoot` 會自動設定 `novalidate`、阻止瀏覽器預設 submit，並觸發 Signal Forms 的 submit flow。

因此實務上不要再另外維護：

```ts
isSubmitting = signal(false);
```

如果狀態本來就是 Signal Forms 的 submission state。

---

# 🧪 Day 11：Cross-field Logic 測試

昨天 Day 10 我們進入 Signal Forms testing。

今天馬上把昨天的測試策略套進來。

Angular 官方目前建議：如果要驗證 validation rule、`errors()`、`valid()`、`invalid()` 或 cross-field reactive dependencies，可以優先使用 **isolated tests**；只有需要驗證 DOM、輸入事件、focus 或 accessibility 時，才需要 component-bound tests。

例如可以先測：

```ts
it('密碼不一致時應該 invalid', () => {
  const model = signal({
    password: '12345678',
    confirmPassword: '12345679',
  });

  const registerForm = form(model, (schema) => {
    validate(schema.confirmPassword, ({valueOf}) => {
      if (valueOf(schema.password) !== valueOf(schema.confirmPassword)) {
        return {
          kind: 'passwordMismatch',
          message: '兩次輸入的密碼不一致',
        };
      }

      return null;
    });
  });

  expect(registerForm.confirmPassword().invalid()).toBe(true);
});
```

再測正確情境：

```ts
it('密碼一致時應該 valid', () => {
  const model = signal({
    password: '12345678',
    confirmPassword: '12345678',
  });

  const registerForm = form(model, (schema) => {
    validate(schema.confirmPassword, ({valueOf}) => {
      if (valueOf(schema.password) !== valueOf(schema.confirmPassword)) {
        return {
          kind: 'passwordMismatch',
          message: '兩次輸入的密碼不一致',
        };
      }

      return null;
    });
  });

  expect(registerForm.confirmPassword().valid()).toBe(true);
});
```

這就是今天非常重要的觀念：

```text
Cross-field Rule
      ↓
Isolated Test
      ↓
再用 Component Test 驗證 UI
```

而不是所有表單測試都從 DOM 開始。

---

# 🅱️ Nx Monorepo 實戰

如果把這個註冊功能放進 Nx Monorepo，可以拆成：

```text
libs/
├── feature/
│   └── account-register/
│       ├── register-page.ts
│       └── register-form.ts
│
├── data-access/
│   └── account/
│       ├── account-api.ts
│       ├── account.dto.ts
│       └── account.mapper.ts
│
├── ui/
│   └── form/
│       ├── field-error/
│       └── password-field/
│
└── util/
    └── validation/
        └── password-rules.ts
```

建議責任：

```text
feature/account-register
        │
        ├── Form Model
        ├── Signal Forms Schema
        └── Page Flow
        │
        ↓
data-access/account
        │
        ├── DTO
        ├── API
        └── Mapper
        │
        ↓
Backend
```

共用 UI：

```text
ui/form/field-error
        ↓
讀取 FieldState.errors()
        ↓
所有 Feature 共用
```

### 不要這樣拆

```text
libs/shared/form
   ↓
所有 Feature 都依賴
   ↓
整個 FieldTree / Feature logic 都放進 shared
```

Shared library 應該放「真的共用」的 UI 或純 validation helper，不要把某一個 feature 的整棵表單模型變成 global dependency。

---

# 🤖 Nx 23.2 × AI Agent 實戰

截至 2026-09-29，Nx 官方 changelog 最新列出的版本是 **Nx 23.2**，發布日期為 **2026-09-02**。23.2 帶來 Oxlint / Oxfmt、較精簡的成功任務輸出、agent sandbox 支援、跨 worktree / clone / sandbox 的 cache，以及 Angular 22.1 支援。

其中對 AI coding agent 特別有價值的是「成功的 task 不再把大量 log 全部吐給 agent」。Nx 23.2 的 failure-only output 會把成功與 cache hit 壓成單行，只有失敗 task 顯示完整輸出，降低 agent 需要處理的 token 量。

實務上可以讓 Agent 遵循：

```text
修改 register feature
        ↓
檢查 Project Graph / boundary
        ↓
nx affected -t lint test build
        ↓
成功 → PR
失敗 → 只讀失敗 task
        ↓
修正
        ↓
再次 affected
```

如果 workspace 要導入 Oxlint，可以先試：

```bash
nx add @nx/oxlint
```

Nx 官方目前將 `@nx/oxlint` 標示為 experimental，且 Angular template / HTML 等規則目前仍可能需要 ESLint，因此不要一次把所有 lint 規則全部搬走。

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-29**，Angular 官方支援表顯示：

| 版本 | 狀態 | 發布日期 | LTS 結束 |
|---|---|---|---|
| Angular 22 | Active | 2026-06-03 | 2028-06 |
| Angular 21 | LTS | 2025-11-19 | 2027-06 |
| Angular 20 | LTS | 2025-05-28 | 2026-11-28 |

Angular 22 是目前最新的 stable major，Angular 21 已進入 LTS。

### Angular 21 專案今天要注意什麼？

Signal Forms 的整體能力需要 Angular 21+；但部分 API，例如 `submit()` 與 `FormRoot`，官方 API reference 已標示為 **stable since v22.0**。因此如果你的專案還在 Angular 21，不要把「官方現在的 stable 狀態」倒推成「Angular 21 每個 Signal Forms API 都同樣穩定」。

升級時也要一起檢查 Node / TypeScript 相容性。Angular 官方目前列出的 Angular 21 相容版本包含 Node `^20.19.0 || ^22.12.0 || ^24.0.0`、TypeScript `>=5.9.0 <6.0.0`；Angular 22 則需要 TypeScript `>=6.0.0 <6.1.0`，Node `^20.19.0 || ^22.12.0 || ^24.0.0`。

升級原則：

```text
Angular 21
   ↓
確認 Node / TypeScript / Nx 相容性
   ↓
先跑現有 test / lint / build
   ↓
nx migrate latest
   ↓
review migration
   ↓
逐步驗證
   ↓
Angular 22
```

Nx 23.2 的 Angular migration 也已支援 Angular 22.1，官方建議使用 `nx migrate` 管理版本升級，而不是手動一次改大量套件版本。

---

# 📚 Signal Forms 課程進度

目前課程已經從基礎一路進到跨欄位邏輯：

```text
Day 1
signal() + form()
        ↓
Day 2
Validation
        ↓
Day 3
Field State + Submit
        ↓
Day 4
Nested Object
        ↓
Day 5
Array / Dynamic Form
        ↓
Day 6
API DTO ↔ Form Model
        ↓
Day 7
httpResource
        ↓
Day 8
RxJS Interop
        ↓
Day 9
Shared UI Components
        ↓
Day 10
Testing + Feature Architecture
        ↓
Day 11 ← 今天
Cross-field Logic
        ↓
下一階段
Conditional / Async Validation
```

今天不要重新學 `required()`、`email()`，真正要掌握的是：

```ts
valueOf()
stateOf()
fieldTreeOf()
```

以及：

```ts
validate()
when
```

---

# 🎯 今日實作 Checklist

- [ ] 能說明什麼是 cross-field validation
- [ ] 使用 `valueOf()` 讀取另一個欄位
- [ ] 使用 `stateOf()` 讀取另一個欄位狀態
- [ ] 理解 `fieldTreeOf()` 的用途
- [ ] 用 `validate()` 寫密碼確認規則
- [ ] 用 `when` 寫 conditional validation
- [ ] 使用 `FormRoot` 管理 submit
- [ ] 使用 `submitting()` 防止重複送出
- [ ] 為 cross-field rule 寫 isolated test
- [ ] 把 Signal Form 放在 Nx feature library
- [ ] API / DTO 放在 data-access library
- [ ] 使用 `nx affected -t lint test build`
- [ ] 知道 Angular 21 與 Angular 22 的支援差異
- [ ] 升級前確認 Node / TypeScript / Nx compatibility

---

# 📌 今日一句話

> **跨欄位驗證不是再多寫幾個 if，而是把欄位之間的依賴正式放進 Signal Forms schema，讓 value、state、validation 與錯誤保持在同一個 reactive model。**

---

# 🔗 官方來源

- Angular Signal Forms Overview：https://angular.dev/guide/forms/signals/overview
- Angular Signal Forms Essentials：https://angular.dev/essentials/signal-forms
- Angular Cross-field Logic：https://angular.dev/guide/forms/signals/cross-field-logic
- Angular Validation：https://angular.dev/guide/forms/signals/validation
- Angular Field State：https://angular.dev/guide/forms/signals/field-state-management
- Angular Form Submission：https://angular.dev/guide/forms/signals/form-submission
- Angular Signal Forms Testing：https://angular.dev/guide/forms/signals/testing
- Angular `submit()` API：https://angular.dev/api/forms/signals/submit
- Angular `FormRoot` API：https://angular.dev/api/forms/signals/FormRoot
- Angular Versioning / Releases：https://angular.dev/reference/releases
- Angular Version Compatibility：https://angular.dev/reference/versions
- Nx Changelog：https://nx.dev/changelog
- Nx 23.2 Release：https://nx.dev/blog/nx-23-2-release
- Nx Angular Migrations：https://nx.dev/docs/technologies/angular/migrations
