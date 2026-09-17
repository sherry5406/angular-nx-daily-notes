# Signal Forms × RxJS：建立清楚的 Reactive Data Flow｜2026-09-17

> 今日主軸：Signal Forms 管理表單狀態，RxJS 處理 Observable 串流；透過 Angular RxJS interop 建立 Signal ↔ Observable 邊界。

## ⭐ 今日最值得看的 Angular 技術

今天進入 Signal Forms Day 8：**Signal Forms + RxJS**。

Signal Forms 適合處理 form model、FieldTree、validation、touched、dirty 與 submission；RxJS 適合 debounce、取消舊請求、組合事件流，以及既有 Observable 型態的 data-access service。Angular 官方提供 `toSignal()` 與 `toObservable()` 作為兩者的橋接。`toSignal()` 會訂閱 Observable 並把最新值暴露成 Signal；`toObservable()` 則把 Signal 的值轉成 Observable。citeturn1search11

Signal Forms 目前的官方 guide 要求 Angular 21+；Angular API 頁面則把 `form()`、`FormRoot` 等 API 標為 v22 起 stable。citeturn1search4turn1search8turn0search18

---

## 🔍 核心概念 / API

### `toSignal()`：Observable → Signal

```ts
products = toSignal(this.productApi.getProducts(), {
  initialValue: [],
});
```

適合把既有 RxJS service 的結果交給 signal-driven UI。

### `toObservable()`：Signal → Observable

```ts
keyword$ = toObservable(
  computed(() => this.searchModel().keyword.trim()),
);
```

之後就可以使用 RxJS：

```ts
keyword$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap((keyword) => keyword
    ? this.productApi.searchProducts(keyword)
    : of([])),
);
```

這個 pipeline 很適合搜尋框：使用者快速輸入時，`debounceTime()` 減少請求，`distinctUntilChanged()` 忽略相同值，`switchMap()` 讓最新搜尋取代舊搜尋。

### 不要建立兩份 source of truth

不推薦同時維護：

```ts
keyword = signal('');
keywordSubject = new BehaviorSubject('');
```

應讓 Signal Forms model 作為表單資料的主要來源，再透過 `toObservable()` 接入需要 RxJS 的地方。Angular 官方也將 form model 定義為表單資料的 single source of truth。citeturn0search13

---

# 💻 可以直接實作的完整範例

## Component

```ts
import { Component, computed, signal } from '@angular/core';
import { form, FormField, maxLength } from '@angular/forms/signals';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, of, switchMap } from 'rxjs';

interface Product {
  id: number;
  name: string;
}

interface SearchModel {
  keyword: string;
}

@Component({
  selector: 'app-product-search',
  imports: [FormField],
  template: `
    <label>
      搜尋產品
      <input [formField]="searchForm.keyword" />
    </label>

    @if (products().length === 0) {
      <p>沒有搜尋結果</p>
    } @else {
      <ul>
        @for (product of products(); track product.id) {
          <li>{{ product.name }}</li>
        }
      </ul>
    }
  `,
})
export class ProductSearchComponent {
  searchModel = signal<SearchModel>({ keyword: '' });

  searchForm = form(this.searchModel, (path) => {
    maxLength(path.keyword, 50);
  });

  keyword$ = toObservable(
    computed(() => this.searchModel().keyword.trim()),
  );

  searchResult$ = this.keyword$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((keyword) =>
      keyword
        ? this.productApi.searchProducts(keyword)
        : of([] as Product[]),
    ),
  );

  products = toSignal(this.searchResult$, {
    initialValue: [] as Product[],
  });

  constructor(private readonly productApi: ProductApi) {}
}
```

### 資料流

```text
User input
   ↓
[formField]
   ↓
Signal Forms model
   ↓
computed()
   ↓
toObservable()
   ↓
debounceTime()
   ↓
distinctUntilChanged()
   ↓
switchMap()
   ↓
API Observable
   ↓
toSignal()
   ↓
products()
   ↓
Template
```

Angular 官方的 Signal Forms model 會與 `[formField]` 自動同步；`form()` 建立的 FieldTree 會鏡像 model 結構。citeturn0search13turn1search3

---

# 🧠 Signal Forms 課程 Day 8

目前課程進度：

1. `signal()` + `form()` 基礎
2. FormField / Form Model / FieldTree
3. Field State / Submit lifecycle
4. Nested Object
5. Array / Dynamic Form
6. API DTO ↔ Form Model
7. Signal Forms + `httpResource`
8. **Signal Forms + RxJS ← 今天**

今天要記住：

```text
Signal Forms = 表單狀態
RxJS         = 非同步事件流
Interop      = 兩者之間的橋
```

不要為了 Signals 而重寫已經穩定的 Observable data-access。先建立 boundary，再逐步現代化。

---

# 🧪 測試策略

