# Kuboard GitOps 專案管理工具

單頁 HTML 工具，用於在私有 GitLab 上管理 `kuboard_gitops` 專案的複製、環境建立、Config 修改與 Kuboard 部署 YAML 產生。

---

## 快速開始

1. 下載 `kuboard-gitops-tool.html`
2. 用瀏覽器直接開啟（不需要 server）
3. 填入 GitLab 連線資訊 → 點「測試連線」
4. 開始使用各功能頁籤

---

## 功能總覽

### 0. GitLab 連線設定

| 欄位 | 說明 | 預設值 |
|------|------|--------|
| GitLab 網址 | 私有 GitLab 位址 | `https://git.trevi.cc` |
| Private Access Token | GitLab → User Settings → Access Tokens 產生，需要 `api` scope | — |
| Namespace / Group | 專案所在的 group | `hi_devops` |
| Repository 名稱 | repo 名稱 | `kuboard_gitops` |

連線成功後，工具會自動讀取 `game_project/` 下所有現有專案填入下拉選單。

---

### 頁籤 1：專案複製

從現有專案（如 `tg106`）複製成新專案（如 `tg107`）。

**操作步驟：**
1. 選擇來源專案
2. 填入新專案 ID（小寫，例如 `tg140`）
3. 勾選要保留的環境（dev / stage / uat / gli / prod）
4. 點「執行複製」

**複製行為：**
- 自動替換所有檔案內的舊專案名稱為新專案名稱
- 自動移除 `nodePort` 設定（避免 port 衝突）
- 透過 GitLab Commits API 一次提交，不需要 clone repo

**多平台腳本下載：**

如果不想用網頁操作，也可以下載腳本在本機執行：

| 腳本 | 適用平台 | 執行方式 |
|------|---------|---------|
| `.sh` | Linux / macOS | `GL_TOKEN=xxx bash clone-tg140.sh` |
| `.ps1` | Windows PowerShell | `.\clone-tg140.ps1 -Token xxx` |
| `.bat` | Windows 命令提示字元 | 設定 `GL_TOKEN` 環境變數後執行 |

> 腳本需要本機安裝 Git、Python 3，且已 clone repo。腳本須放在 repo 根目錄下的子目錄（如 `scripts/`）執行。

---

### 頁籤 2：環境複製

在現有專案內，複製一個 overlay 環境到另一個環境。

**範例：** `tg106/game_project/overlays/stage` → `tg106/game_project/overlays/prod`

**操作步驟：**
1. 選擇目標專案
2. 選擇來源環境
3. 選擇目標環境
4. 點「複製環境」

複製時會自動將檔案內容的環境名稱（如 `stg`）替換為目標環境對應的後綴。

---

### 頁籤 3：Stage Config 修改

在 `pp` 專案的三個 stage config 中，新增新遊戲的設定條目。

**需要填入：**

| 欄位 | 說明 | 範例 |
|------|------|------|
| 目標專案 | 新專案 ID | `tg140` |
| 環境 | 要修改的環境 | `stage` |
| 專案英文全名 | 遊戲英文名稱 | `Chicky Mines` |
| 遊戲縮寫碼 | 4 字大寫縮寫 | `CKMN` |

**修改的三個檔案：**

```
game_project/pp/game_project/overlays/stage/
├── backend-console-api-backend/conf/config.yaml
├── cerberus-backend-api-backend/conf/config.yaml
└── game-backend-api-backend/conf/config.yaml
```

**每個檔案新增的內容：**

- `tidb` 區塊：新增資料庫連線設定（`{專案名}_game`）
- `games` 清單：新增遊戲項目（gmcode、name、rooms）
- `gi` 區塊：新增 gi-backend service URL
- `centersvr` 區塊：新增 centersvr service URL

填入後可在頁面預覽 diff，確認無誤後點「確認並推送到 GitLab」。

---

### 頁籤 4：部署執行

#### 下載 Kuboard YAML

從 GitLab 讀取指定專案的 kustomize overlay，輸出完整的 Kubernetes manifest，可直接貼入 Kuboard「Create from YAML」。

**操作步驟：**
1. 選擇目標專案
2. （選填）填入 Image Tag Override，統一覆蓋所有服務的 image tag
3. 點選要下載的環境按鈕

**YAML 包含的資源：**

| 資源類型 | 來源 |
|---------|------|
| `Namespace` | `namespace.yaml` |
| `Deployment` | base `deployment.yaml` + overlay `patch-deployment.yaml` 合併 |
| `Service` | base `service.yaml` + `patch-service.yaml` 合併 |
| `ConfigMap` | `.env` 檔（key/value 格式）+ `conf/config.yaml`（多行格式） |
| `Secret` / `Ingress` | overlay 內其他含 `apiVersion` 的 yaml 自動收入 |

多個資源以 `---` 分隔，Kuboard 會依序建立。

**在 Kuboard 匯入的步驟：**
1. 進入目標 Cluster → 選擇 Namespace（或先讓工具建立）
2. 右上角「Create from YAML」
3. 貼入下載的 YAML 內容
4. 點「Apply」

#### CI/CD 部署觸發（需先設定）

目前尚需在 `.gitlab-ci.yml` 加入支援 API 觸發的 deploy job，並在 Runner 設定 kubeconfig，才能使用此功能。

---

## 環境後綴對照

| 環境名稱 | k8s namespace 後綴 |
|---------|------------------|
| `dev` | `-dev` |
| `stage` | `-stg` |
| `uat` | `-stage` |
| `gli` | `-gli` |
| `prod` | `-prod` |

---

## 注意事項

- 工具使用瀏覽器直接呼叫 GitLab API，**Token 僅存在瀏覽器記憶體中，不會傳送到其他地方**
- 複製操作會直接 commit 到 `main` branch，建議先確認來源專案與新專案 ID 無誤
- Config 修改以 `pp` 專案為模板來源，若 `pp` 專案的 config 結構有異動需同步確認
- 若遇到 CORS 錯誤，請確認 GitLab 的 CORS 設定有允許你的來源，或改用同網域下的 GitLab Pages 部署此工具
