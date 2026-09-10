# Nx Remote Cache × Affected CI 實戰｜2026-09-09

> 今日主軸：Nx Monorepo 的 Affected、Project Graph、Remote Cache，以及如何把它們組合成真正有效的 CI 優化策略。

## ⭐ 今日最值得看的技術

在 Nx Monorepo 裡，CI 變慢通常不是單一指令太慢，而是「做了太多不需要做的工作」。Nx 的解法可以拆成三層：

```text
1. Affected       → 少跑不受影響的專案
2. Remote Cache   → 已經算過的結果不要重算
3. Distribution   → 剩下的工作平行分散到多台機器
```

Nx 官方目前也把 Affected 與 Remote Caching、Distributed Task Execution 視為互補能力：Affected 先縮小工作範圍，Cache 再避免重複計算。

---

## 🔍 核心概念：Affected 到底在做什麼？

假設 Monorepo：

```text
apps/
  eip-web
  admin-web

libs/
  ui
  user-data-access
  order-data-access
```

今天只修改：

```text
libs/user-data-access
```

如果直接：

```bash
nx run-many -t test
```

可能會把所有 project 都拿來測試。

而：

```bash
nx affected -t test
```

會利用 Git 變更與 Nx Project Graph 找出受影響的 projects，再只執行必要的 task。

```text
Git changed files
       ↓
Nx Project Graph
       ↓
找到直接受影響的 project
       ↓
沿 dependency graph 找 dependent projects
       ↓
只執行 affected tasks
```

查看 affected graph：

```bash
nx graph --affected
```

---

## 🧠 Project Graph 為什麼重要？

Nx 不只是管理資料夾，而是理解 workspace 中 project 之間的 dependency relationship。

例如：

```text
user-data-access
       ↓
user-feature
       ↓
eip-web
```

如果修改 `user-data-access`，Nx 可以沿著 graph 判斷後面的 project 也可能受到影響。

這也是 Monorepo 使用 Nx 後最值得建立的觀念：

> **Project Graph 是 CI 智慧化的基礎。**

---

# 💻 可以直接實作的 CI 範例

GitHub Actions 中可以採用：

```yaml
- uses: actions/checkout@v7
  with:
    fetch-depth: 0

- run: npx nx affected -t lint test build
```

`fetch-depth: 0` 很重要，因為 Nx 的 affected 計算需要 Git history。

CI 可以透過 `NX_BASE` / `NX_HEAD` 指定比較範圍，例如：

```bash
NX_BASE=origin/main
NX_HEAD=$PR_BRANCH_NAME
```

實際設定仍應配合你的 CI trigger 與 branch strategy。

---

# 🚀 Remote Cache：為什麼 Affected 還不夠？

假設一個 PR 影響 10 個 projects。

```text
nx affected -t test
        ↓
10 projects 要跑
```

如果工程師又修改一個檔案再 push：

```text
10 projects
↓
再次執行
```

即使這些 task 的實際 inputs 沒有改變，也可能重新計算。

Remote Cache 的價值就在這裡。

Nx 會根據 task inputs 計算 hash：

```text
Source files
+ project dependencies
+ workspace config
+ dependency versions
+ runtime / command inputs
        ↓
Task Hash
        ↓
Cache Hit ?
   ├─ Yes → 還原結果
   └─ No  → 執行 task
```

Remote Cache 可以讓不同開發者與 CI job 共用相同的 task results。

---

## 🔥 Affected + Cache 的差異

這兩個很容易混在一起，但責任完全不同：

| 技術 | 解決的問題 |
|---|---|
| Affected | 哪些 project 根本不用跑？ |
| Cache | 需要跑的 task 是否已經算過？ |
| Distribution | 剩下的 task 能不能分散執行？ |

可以想成：

```text
所有 Projects
      ↓
Affected
      ↓
需要執行的 Projects
      ↓
Remote Cache
      ↓
真正需要重新計算的 Tasks
      ↓
Parallel / Distributed Execution
```

---

# 🧪 Cache Miss 排查

如果你發現：

```text
明明沒改什麼
為什麼每次都 Cache Miss？
```

先檢查三件事情：

### 1. Target 是否可 cache

確認 target 有 cache 設定：

```json
{
  "targets": {
    "build": {
      "cache": true
    }
  }
}
```

也可以在 `nx.json` 的 `targetDefaults` 統一設定。

### 2. Inputs 是否正確

Inputs 決定 task hash。

如果不小心把每次都會變動的檔案放進 inputs，就容易造成不必要的 cache miss。

### 3. Outputs 是否正確

Outputs 決定 cache hit 後哪些產物可以被還原。

因此：

```text
inputs  → 決定「這次是不是同一個計算」
outputs → 決定「結果要保存 / 還原什麼」
```

---

# 🅱️ Angular × Nx Monorepo 實戰

如果團隊的 Angular Monorepo 有很多 shared components，我會建議把 CI 優化和 library boundaries 一起看。

例如：

```text
libs/
├── ui/
│   ├── button
│   └── table
├── feature/
│   ├── user
│   └── order
└── data-access/
    ├── user
    └── order
```

如果 `ui/button` 被所有 feature 使用，它的變更自然會造成較大的 affected 範圍。

因此看到：

```text
修改一個 shared library
↓
80% projects affected
```

不要只怪 Nx。

更應該回頭檢查：

- shared library 是否責任過大？
- 是否有不必要的依賴？
- Feature 是否直接依賴 implementation？
- UI、Feature、Data Access 是否有清楚 boundary？

**CI 效能問題有時其實是 architecture 問題的結果。**

---

# 🎯 今日實作 Checklist

- [ ] 執行 `nx graph` 看目前 Project Graph
- [ ] 執行 `nx graph --affected` 看 PR 影響範圍
- [ ] 將 CI 的全量 `test/build/lint` 改成 `nx affected`
- [ ] 確認 GitHub Actions 有完整 Git history
- [ ] 檢查重要 targets 是否可 cache
- [ ] 檢查 `inputs` / `outputs`
- [ ] 找出 Monorepo 中「一改就影響大量 projects」的 shared library

---

# 📌 今日一句話

> **Nx CI 優化不是單純「開 Cache」，而是先用 Affected 減少不必要工作，再用 Remote Cache 消除重複計算，最後才考慮分散執行。**

---

# 🔗 官方來源

- Nx Affected：https://nx.dev/docs/features/ci-features/affected
- Nx Remote Caching：https://nx.dev/docs/features/ci-features/remote-cache
- Nx How Caching Works：https://nx.dev/docs/concepts/how-caching-works
- Nx Setting Up CI：https://nx.dev/docs/getting-started/setup-ci
- Nx GitHub Actions：https://nx.dev/docs/features/ci-features/github-integration
- Nx Project Graph：https://nx.dev/docs/features/explore-graph
- Nx Cache Miss Troubleshooting：https://nx.dev/docs/kb/troubleshoot-cache-misses
