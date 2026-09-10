# Angular SSR URL Parsing 安全更新｜2026-09-10

> 今日主軸：Angular SSR 的 URL 解析、安全檢查、NG05703，以及前端工程師在 SSR / proxy 環境需要注意的 URL 信任邊界。

## ⭐ 今日最值得看的 Angular 技術

Angular SSR 不只是把 Angular App 放到 Node.js 上執行。當伺服器需要把相對 URL 解析成絕對 URL 時，URL parsing 本身就會成為安全邊界。

Angular 目前在 SSR 環境提供 URL origin validation。當一個看起來像相對路徑的 URL 最後被解析成不同 origin 時，Angular 會以 `NG05703` 阻擋它，避免 URL parser 差異造成 SSRF 或安全繞過。

這件事情對前端工程師很重要，因為問題往往不是出在 Angular component，而是：

```text
Browser / Client
      ↓
Proxy / Load Balancer
      ↓
SSR Server
      ↓
URL parsing
      ↓
HTTP request / navigation
```

只要不同層對 URL、host、forwarded headers 的理解不一致，就可能產生安全問題。

---

## 🔍 NG05703 是什麼？

Angular 官方的錯誤碼：

```text
NG05703
Suspicious URL Origin Change
```

它發生在 SSR 環境中：Angular 嘗試解析 URL 時，發現結果的 origin 與預期不同。

簡化理解：

```text
原本以為：
/path/to/resource

解析後卻變成：
https://unexpected-origin/...
```

Angular 會阻擋這種 unexpected origin change。

### 為什麼需要這個檢查？

因為某些 URL 寫法在不同 parser / browser / server 環境可能被解讀成不同結果。

例如 Angular 官方特別提到 backslash-prefixed URL 的情況。某些以 `/\\` 或 `\\\\` 開頭的輸入，在特定 URL parsing 情境下可能產生非預期的 host / origin 解讀。

重點不是記住某一個攻擊字串，而是理解：

> **SSR 的 URL parsing 不能只假設「使用者給的是 path，所以一定安全」。**

---

# 🧠 Frontend 工程師要注意的 URL 信任邊界

在純 SPA 中，很多 URL 問題最後只影響 browser navigation。

但 SSR 不同：

```text
Request
  ↓
SSR
  ↓
Server-side URL resolution
  ↓
Server-side HTTP request
```

因此 URL 可能在 server 端被使用。

這也是 SSRF 防護的重要原因之一。

Angular 官方安全文件也針對 SSR request handling 提供 forwarded header validation，例如：

- `Host`
- `Forwarded`
- `X-Forwarded-Host`
- `X-Forwarded-Proto`
- `X-Forwarded-Prefix`
- `X-Forwarded-Port`

這些 header 在 proxy / load balancer 架構中特別值得注意。

---

## 💻 實務上怎麼排查？

如果 Angular SSR 升級後開始看到：

```text
NG05703: Suspicious URL Origin Change
```

建議依照下面順序排查。

### 1. 找出發生錯誤的 URL

先確認是哪一個 URL 被 Angular 判定為 origin change。

```text
Request URL
↓
SSR route / resource
↓
Resolved absolute URL
↓
Actual origin
```

不要直接把 validation 關掉來「讓錯誤消失」。

### 2. 檢查 proxy headers

如果架構是：

```text
Cloud / LB
  ↓
Nginx
  ↓
Node SSR
```

確認 forwarded headers 是否真的代表外部 request 的資訊。

尤其是：

```text
Host
X-Forwarded-Host
X-Forwarded-Proto
X-Forwarded-Port
```

### 3. 檢查 URL 是否真的應該是 relative URL

例如 application code 中：

```ts
fetch(url)
```

不要只看變數名稱叫 `url` 就假設它永遠是安全 relative path。

可以先明確區分：

```ts
const apiPath = '/api/users';
```

與：

```ts
const apiOrigin = 'https://api.example.com';
```

讓資料型態與責任更清楚。

---

# 🧩 Angular SSR + API URL 的架構建議

企業 Angular 專案常見：

```text
Angular Browser
       ↓
Angular SSR
       ↓
API Gateway
       ↓
Backend API
```

建議不要讓 component 自己組 URL：

```ts
// 不推薦散落在 component
const url = host + '/api/user/' + id;
```

可以集中在 data-access / API service：

```text
libs/
├── feature/
│   └── user/
├── data-access/
│   └── user/
│       ├── user.api.ts
│       └── user.models.ts
└── util/
    └── url/
```

如此一來 SSR、Browser、proxy 的 URL 行為比較容易統一管理。

---

# 🅱️ Nx Monorepo 實戰：把 SSR 邊界放對地方

如果你的團隊使用 Nx Monorepo，可以把 SSR / API / URL 相關責任拆開：

```text
apps/
  eip-web/

libs/
  feature/
    user/
  data-access/
    user/
  util/
    url/
  server/
    request-context/
```

概念上：

```text
Feature
  ↓
Data Access
  ↓
URL / API configuration
  ↓
SSR request context
```

這比讓每個 feature 自己讀取 `Host`、自己判斷 `X-Forwarded-*` 更容易維護。

同時也可以利用 Nx Project Graph 檢查依賴方向，避免 browser-only library 被 server code 錯誤引用。

---

# 🧪 驗收 Checklist

如果近期有 Angular SSR / Angular 版本升級，可以把下面項目加入驗收：

- [ ] SSR request 可以正常處理一般 relative URL
- [ ] 非預期 origin change 會被阻擋
- [ ] proxy / load balancer 的 Host 設定正確
- [ ] `X-Forwarded-Host` / `X-Forwarded-Proto` 行為符合部署架構
- [ ] API URL 沒有散落在 component
- [ ] SSR 與 browser 使用一致的 API configuration
- [ ] Nx dependency graph 沒有 browser/server 邊界錯置
- [ ] Angular 升級後確認 SSR integration tests

---

# 🔄 Angular 21 ↔ 最新 Angular

這類 SSR 安全更新值得放進升版驗收，而不是只看 TypeScript compile 是否成功。

建議升版時至少分成：

```text
Compile
  ↓
Unit Test
  ↓
SSR Build
  ↓
SSR Runtime
  ↓
Proxy / Production-like Environment
  ↓
Security Regression Test
```

尤其企業系統通常有 CDN、Reverse Proxy、API Gateway，多一層 infrastructure 就多一層 URL / header interpretation。

---

# 🎯 今日實作 Checklist

- [ ] 在自己的 SSR 專案搜尋 URL 組裝邏輯
- [ ] 確認哪些 URL 會在 server side 被解析
- [ ] 檢查 proxy 的 Host / Forwarded headers
- [ ] 搜尋是否有 component 自己組 API URL
- [ ] 把 API URL 集中到 data-access / configuration
- [ ] 升版後加入 SSR runtime 驗收，而不是只做 build

---

# 📌 今日一句話

> **SSR 的 URL 不是單純字串問題；它會跨越 Browser、Proxy、Node 與 API，因此 URL parsing 本身就是安全邊界。**

---

# 🔗 官方來源

- Angular NG05703：https://angular.dev/errors/NG05703
- Angular Security：https://angular.dev/best-practices/security
- Angular SSR：https://angular.dev/guide/ssr
- Angular 版本與支援週期：https://angular.dev/reference/releases
- Nx Project Graph：https://nx.dev/docs/features/explore-graph
