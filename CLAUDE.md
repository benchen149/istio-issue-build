# Claude Code — Project Conventions

## 專案簡介

`istio-issue-build`：`istio/istio` 的個人 fork，用於復現、調查、修復特定 istio issue，並打包成客製化 image 進行驗證。

- **upstream**：`https://github.com/istio/istio.git`
- **origin**：`https://github.com/benchen149/istio-issue-build.git`
- **目前版本**：`1.31`

詳細前置作業與 build 流程請參考 [`issue-build-notes.md`](issue-build-notes.md)。

---

## 開發流程

### 非 issue 變更（文件、設定等）

```
feature branch → PR → develop → PR → master
```

### Upstream issue 修復

```
master → {issue-number}-{slug}（修改、build image、驗證）→ 打 tag → 刪 branch
```

不將 issue branch merge 回 master。若修復成熟要貢獻，對 upstream `istio/istio` 開 PR。

使用 `/github-flow` slash command 自動引導流程。

### Branch 策略

| Branch | 用途 |
|--------|------|
| `master` | 永遠乾淨，只接受 upstream sync 或來自 `develop` 的 PR |
| `develop` | 長期存在，累積非 issue 的個人變更（文件、設定） |
| `{issue-number}-{slug}` | upstream issue 專用，驗證完打 tag 後刪除 |

### Image 命名慣例

Branch 名稱、image TAG、upstream issue 三者對應：

| 情境 | TAG 範例 |
|---|---|
| 復現 / 調查 | `1.31-issue-12345` |
| 嘗試修復 | `1.31-fix-12345` |
| 多次迭代 | `1.31-fix-12345-v2` |

完整 image 參考：`docker.io/benchen149/pilot:1.31-fix-12345`

### Branch Lifecycle

1. 從 `master` 開 branch：`git checkout -b 12345-fix-gateway-crash`
2. 修改、build image、驗證
3. 若要貢獻，對 upstream `istio/istio` 開 PR
4. 完成後打 tag 保留快照：`git tag issue-12345-verified`
5. 刪除 branch：`git push origin --delete 12345-fix-gateway-crash`

> 打 tag 再刪 branch：branch 清單保持乾淨，tag 與 image 仍可對應回 upstream issue。

---

## 環境設定

新環境第一次執行 `/setup-github-ssh` 完成設定：
- SSH key 產生（必須設定 passphrase）
- GitHub known_hosts fingerprint 驗證
- `gh` CLI 登入（Fine-grained Token，限定單一 repo）
- git config user.email / user.name

### GitHub Token（gh CLI）

- 使用 Fine-grained Personal Access Token，限定此 repo
- 最小權限：Contents / Issues / Pull requests Read & Write、Metadata Read-only
- 有效期：建議 90 天
- 儲存位置：`~/.config/gh/hosts.yml`（明文，不可納入 git）

---

## 同步上游

```bash
git fetch upstream
git merge upstream/master
git push origin master
```

---

## Build 客製化 Image

```bash
export HUB=<your-registry>   # 例如 docker.io/benchen149
export TAG=<custom-tag>      # 例如 1.31-fix-issue-12345

make docker.all              # build 所有 image
make docker.push             # push 到 registry
```

個別 component：

```bash
make docker.pilot
make push.docker.pilot
```

---

## Slash Commands

| Command | 說明 |
|---------|------|
| `/setup-github-ssh` | 新環境一次性設定：SSH key、GitHub known_hosts、gh CLI 登入 |
| `/github-flow` | 完整開發流程（issue → PR → merge） |

---

## Issue 寫作七段流程

每次開新 practice issue 前，依序填寫以下七段。參考範本：[issue #9](https://github.com/benchen149/istio-issue-build/issues/9)、[issue #10](https://github.com/benchen149/istio-issue-build/issues/10)。

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

## Issue / PR 標題準則

- **一律使用英文**
- 格式：`<type>: <short description>`
- 範例：`feat: add xxx`、`fix: xxx not working`、`chore: update xxx`

---

## 安全規範

- SSH key passphrase 必須設定，不可留空
- known_hosts 加入前必須對比 GitHub 官方 fingerprint
- `gh` CLI token 不可提交至任何 repo 或 dotfiles
- `~/.ssh/id_ed25519` 不可複製到他處或提交至任何 repo
