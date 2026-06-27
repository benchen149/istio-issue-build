# 開發前置作業與流程說明

## 一、Branch 策略總覽

| Branch | 用途 |
|---|---|
| `master` | 永遠乾淨，只接受 upstream sync 或來自 `develop` 的 PR |
| `develop` | 長期存在，累積非 issue 的個人變更（文件、設定） |
| `{issue-number}-{slug}` | upstream issue 專用，驗證完打 tag 後刪除 |

**非 issue 變更流程：**
```
feature branch → PR → develop → PR → master
```

**Upstream issue 修復流程：**
```
master → {issue-number}-{slug} → 修改、build image、驗證 → 打 tag → 刪 branch
```

---

## 二、新環境初始化

```bash
# Clone 個人 fork
git clone https://github.com/benchen149/istio-issue-build.git
cd istio-issue-build

# 加入 upstream remote
git remote add upstream https://github.com/istio/istio.git

# 確認 remote 設定
git remote -v
```

> `.claude/settings.local.json` 不會跟著 clone 過來（gitignored），新環境需執行 `/setup-github-ssh` 重新設定。

---

## 三、同步上游（upstream）

### 何時需要 sync

- **開新 issue branch 之前**：必須 sync，確保從最新的 master 分支出去
- **在同一個 branch 內持續修改**：不需要
- **距離上次 sync 已超過幾天**：建議先確認是否落後，再決定是否 sync

### 確認是否落後

```bash
# 先 fetch（不會改動任何 branch）
git fetch upstream

# 查看落後幾個 commit（沒有輸出代表已是最新）
git log master..upstream/master --oneline
```

### 執行 sync

```bash
# 合併上游 master 到本地
git merge upstream/master

# 推回自己的 fork
git push origin master
```

> 這三步對上游 repo（`istio/istio`）零影響；`push` 目標是 `origin`（個人 fork），不是 `upstream`。

---

## 四、修改 Issue 並打包客製化 Istio Image

### 1. 修改程式碼

在 repo 中針對目標 issue 進行修改。

### 2. 設定版號與 Registry

透過環境變數指定 image hub（registry）與 tag（版號）：

```bash
export HUB=<your-registry>        # 例如 docker.io/<yours account>
export TAG=<custom-tag>           # 例如 1.31-fix-issue-12345
```

> 預設 `TAG` 為當前 git commit hash，`HUB` 預設為 `istio`。

### 3. Build Image

```bash
# Build 所有 image
make docker.all

# 或只 build 特定 component（例如 pilot）
make docker.pilot
```

### 4. Push Image

```bash
# Push 所有 image 到指定 registry
make docker.push

# 或只 push 特定 component
make push.docker.pilot
```

### 5. 版號慣例建議

Branch 名稱、image TAG、upstream issue 三者對應：

| 情境 | TAG 範例 |
|---|---|
| 復現 / 調查 | `1.31-issue-12345` |
| 嘗試修復 | `1.31-fix-12345` |
| 多次迭代 | `1.31-fix-12345-v2` |

完整 image 參考：`docker.io/benchen149/pilot:1.31-fix-12345`

### 6. Branch Lifecycle

1. 從 `master` 開 branch：`git checkout -b 12345-fix-gateway-crash`
2. 修改、build image、驗證
3. 若要貢獻，對 upstream `istio/istio` 開 PR
4. 完成後打 tag 保留快照：`git tag issue-12345-verified`
5. 刪除 branch：`git push origin --delete 12345-fix-gateway-crash`

> 打 tag 再刪 branch：branch 清單保持乾淨，tag 與 image 仍可對應回 upstream issue。

---

## 五、Claude Commands — Development Workflow

此專案提供 Claude Code slash commands，讓任何新的開發環境都能快速完成環境設定並遵循統一的開發流程。

| Command | 說明 |
|---|---|
| `/setup-github-ssh` | 新環境一次性設定：SSH key 產生、GitHub 綁定、gh CLI 登入（含安全規範） |
| `/github-flow` | 完整開發流程：issue → branch → commit → PR → merge |

**建議順序（新環境第一次）：**
```
1. /setup-github-ssh   # 完成 SSH + gh CLI 認證設定
2. /github-flow        # 開始功能開發
```

詳細步驟請參考 `.claude/commands/` 目錄下的對應 `.md` 檔案。

---

## 六、Issue 寫作七段流程

每次開新 practice issue 前，依序填寫以下七段。參考範本：[issue #9](https://github.com/benchen149/istio-issue-build/issues/9)。

### 段落結構

1. **Upstream 資訊** — 原始 issue/PR 連結、為何選 backport commit（不選 master commit）、fix 大意一段話
2. **User 情境** — 需求、設定 YAML、bug 如何靜悄悄失效（沒有 error 但行為不對）
3. **根本原因** — 舊碼位置（`file:line`）、判斷錯在哪、舊 vs 新對照表
4. **前置作業** — 確認 tag 存在、確認目標檔案穩定、dry run（`--no-commit` 確認無 CONFLICT）
5. **復現手法** — apply 最小測試資源、驗證指令＋python script 存檔、期望 bug 狀態輸出
6. **修改過程** — 開 branch → cherry-pick → build → kind load → 替換 istiod → 再次驗證
7. **修復後 User 建議** — fix 解的是 Istio 自身問題還是 user 設定問題？兩件事分開說

### 選題標準

| 標準 | 確認方式 |
|------|----------|
| 目標版本有此 bug | `git log tags/1.13.5..upstream/release-1.13 \| grep <commit>` 有輸出 |
| 有 backport commit | 優先使用目標版本的 backport，衝突風險最低 |
| Fix diff 小 | `git show <commit> --stat`，避免大規模重構 |
| 可用 `istioctl` 驗證 | 想好「修前看什麼 → 修後看什麼」，不依賴流量行為 |

### 常見坑

| 情況 | 正確做法 |
|------|----------|
| Multi-line python 指令 | 存 `/tmp/check_xxx.py`，不用 `python3 -c` inline |
| 改回 stock image 後看不到 bug | 重新 apply 測試資源（清理後需重建） |
| dry run 後想復原 | `git reset --hard HEAD`，不是 `git cherry-pick --abort` |
| build 前忘確認 branch | `git branch --show-current` |

---

## 七、練習題目參考

以下為已完整走過 issue branch 流程的練習題目，可作為新手入門或流程複習的參考範本。

| Issue | 說明 | 難度 | 驗證方式 |
|-------|------|------|----------|
| [#9 VirtualService catch-all route 截斷後續規則](https://github.com/benchen149/istio-issue-build/issues/9) | `isCatchAllMatch` 只看 Istio API 層，導致 port-only match 被誤判，canary 規則永遠不可達 | ⭐ 低 | `istioctl pc routes` 看 route count |

> 每個題目都包含：upstream fix 來龍去脈、user 情境、dry run、復現、build & verify 的完整步驟，可直接照著手動操作。

---

## 七、備註

- `VERSION` 檔案目前為 `1.31`，build 時會自動附加 `-custom` 後綴
- 本 repo 為 `istio/istio` 的 fork，upstream 為官方 repo，origin 為個人 fork
