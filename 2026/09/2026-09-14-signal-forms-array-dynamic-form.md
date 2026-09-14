# Signal Forms Array × Dynamic Form 實戰｜2026-09-14

> 今日主軸：Signal Forms 進入 **Array / Dynamic Form**。從昨天的 Nested Object 往下一步，學會處理「可新增、可刪除、數量不固定」的表單資料，並用 `applyEach()` 讓每個 array item 自動共用 validation schema。

## ⭐ 今日最值得看的 Angular 技術

今天最值得學的是 Angular Signal Forms 的 **Array Field + `applyEach()`**。

前幾天的學習路線已經完成：

```text
Day 1
signal() → form() → formField

Day 2
schema → required() / email() / validation

Day 3
FieldState → errors() / touched() / dirty() / pending()
         → FormRoot / submit()

Day 4
Nested Object + Form Model
```

今天進入：

```text
Array Model
   ↓
FieldTree Array
   ↓
@for render items
   ↓
新增 / 刪除 item
   ↓
applyEach()
   ↓
每個 item 共用 validation
```

Angular 官方目前的 Signal Forms 已經支援陣列欄位；`applyEach()` 可以把同一份 schema 套用到 array 中的每一個 item，而且之後動態加入的新 item 也會套用同樣規則。

這非常適合：

- 訂單明細
- 聯絡電話清單
- Email 收件人清單
- 家庭成員
- 技能 / 標籤
- 動態地址
- 商品規格
- 問卷重複題目

