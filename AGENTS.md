# AGENTS.md

本文件適用於整個 `gitops-demo-apps` repository。工作前請先閱讀本文件、`README.md` 與 `docs/ci-cd.md`。

## 專案定位

本 repo 是 Kubernetes application manifests 的 GitOps source of truth：

- 管理 frontend / backend workload manifests、Kustomize overlays 與 Argo CD ApplicationSet。
- frontend / backend 原始碼、Dockerfile 與映像建置 workflow 分別由 `gitops-demo-frontend`、`gitops-demo-backend` 管理。
- Kubernetes 叢集由 `gitops-demo-cluster` 管理。
- Argo CD 安裝、bootstrap 與 cluster registration 由 `gitops-demo-infra` 管理。
- 本 repo 不包含 Terraform，不使用 kubeconfig，也不直接部署至叢集。
- source repo 的 workflow 只建置並推送映像，不會回寫本 repo；映像 tag 變更仍需在 overlay 明確提交。
- GitHub repository 名稱為 `gitops-demo-apps`。
- 預設分支是 `master`。

## 目錄責任

- `apps/<app>/base/`：dev 與 prod 共用 resources。
- `apps/<app>/overlays/dev/`：dev namespace、image、replicas 與環境差異。
- `apps/<app>/overlays/prod/`：prod namespace、image、replicas 與環境差異。
- `argocd/applicationsets/`：dev/prod ApplicationSet。
- `.github/workflows/validate-manifests.yml`：唯一的 CI workflow。
- `.github/actions/setup-k8s-tools/`：CI 驗證工具安裝。
- `.github/yamllint.yaml`：YAML lint 規則。
- `docs/ci-cd.md`：CI trigger、驗證內容、本機執行方式與 Argo CD 同步邊界。

## 註解與術語規範

- 人工維護的程式碼、GitHub Actions、manifest、設定檔與腳本註解必須使用繁體中文。
- 專有名詞、產品名稱、API、Kubernetes 資源種類、欄位名稱、命令、路徑與識別字可保留英文，但英文專有名詞必須放在中文敘述中，不得以完整英文句子撰寫註解。
- `Management Cluster`、`Worker Cluster`、`Cluster` 與 `S3 State Bucket` 均視為專有名詞，不得翻譯成中文，也不得使用其他大小寫變體。
- 複數形式必須寫成 `Management Clusters`、`Worker Clusters` 與 `S3 State Buckets`。
- README、docs、Terraform description、workflow 顯示文字、summary 與人工維護的執行訊息也必須遵守相同的專有名詞大小寫。
- 自動生成檔案（例如 `.terraform.lock.hcl`）的生成器註解、shebang、lint directive 與被註解掉的程式碼不需翻譯或改寫。

## Manifest 規範

- 每個 application 維持 `base`、`overlays/dev`、`overlays/prod` 的對稱結構。
- 環境共用內容放在 base；namespace、replicas、環境變數與 image tag 等差異放在 overlay。
- dev overlay namespace 必須是 `dev`；prod overlay namespace 必須是 `prod`。
- prod 不得引用 dev path、dev namespace 或 mutable dev image tag。
- Deployment、Service、Kustomize image name、replica target、patch target 與 ApplicationSet app name 必須一致。
- 優先使用 Kustomize `images`、`replicas` 與 `patches`，不要在 overlay 複製完整 base resource。
- 不得提交明文 Secret、token、password、kubeconfig 或其他 credentials。
- 不得提交 `output/`、`rendered/` 或 `overlays/local/`。

新增、移除或重新命名 application 時，需同步檢查：

- dev/prod overlays
- dev/prod ApplicationSet app 清單
- README 的路徑與環境表格
- CI 是否仍能自動找到所有 overlays

## Argo CD 規範

- `repoURL` 必須指向本 repository，並與 `gitops-demo-infra` 的根 Application 一致；正確 URL 為 `https://github.com/KittyChen913/gitops-demo-apps.git`。
- `targetRevision` 必須與預設分支 `master` 一致。
- dev ApplicationSet 只能指向 dev overlays 與 `dev` namespace。
- prod ApplicationSet 只能指向 prod overlays 與 `prod` namespace。
- dev 目前啟用 automated sync、prune 與 self-heal。
- prod ApplicationSet 產生的 Applications 目前未啟用 automated sync，Git 變更不代表 workload 已同步至叢集。
- 不得在本 repo 的 CI 加入 `kubectl apply`、Helm deploy、Argo CD API 或 sync 命令。

## CI 規範

CI 保持最小化，只執行：

1. `yamllint` 檢查 `apps/` 與 `argocd/`。
2. `kustomize build` render 所有 dev/prod overlays。
3. `kubeconform` 驗證 render 後的 Kubernetes resources。

不要新增以下流程，除非使用者明確改變專案需求：

- dev/prod 專用 pipeline
- tag release workflow
- promotion workflow
- artifact upload
- rendered diff bot
- reusable workflow 或 CI shell helper
- GitHub Environment approval
- cluster credentials 或部署步驟

Workflow shell 中應使用 `set -euo pipefail`、引用變數，並將 GitHub expression 先放入 `env:` 再用於 shell 邏輯。

## 建議驗證

```bash
yamllint -c .github/yamllint.yaml apps argocd

for overlay in apps/*/overlays/dev apps/*/overlays/prod; do
  kustomize build "$overlay" | kubeconform \
    -strict \
    -summary \
    -kubernetes-version 1.30.0
done

actionlint
```

若本機缺少工具，需明確說明未執行的驗證，不要宣稱完整通過。

## 文件與回覆

- 文件使用繁體中文，技術名稱、命令、路徑與識別字保留英文。
- 修改 manifest 結構或 ApplicationSet 時，需更新 README。
- 修改 CI trigger、驗證工具、驗證命令或 Argo CD 同步邊界時，需更新 `docs/ci-cd.md`。
- 回覆使用者時說明修改檔案、驗證結果與未執行項目。
- 不要回復使用者既有未提交變更。