Signal Forms schema 可以做 isolated test，驗證 `errors()`、`valid()`、`invalid()`、disabled/readonly 與 cross-field rules；RxJS pipeline 則獨立驗證 debounce、distinct 與 switchMap 行為。Angular 官方 testing guide 也建議表單邏輯不需要一定透過 DOM 測試。citeturn0search21

推薦分層：

```text
Form schema
   ↓
isolated test

RxJS pipeline
   ↓
stream/service test

DOM binding
   ↓
component test
```

---

# 🅱️ Nx Monorepo 實戰

建議結構：

```text
libs/
├── feature/product-search/
├── data-access/products/
├── ui/product-list/
└── util/product-types/
```

責任分工：

```text
feature
  ↓
Signal Form model
  ↓
toObservable()
  ↓
data-access
  ↓
Observable API
  ↓
toSignal()
  ↓
ui
```

這樣可以保留既有 RxJS service 的可重用性，同時讓新 Feature 使用 Signal-driven UI。

Nx `affected` 會依 Git 變更與 Project Graph 找出受影響的 projects，只執行需要的 tasks；CI 常見用法是：

```bash
pnpm nx affected -t lint test build
```

citeturn1search0

---

# 🤖 Nx × AI Agent 實戰

清楚的 library boundary 也有利於 coding agent 工作：

```text
Agent 修改 feature
       ↓
Project Graph
       ↓
Affected projects
       ↓
lint / test / build
```

Nx 23.2 官方更新包含更精簡、token-friendly 的 CLI 輸出，以及對 agent sandbox 的改善；`nx configure-ai-agents` 可協助設定 agent 執行 Nx 所需的環境。citeturn1search1

---

# 🔄 Angular 21 ↔ 最新 Angular

截至 **2026-09-17**，Angular 官方支援資訊：

| 版本 | 狀態 | Released | LTS 結束 |
|---|---|---|---|
| Angular 22 | Active | 2026-06-03 | 2028-06 |
| Angular 21 | LTS | 2025-11-19 | 2027-06 |
| Angular 20 | LTS | 2025-05-28 | 2026-11-28 |

Angular 22 是目前 Active major，Angular 21 是 LTS。Angular 官方 release schedule 列 Angular 22.2 約於 2026 年 9 月推出，Angular 23 預計 2027 年 6 月。citeturn0search19

對 Angular 21 專案：

```text
Angular 21 LTS
   ↓
先以新 Feature 小範圍導入 Signal Forms
   ↓
確認團隊測試與 shared UI 策略
   ↓
再規劃 Angular 22 migration
```

不要把 Angular major upgrade、Reactive Forms 全面重寫、Nx 架構重構一次綁在同一個 migration。

---

# 📦 Nx 23.2 今日注意事項

Nx 23.2 官方 release 特別值得 Angular 團隊注意：

- 支援 Angular 22.1
- 改善 Angular Rspack
- 更精簡 CI / agent output
- cache 可跨 sandbox、worktree 與 repository clone
- `nx configure-ai-agents`
- 可以更方便執行單一 migration
- Vitest generator 支援改善

Nx 官方也要求 `nx` 與 `@nx/*` 套件維持相同版本，升級應優先使用 `nx migrate`。citeturn1search1turn1search9

---

# 🎯 今日實作 Checklist

- [ ] 建立 Signal Form model
- [ ] 使用 `[formField]`
- [ ] 使用 `toObservable()`
- [ ] 加入 `debounceTime()`
- [ ] 加入 `distinctUntilChanged()`
- [ ] 使用 `switchMap()`
- [ ] 使用 `toSignal()` 回到 Signal UI
- [ ] 維持單一 source of truth
- [ ] API service 放 Nx `data-access`
- [ ] 搜尋頁放 Nx `feature`
- [ ] 純展示元件放 Nx `ui`
- [ ] 分別測試 form schema 與 RxJS pipeline
- [ ] 執行 `pnpm nx affected -t lint test build`
- [ ] 確認 Angular 21 / 22 的 Signal Forms 採用策略

---

# 📌 今日一句話

> **Signals 描述目前狀態，RxJS 描述事件如何流動；Signal Forms × RxJS 的重點不是二選一，而是用 interop API 把兩種模型接在正確的 boundary。**

---

# 🔗 官方來源

- Angular Signal Forms：https://angular.dev/guide/forms/signals/overview
- Angular Signal Forms Essentials：https://angular.dev/essentials/signal-forms
- Angular Form Models：https://angular.dev/guide/forms/signals/models
- Angular Signal Forms Testing：https://angular.dev/guide/forms/signals/testing
- Angular RxJS Interop：https://angular.dev/ecosystem/rxjs-interop
- Angular `form()` API：https://angular.dev/api/forms/signals/form
- Angular Releases：https://angular.dev/reference/releases
- Nx 23.2 Release：https://nx.dev/blog/nx-23-2-release
- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Release Schedule：https://nx.dev/docs/reference/releases