官方目前也將 Signal Forms 的核心 API 標示為 Angular 22 stable；Signal Forms 本身要求 Angular v21 以上。詳見官方文件：[Signal Forms Overview](https://angular.dev/guide/forms/signals/overview)。

---

## 🔍 核心概念 / API

### 1. Array 直接存在 Form Model 裡

先從最簡單的 array 開始：

```ts
interface ContactFormModel {
  name: string;
  phoneNumbers: string[];
}

contactModel = signal<ContactFormModel>({
  name: '',
  phoneNumbers: [''],
});
```

建立 form：

```ts
contactForm = form(this.contactModel);
```

Signal Forms 會依照 model shape 建立 FieldTree：

```text
contactForm
├── name
└── phoneNumbers
    ├── [0]
    ├── [1]
    └── [2]
```

Angular 官方的 Form Model 文件也特別強調：form model 是 Signal Forms 的 single source of truth，而 objects / arrays 是 FieldTree 的 structural layer；因此表單 model 最好保持為 plain JavaScript objects / arrays，而不要直接把 `Map`、`Set` 或 class instance 當成結構層。

參考：[Form models](https://angular.dev/guide/forms/signals/models)。

---

### 2. `@for` 顯示動態欄位

Template 可以直接依 array field render：

```html
@for (phone of contactForm.phoneNumbers; track $index; let i = $index) {
  <input [formField]="phone" />
  <button type="button" (click)="removePhone(i)">
    刪除
  </button>
}
```

這裡一個很重要的觀念是：**不要為每一個可能的欄位預先建立一堆 signal。**

不要這樣：

```ts
phone1 = signal('');
phone2 = signal('');
phone3 = signal('');
```

而是讓 model 表達真實資料：

```ts
phoneNumbers: string[];
```

然後讓 FieldTree 隨著 model 的 array 長度一起變化。

---

### 3. 新增 / 刪除 item：修改 model

Signal Forms 的 form 會使用 model signal 作為資料來源，所以動態新增資料時，可以直接更新 model：

```ts
addPhone() {
  this.contactModel.update(model => ({
    ...model,
    phoneNumbers: [...model.phoneNumbers, ''],
  }));
}
```

刪除：

```ts
removePhone(index: number) {
  this.contactModel.update(model => ({
    ...model,
    phoneNumbers: model.phoneNumbers.filter((_, i) => i !== index),
  }));
}
```

這種寫法的好處是資料結構非常直覺：

```text
User action
   ↓
update model signal
   ↓
FieldTree 重新反映 array
   ↓
Template 更新
```

Angular 官方 Dynamic Forms 文件也採用「更新 model signal 讓 array 增減」的方向；新加入的 item 會取得新的 field state。

參考：[Dynamic forms with JSON](https://angular.dev/guide/forms/signals/dynamic-forms-with-json)。

---

### 4. `applyEach()` 是今天最重要的 API

如果 array 是：

```ts
items: {
  name: string;
  quantity: number;
}[];
```

我們通常希望每一筆都有：

```text
name → required
quantity → >= 1
```

不用自己在 `@for` 裡判斷，而是在 schema 裡使用：

```ts
applyEach(schemaPath.items, (item) => {
  required(item.name);
  min(item.quantity, 1);
});
```

官方定義的 `applyEach()` 就是「把 schema 套用到 array 的每個 item」。而且不是只套用建立當下存在的資料；後續新增的 item 也會套用同樣規則。

參考：[Angular `applyEach()` API](https://angular.dev/api/forms/signals/applyEach)。

---

### 5. `schema()` 可以把 item validation 抽成可重用 schema

當 item 規則變多時，不建議全部塞在 `form()` 裡：

```ts
const lineItemSchema = schema<LineItem>((item) => {
  required(item.name);
  min(item.quantity, 1);
});
```

再：

```ts
orderForm = form(this.orderModel, (path) => {
  applyEach(path.items, lineItemSchema);
});
```

這會讓 Nx Monorepo 裡的 validation logic 更容易重用。

Angular 官方 Schema 文件也建議：如果相同資料 shape 的 rules 會被多個 form 使用，可以用 `schema()` 建立 reusable schema，再搭配 `apply()` 或 `applyEach()`。

參考：[Schemas and schema composability](https://angular.dev/guide/forms/signals/schemas)。

---

## 💻 可以直接實作的完整範例

下面做一個很常見的「訂單明細」表單。

需求：

1. 訂單名稱必填
2. 至少有一筆商品
3. 每個商品名稱必填
4. 數量至少 1
5. 可以新增商品
6. 可以刪除商品
7. 新增商品後自動套用 validation
8. 使用 Signal Forms 管理整個 model

### Component

```ts
import { Component, signal } from '@angular/core';
import {
  applyEach,
  FormField,
  form,
  min,
  required,
  schema,
} from '@angular/forms/signals';

interface LineItem {
  name: string;
  quantity: number;
}

interface OrderFormModel {
  title: string;
  items: LineItem[];
}

const lineItemSchema = schema<LineItem>((item) => {
  required(item.name, {
    message: '請輸入商品名稱',
  });

  min(item.quantity, 1, {
    message: '數量至少為 1',
  });
});

@Component({
  selector: 'app-order-form',
  imports: [FormField],
  template: `
    <form (submit)="$event.preventDefault()">
      <h2>建立訂單</h2>

      <label>
        訂單名稱
        <input [formField]="orderForm.title" />
      </label>

      @if (orderForm.title().touched() && orderForm.title().invalid()) {
        @for (error of orderForm.title().errors(); track error.kind) {
          <p>{{ error.message }}</p>
        }
      }

      <h3>商品明細</h3>

      @for (
        item of orderForm.items;
        track $index;
        let i = $index
      ) {
        <fieldset>
          <legend>商品 {{ i + 1 }}</legend>

          <label>
            商品名稱
            <input [formField]="item.name" />
          </label>

          @if (item.name().touched() && item.name().invalid()) {
            @for (error of item.name().errors(); track error.kind) {
              <p>{{ error.message }}</p>
            }
          }

          <label>
            數量
            <input
              type="number"
              [formField]="item.quantity"
            />
          </label>

          @if (item.quantity().touched() && item.quantity().invalid()) {
            @for (error of item.quantity().errors(); track error.kind) {
              <p>{{ error.message }}</p>
            }
          }

          <button
            type="button"
            (click)="removeItem(i)"
            [disabled]="orderModel().items.length === 1"
          >
            刪除商品
          </button>
        </fieldset>
      }

      <button type="button" (click)="addItem()">
        + 新增商品
      </button>

      <hr />

      <pre>{{ orderModel() | json }}</pre>
    </form>
  `,
})
export class OrderFormComponent {
  orderModel = signal<OrderFormModel>({
    title: '',
    items: [
      {
        name: '',
        quantity: 1,
      },
    ],
  });

  orderForm = form(this.orderModel, (path) => {
    required(path.title, {
      message: '請輸入訂單名稱',
    });

    applyEach(path.items, lineItemSchema);
  });

  addItem() {
    this.orderModel.update((model) => ({
      ...model,
      items: [
        ...model.items,
        {
          name: '',
          quantity: 1,
        },
      ],
    }));
  }

  removeItem(index: number) {
    this.orderModel.update((model) => ({
      ...model,
      items: model.items.filter((_, i) => i !== index),
    }));
  }
}
```

> 範例中的 `json` pipe 請依你的 Angular standalone component 設定匯入 `JsonPipe`。這裡的重點是 `FormField`、array FieldTree、`applyEach()` 與 model update。

---

## 🧠 這段程式最值得理解的地方

### 新增 item 不需要重新建立 Form

很多人第一次看到 Dynamic Form，會直覺寫：

```ts
forms = signal([]);
```

每次新增後再重建整個 form。

Signal Forms 的思考方式比較簡單：

```text
orderModel.items
       ↓
orderForm.items
       ↓
@for
```

所以：

```ts
this.orderModel.update(...)
```

就是核心操作。

`form()` 產生的 FieldTree 會反映 model 的 array 結構。

---

### `applyEach()` 解決「每筆資料都要相同規則」

如果沒有 `applyEach()`，很容易把 validation 寫進 component method：

```ts
validateItem(index: number) {
  // 手動判斷第 index 筆
}
```

這會讓 validation 跟 UI 操作耦合。

使用：

```ts
applyEach(path.items, lineItemSchema);
```

就變成：

```text
LineItem Schema
      ↓
applyEach()
      ↓
item[0]
item[1]
item[2]
item[3]
...
```

這是今天最重要的設計觀念。

---

# 🆕 Signal Forms 持續課程 — Day 5

## Day 5：Array / Dynamic Form

今天正式進入原本課程規劃的：

```text
Day 5
Array / Dynamic Form
```

### 今日一定要會

#### ① Array 放進 Form Model

```ts
interface FormModel {
  items: Item[];
}
```

#### ② `form()` 直接建立 FieldTree

```ts
formModel = form(this.model);
```

#### ③ `@for` 顯示 array field

```html
@for (item of formModel.items; track $index) {
  <input [formField]="item.name" />
}
```

#### ④ 更新 model 新增 item

```ts
model.update(value => ({
  ...value,
  items: [...value.items, createItem()],
}));
```

#### ⑤ 更新 model 刪除 item

```ts
model.update(value => ({
  ...value,
  items: value.items.filter((_, index) => index !== targetIndex),
}));
```

#### ⑥ 用 `applyEach()` 套用 item schema

```ts
applyEach(path.items, itemSchema);
```

---

## 🧩 Array Item Identity：不要只把它想成 index

Dynamic Form 還有一個很值得注意的觀念：**array item 的 identity 不完全等於 index。**

例如：

```ts
[
  { id: 101, name: 'Apple' },
  { id: 102, name: 'Banana' },
]
```

如果資料排序後變成：

```ts
[
  { id: 102, name: 'Banana' },
  { id: 101, name: 'Apple' },
]
```

UI 上的 index 改變了，但 item 本身仍然是同一筆資料。

Angular Signal Forms 有針對 array item identity 的處理；官方文件指出，對 object array 而言，FieldTree 對 item 的追蹤不是單純依賴 position，因此重新排序時，對應的 field state 可以跟著資料 item 移動。

參考：[Dynamic forms with JSON](https://angular.dev/guide/forms/signals/dynamic-forms-with-json)。

這也是為什麼實務上不要把「第 2 筆」當成資料本身的 identity。

---

# 🅱️ Nx Monorepo 實戰

Dynamic Form 很適合拆成 Nx 的三層：

```text
apps/
└── web/

libs/
├── feature/
│   └── order/
│       └── create-order/
│
├── data-access/
│   └── order/
│       ├── order-api.ts
│       └── order.models.ts
│
└── ui/
    └── form/
        ├── field-error/
        └── dynamic-list/
```

推薦依賴方向：

```text
feature/create-order
        ↓
 data-access/order
        ↓
       API

feature/create-order
        ↓
      ui/form
```

### Feature Library

負責：

- `OrderFormModel`
- Signal Form schema
- add / remove item
- submit flow
- 使用者互動

### Data Access Library

負責：

- `CreateOrderDto`
- API service
- DTO mapping
- API error handling

### UI Library

負責：

- Field error 顯示
- Loading button
- 可重用的表單外觀
- 通用 dynamic list UI

這樣可以避免把 API contract、feature state、UI presentation 全部塞在同一個 component。

---

## ⚡ Nx 23：今天值得注意的新變化

截至 **2026-09-14**，Nx 官方目前列出的 Current major 是 **Nx 23**，而 Nx 23.2 於 **2026-09-02** 發布。

Nx 23 的重點包含：

- Agentic migrations
- 更快的 Nx Agents
- Task sandboxing
- Native TypeScript loading
- 更細緻的 target configuration
- 移除長期 deprecated APIs

參考：[Nx Changelog](https://nx.dev/changelog) 與 [Nx 23 release](https://nx.dev/blog/nx-23-release)。

對 Angular Monorepo 而言，今天最值得記住的是：**升級 Nx 不只是升級 CLI；要確認 `nx` 與所有 `@nx/*` 套件保持相同版本線。**

Nx 官方也明確建議 `nx` 與 `@nx` packages 使用 matching versions。

---

## 🔍 `nx affected` + Dynamic Form

假設今天只修改：

```text
libs/feature/order/create-order
```

CI 可以使用：

```bash
nx affected -t lint test build
```

先看受影響的 projects：

```bash
nx affected:graph
```

或依目前 Nx CLI 工作流查看 affected graph。

核心概念：

```text
Git diff
   ↓
Project Graph
   ↓
Affected Projects
   ↓
lint / test / build
```

這對大型 Angular Nx Monorepo 很重要，因為 Dynamic Form 的修改通常只影響某個 feature library，不需要每一次都把所有 app / lib 全部重新驗證。

---

# 🆚 Angular 21 與最新 Angular 的差異 / 升級注意事項

截至 **2026-09-14**，Angular 官方版本支援資料顯示：

| 版本 | 狀態 | 發布日期 | LTS 結束 |
|---|---|---|---|
| Angular 22 | Active | 2026-06-01 | 2027-11-?? |
| Angular 21 | LTS | 2025-11-19 | 2027-05-19 |

Angular 官方支援策略是 major 通常支援 18 個月；Angular 21 的 Active 階段已結束，現在進入 LTS。

官方版本資料：[Angular Releases](https://angular.dev/reference/releases)。

### Signal Forms 的差異特別重要

Angular 21：

```text
Signal Forms
↓
可使用
```

Angular 22：

```text
Signal Forms
↓
核心 API stable
```

目前官方 API 文件已將 `form()`、`applyEach()`、`FormRoot`、`Schema` 等 API 標示為 **stable since v22.0**。

例如：

- [`form()`](https://angular.dev/api/forms/signals/form)
- [`applyEach()`](https://angular.dev/api/forms/signals/applyEach)
- [`FormRoot`](https://angular.dev/api/forms/signals/FormRoot)
- [`Schema`](https://angular.dev/api/forms/signals/Schema)

所以如果你的公司目前使用 Angular 21，不代表不能學 Signal Forms；但在升級到 Angular 22 時，建議重新檢查 Signal Forms API 的 stable 狀態、型別與 migration。

---

## 🧭 Nx × Angular 升級建議

Nx 官方目前的 compatibility matrix 顯示：

```text
Angular 22.1.x → Nx >= 23.2.0
Angular 22.0.x → Nx >= 23.1.0
Angular 21.2.x → Nx >= 22.6.0
Angular 21.1.x → Nx >= 22.4.0
Angular 21.0.x → Nx >= 22.3.0
```

因此不要直接做：

```bash
npm install @angular/core@latest
```

然後期待 Nx 自己全部配好。

比較安全的流程是：

```text
確認 Angular version
        ↓
確認 Nx compatibility matrix
        ↓
確認 Node version
        ↓
nx migrate
        ↓
執行 migrations
        ↓
lint / test / build
        ↓
確認 affected projects
```

Nx 官方也建議使用 `nx migrate` 來升級 workspace。

參考：[Nx and Angular Versions](https://nx.dev/docs/kb/angular-nx-version-matrix)。

---

# 🤖 AI Agent × Dynamic Form 實戰流程

如果你使用 Cursor / Claude Code 類型的 coding agent，Dynamic Form 任務可以要求 Agent 按以下順序工作：

```text
1. 讀取 feature library
2. 找到 Form Model
3. 找到 API DTO
4. 確認 array item type
5. 建立 item schema
6. 使用 applyEach()
7. 實作 add / remove
8. 更新 template @for
9. 執行 nx affected -t lint test build
10. 檢查 cache / graph 結果
```

這比直接對 Agent 說：

```text
「幫我做一個動態表單」
```

更可靠。

因為你把：

```text
Domain Model
+ Form Model
+ Schema
+ UI
+ Nx dependency boundary
+ Verification
```

都明確化了。

---

# 🎯 今日實作 Checklist

### Angular / Signal Forms

- [ ] 能用 `signal<T>()` 建立含 array 的 Form Model
- [ ] 能用 `form()` 建立 array FieldTree
- [ ] 能用 `@for` render dynamic fields
- [ ] 能新增 array item
- [ ] 能刪除 array item
- [ ] 能用 `applyEach()` 套用共同 validation
- [ ] 知道什麼時候使用 `schema()` 抽出 reusable schema
- [ ] 知道 array item identity 不應只等同 index

### Nx

- [ ] Feature / Data Access / UI 分層
- [ ] Form Model 不直接污染 API DTO
- [ ] Dynamic Form validation 放在 feature/schema 層
- [ ] API mapping 放 data-access
- [ ] 執行 `nx affected -t lint test build`
- [ ] 確認 Nx 與 `@nx/*` 版本一致

### 升級

- [ ] 確認 Angular 21 / 22 版本
- [ ] 查 Nx Angular compatibility matrix
- [ ] 確認 Node.js 支援版本
- [ ] 升級時優先使用 `nx migrate`

---

# 📌 今日一句話

> **Dynamic Form 不需要把表單變複雜：讓 Array 成為 Model，讓 `applyEach()` 管理每個 item 的規則，Signal Forms 就能把資料、validation 與 UI 維持在同一棵 FieldTree 上。**

---

# 🔗 官方來源

### Angular

- [Angular Signal Forms Overview](https://angular.dev/guide/forms/signals/overview)
- [Angular Signal Forms Essentials](https://angular.dev/essentials/signal-forms)
- [Form Models](https://angular.dev/guide/forms/signals/models)
- [Schemas and schema composability](https://angular.dev/guide/forms/signals/schemas)
- [Validation](https://angular.dev/guide/forms/signals/validation)
- [Dynamic Forms with JSON](https://angular.dev/guide/forms/signals/dynamic-forms-with-json)
- [`applyEach()` API](https://angular.dev/api/forms/signals/applyEach)
- [`form()` API](https://angular.dev/api/forms/signals/form)
- [Angular Releases](https://angular.dev/reference/releases)
- [Angular Version Compatibility](https://angular.dev/reference/versions)

### Nx

- [Nx Changelog](https://nx.dev/changelog)
- [Nx 23 Release](https://nx.dev/blog/nx-23-release)
- [Nx and Angular Versions](https://nx.dev/docs/kb/angular-nx-version-matrix)
- [Nx Release Schedule](https://nx.dev/docs/reference/releases)
- [Nx with Node.js](https://nx.dev/docs/technologies/node/introduction)
- [Nx Migrations](https://nx.dev/docs/technologies/angular/migrations)

---

**文章日期：2026-09-14**  
**Signal Forms 課程：Day 5 — Array / Dynamic Form**  
**主軸：Signal Forms Array + `applyEach()` + Nx Monorepo Architecture**
