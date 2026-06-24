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

## 二、同步上游（upstream）

```bash
# 取得上游最新變更
git fetch upstream

# 合併上游 master 到本地
git merge upstream/master

# 推回自己的 fork
git push origin master
```

---

## 三、修改 Issue 並打包客製化 Istio Image

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

## 四、Claude Commands — Development Workflow

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

## 五、備註

- `VERSION` 檔案目前為 `1.31`，build 時會自動附加 `-custom` 後綴
- 本 repo 為 `istio/istio` 的 fork，upstream 為官方 repo，origin 為個人 fork
