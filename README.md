# pterodactyl-on-k8s

在 Kubernetes 上執行 [Pterodactyl Panel](https://pterodactyl.io/) 的最小可用設定，以 Kustomize 組成。可以 clone 下來手動部署，也可以直接用 Argo CD 部署，不需要 fork。

| 元件 | 版本 | 用途 |
| --- | --- | --- |
| Pterodactyl Panel | v1.15.2 | 網頁面板 |
| MariaDB | 11.8 | 資料庫 |
| Valkey | 9.1 | Cache、Session、Queue |

Wings 不在這個 repo 裡。請照[官方文件](https://docs.pterodactyl.io/v1/wings)把 Wings 安裝在獨立主機上（Docker + systemd），不要跟 Kubernetes 節點混用。

## 目錄

- [前提](#前提)
- [檔案結構](#檔案結構)
- [選擇 Ingress](#選擇-ingress)
- [建立 namespace 與 Secret](#建立-namespace-與-secret)
- [方式一：手動部署](#方式一手動部署)
- [方式二：Argo CD 部署](#方式二argo-cd-部署)
- [部署之後](#部署之後)
- [在反向代理後面](#在反向代理後面)
- [日常維運](#日常維運)
- [注意事項](#注意事項)
- [疑難排解](#疑難排解)

## 前提

- Kubernetes 叢集（以 k3s 測試）
- **預設的 StorageClass**：用 `kubectl get storageclass` 確認有一個標示 `(default)`，否則 PVC 會停在 `Pending`
- **支援 NetworkPolicy 的 CNI**：k3s 內建、Calico、Cilium 都支援；只用 Flannel 不支援。不支援時 policy 不會報錯，但**沒有作用**，資料庫會對整個叢集開放
- **Traefik v3** 作為 Ingress Controller（k3s 內建）
- 使用 `https/` 時：**cert-manager** 和一個 **ClusterIssuer**
- 使用 Argo CD 部署時：**Argo CD 2.8 以上**（支援 `kustomize.patches`）

確認 Traefik 版本：

```bash
kubectl -n kube-system get deploy traefik -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## 檔案結構

```
pterodactyl-on-k8s/
├── base/                      共用的部分
│   ├── kustomization.yaml
│   ├── namespace.yaml         pterodactyl namespace
│   ├── mariadb.yaml           MariaDB StatefulSet、headless Service、NetworkPolicy
│   ├── valkey.yaml            Valkey StatefulSet、headless Service、NetworkPolicy
│   ├── configmap.yaml         Panel 設定：網址、資料庫、Valkey、寄信
│   └── panel.yaml             Panel Deployment、PVC、Service
├── https/                     Traefik 直接對外，cert-manager 簽 HTTPS 憑證
│   ├── kustomization.yaml
│   └── ingress.yaml
├── proxy/                     前面有反向代理處理 HTTPS，Traefik 只收 HTTP
│   ├── kustomization.yaml
│   └── ingress-proxy.yaml
└── argocd/
    └── application.yaml       Argo CD Application 範本
```

MariaDB 和 Valkey 都設了 NetworkPolicy，只有同一個 namespace 內的 Pod 能連線。

## 選擇 Ingress

| 目錄 | 使用情境 |
| --- | --- |
| `https/` | Traefik 直接對外，由 cert-manager 簽發憑證，HTTP 自動轉址到 HTTPS |
| `proxy/` | 前面有 Caddy、Nginx 等反向代理處理 HTTPS（見[在反向代理後面](#在反向代理後面)） |

## 建立 namespace 與 Secret

兩種部署方式都要先做這一步。Secret 不放進 repo，Argo CD 也**不會**建立 Secret。

```bash
kubectl create namespace pterodactyl

kubectl -n pterodactyl create secret generic mariadb-secret \
  --from-literal=root-password="$(openssl rand -hex 24)" \
  --from-literal=password="$(openssl rand -hex 24)"

kubectl -n pterodactyl create secret generic valkey-secret \
  --from-literal=password="$(openssl rand -hex 24)"

kubectl -n pterodactyl create secret generic panel-secret \
  --from-literal=APP_KEY="base64:$(openssl rand -base64 32)" \
  --from-literal=HASHIDS_SALT="$(openssl rand -hex 10)" \
  --from-literal=HASHIDS_LENGTH="8" \
  --from-literal=MAIL_USERNAME="<SMTP 帳號>" \
  --from-literal=MAIL_PASSWORD='<SMTP 密碼>'
```

> [!IMPORTANT]
> **立刻備份 `APP_KEY`、`HASHIDS_SALT`、`HASHIDS_LENGTH`。**
> `APP_KEY` 遺失後，資料庫裡的 2FA、API key 和 Wings token 都會解不開；
> `HASHIDS_SALT` 換掉後，伺服器的網址 ID 會全部改變。
>
> ```bash
> for k in APP_KEY HASHIDS_SALT HASHIDS_LENGTH; do
>   echo "$k=$(kubectl -n pterodactyl get secret panel-secret -o jsonpath="{.data.$k}" | base64 -d)"
> done
> ```

從既有的 Panel 遷移時，`APP_KEY`、`HASHIDS_SALT`、`HASHIDS_LENGTH` 要改填舊 Panel `.env` 裡的值（不含引號）。

## 方式一：手動部署

### 1. Clone 並修改設定

```bash
git clone https://github.com/redbean0721/pterodactyl-on-k8s.git
cd pterodactyl-on-k8s
git checkout v1.0.0
```

| 檔案 | 要修改的欄位 |
| --- | --- |
| `base/configmap.yaml` | `APP_URL`、`APP_SERVICE_AUTHOR`、`MAIL_*`、`TRUSTED_PROXIES`，建議取消註解 `MAIL_EHLO_DOMAIN` |
| `https/ingress.yaml` | 所有 `host`、`tls.hosts`（要跟 `APP_URL` 的網域一致）、`cert-manager.io/cluster-issuer` |
| `proxy/ingress-proxy.yaml` | `host`（要跟 `APP_URL` 的網域一致） |

`TRUSTED_PROXIES` 要填叢集的 Pod CIDR，預設是 k3s 的 `10.42.0.0/16,2001:cafe:42::/56`。其他叢集請改成自己的網段，例如 kubeadm + Flannel 是 `10.244.0.0/16`。

只用 HTTP（沒有 HTTPS）時，`APP_URL` 改成 `http://`，`SESSION_SECURE_COOKIE` 改成 `"false"`，否則會無法登入。

### 2. 部署

一個一個套用：

```bash
kubectl apply -f base/namespace.yaml
kubectl apply -f base/mariadb.yaml
kubectl apply -f base/valkey.yaml
kubectl apply -f base/configmap.yaml
kubectl apply -f base/panel.yaml
kubectl apply -f https/ingress.yaml        # 或 proxy/ingress-proxy.yaml
```

或一次套用：

```bash
kubectl kustomize https/     # 預覽，不會套用
kubectl apply -k https/      # 或 proxy/
```

> [!WARNING]
> 不要用 `kubectl apply -f base/` 套用整個資料夾。資料夾裡的 `kustomization.yaml` 不是 Kubernetes 資源，會導致錯誤。一次套用請用 `-k`。

接著到[部署之後](#部署之後)。

## 方式二：Argo CD 部署

不需要 clone 或 fork。Argo CD 直接讀取這個 repo，你的設定寫在 Application 的 `kustomize.patches` 裡覆蓋預設值。

### 1. 下載範本

```bash
curl -LO https://raw.githubusercontent.com/redbean0721/pterodactyl-on-k8s/v1.0.0/argocd/application.yaml
```

### 2. 修改 Application

| 欄位 | 說明 |
| --- | --- |
| `targetRevision` | 版本 tag，例如 `v1.0.0`。**不要用 `main`**，否則這個 repo 的每次修改都會直接部署到你的叢集 |
| `path` | `https` 或 `proxy` |
| ConfigMap 的 patch | 只列出要改的欄位，其他欄位保留預設值 |
| Ingress 的 patch | 網域和 ClusterIssuer |

`path: proxy` 時，要刪掉 `https` 用的兩段 Ingress patch，改用檔案裡註解掉的 proxy 版本。proxy 的 Ingress 沒有 `tls` 和轉址用的 Ingress，沿用 `https` 的 patch 會導致同步失敗。

> [!NOTE]
> ConfigMap 的 patch 必須包含 `namespace: pterodactyl`，否則 kustomize 找不到要覆蓋的 ConfigMap。
> Ingress 的 patch 使用 JSON Patch，路徑中的 `/` 要寫成 `~1`，例如 `cert-manager.io~1cluster-issuer`。

### 3. 套用

```bash
kubectl apply -f application.yaml
```

Argo CD 會接管先前手動建立的 `pterodactyl` namespace，不會衝突。

> [!CAUTION]
> 範本沒有設定 `resources-finalizer`，所以刪除 Application 時**只會刪掉 Application 本身**，叢集上的資源都會保留。
> 如果自行加上 `resources-finalizer.argocd.argoproj.io`，刪除 Application 時 Argo CD 會連同 `pterodactyl` namespace 一起刪除，**Secret 和資料庫的 PVC 也會跟著消失**。

### 升級到新版本

把 `targetRevision` 改成新的 tag，再 `kubectl apply -f application.yaml`。升級前請先[備份資料庫](#備份資料庫)，並閱讀該版本的 release notes。

## 部署之後

### 確認啟動

```bash
kubectl -n pterodactyl get pods -w
kubectl -n pterodactyl logs deploy/panel -c panel -f
```

第一次啟動會跑資料庫 migration，大約需要 1～3 分鐘。log 出現 `Starting supervisord`、Pod 變成 `1/1 Running` 就完成了。

### 建立管理員

```bash
kubectl -n pterodactyl exec -it deploy/panel -c panel -- php artisan p:user:make
```

`Is this user an administrator?` 回答 `yes`。

### 連接 Wings

在後台建立 Location 和 Node，把 Node 的設定檔複製到 Wings 主機的 `/etc/pterodactyl/config.yml`。

- Wings 要能連到 `APP_URL`
- 瀏覽器的 console 和檔案上傳是**直接連到 Wings**，不經過 Panel
- Panel 使用 HTTPS 時，Wings 也必須使用 HTTPS

## 在反向代理後面

`proxy/` 適用這種架構：

```
使用者 ──HTTPS──▶ 反向代理 ──HTTP──▶ Traefik ──▶ Panel
```

反向代理要把請求轉給 Traefik 的 HTTP port，並**保留原本的 `Host` header**。

`ingress-proxy.yaml` 會把傳給 Panel 的 `X-Forwarded-Proto` 設成 `https`，讓 Panel 產生 `https://` 的連結。這個設定不需要修改 Traefik 本身，任何叢集都能直接使用。

### 進階：記錄使用者真實 IP

預設情況下，Traefik 不信任反向代理傳來的 `X-Forwarded-For`，Panel 的登入紀錄會顯示反向代理的 IP 或叢集內部 IP，而不是使用者的 IP。Panel 可以正常運作，但登入紀錄和以 IP 為單位的登入次數限制會不準確。

需要真實 IP 時，要讓 Traefik 信任反向代理。k3s 內建的 Traefik 用 HelmChartConfig 設定：

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    service:
      spec:
        externalTrafficPolicy: Local
    ports:
      web:
        forwardedHeaders:
          trustedIPs:
            - 192.0.2.10/32     # 反向代理的 IP
```

> [!WARNING]
> 這個設定會影響**整個叢集**的 Traefik。已經有 `traefik-config.yaml` 時，請合併進去，不要覆蓋。

`externalTrafficPolicy: Local` 讓 Traefik 看得到真實的來源 IP。預設的 `Cluster` 可能會做 SNAT，把來源 IP 改成叢集內部位址，`trustedIPs` 就比對不到。副作用是流量只會送到有 Traefik Pod 的節點。

確認目前的設定：

```bash
kubectl -n kube-system get svc traefik -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
```

### 選用：只允許反向代理連線

`ingress-proxy.yaml` 裡有一個 `from-proxy-only` Middleware，預設沒有啟用。Traefik 也能從外網直接連到時，可以啟用它，避免有人繞過反向代理。

前提是 Traefik 看得到反向代理的真實 IP（`externalTrafficPolicy: Local`），否則**連反向代理也會被擋**。啟用方式見檔案內的註解。

## 日常維運

### 修改設定

Panel 只在 **Pod 啟動時**讀取 ConfigMap 和 Secret。修改後要重啟才會生效，Argo CD 同步後也一樣：

```bash
kubectl -n pterodactyl rollout restart deploy/panel
```

> [!NOTE]
> `APP_ENVIRONMENT_ONLY` 為 `"false"` 時，後台 Settings 頁面的寄信、reCAPTCHA 設定會存進資料庫，並**覆蓋** ConfigMap 的值。在後台改過寄信設定後，再改 ConfigMap 不會生效。

### 備份資料庫

```bash
kubectl -n pterodactyl exec mariadb-0 -- sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --single-transaction --routines --triggers panel' \
  > panel-backup-$(date +%Y%m%d-%H%M).sql
```

### 還原資料庫

```bash
kubectl -n pterodactyl scale deploy/panel --replicas=0

kubectl -n pterodactyl exec mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -e "DROP DATABASE panel; CREATE DATABASE panel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"'

kubectl -n pterodactyl exec -i mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" panel' < panel-backup-XXXX.sql

kubectl -n pterodactyl scale deploy/panel --replicas=1
```

使用 Argo CD 時，`selfHeal` 會把 replicas 改回 1。還原期間請先在 Argo CD 暫停自動同步。

### 升級 Panel

Panel 的映像檔**每次啟動都會自動跑 migration**，而 migration 不保證能降版。

1. [備份資料庫](#備份資料庫)
2. 手動部署：`git pull` 並切換到新的 tag，再 `kubectl apply -k https/`
   Argo CD：把 `targetRevision` 改成新的 tag
3. 看 log 確認 migration 完成

降版時不能只把版本改回去，要連同資料庫一起還原到升級前的備份。

> [!CAUTION]
> **不要把 Panel 的 image tag 改成 `latest`。** tag 為 `latest` 時，Pod 每次重建（節點重開機、重新排程）都會拉取最新的映像檔，並自動升級資料庫，沒有機會先備份。大版本（例如 2.0）發布後，也可能在不知情的情況下被直接升級。

升級 Panel 時，Wings 也建議一起更新到對應的版本。

## 注意事項

- **Panel 只能有 1 個 replica**：映像檔內同時執行 cron 和 queue worker，多個 replica 會重複執行排程。
- **內建 Egg 會被覆蓋**：Panel 每次啟動都會執行 seeder，把官方內建的 Egg 更新成映像檔內附的版本。要自訂 Egg，請匯出後再匯入成新的 Egg。
- **PVC 綁定單一節點**：使用 local-path 等本機儲存時，資料存在 Pod 所在節點的硬碟上，該節點離線時服務無法啟動。
- **刪除時 PVC 的差異**：`kubectl delete -k https/` 會刪除 Panel 的 PVC（只有 `.env` 和 log，可以重建），但 MariaDB 和 Valkey 的 PVC 由 StatefulSet 建立，會保留下來。要完全清除時，請另外刪除 `data-mariadb-0` 和 `data-valkey-0`。
- **換 namespace**：Ingress 的 `router.middlewares` annotation 寫死了 `pterodactyl-` 前綴，換 namespace 時要一起修改。

## 疑難排解

查看 Panel 的 Laravel log：

```bash
kubectl -n pterodactyl exec deploy/panel -c panel -- \
  sh -c 'tail -n 60 "$(ls -t /app/storage/logs/*.log | head -1)"'
```

確認 MariaDB 和 Valkey：

```bash
kubectl -n pterodactyl exec mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -e "SELECT VERSION(); SHOW VARIABLES LIKE \"have_ssl\";"'

kubectl -n pterodactyl exec valkey-0 -- sh -c \
  'valkey-cli -a "$VALKEY_PASSWORD" --no-auth-warning ping'
```

| 狀況 | 原因 |
| --- | --- |
| PVC 停在 `Pending` | 叢集沒有預設的 StorageClass |
| Panel 停在 `Init:0/1` | 連不到 Valkey，看 `kubectl -n pterodactyl logs deploy/panel -c wait-for-deps` |
| `CreateContainerConfigError` | Secret 還沒建立，或名稱、key 打錯，看 `kubectl -n pterodactyl describe pod -l app=panel` |
| log 出現 `Waiting for database connection...` | 連不到 MariaDB |
| `SSL is required, but the server does not support it` | MariaDB 版本低於 11.4 |
| `Access denied for user 'pterodactyl'` | MariaDB 初始化之後才修改 `mariadb-secret`，資料庫裡的密碼不會跟著改 |
| `NOAUTH`、`WRONGPASS` | Valkey 密碼不一致 |
| `The MAC is invalid` | `APP_KEY` 跟資料庫不符 |
| 登入後一直被踢回登入頁 | 只用 HTTP，但 `SESSION_SECURE_COOKIE` 是 `"true"` |
| 瀏覽器出現混合內容錯誤、轉址循環 | Panel 以為自己跑在 HTTP 上：檢查 `APP_URL`、`TRUSTED_PROXIES`，反向代理架構要用 `proxy/` |
| Ingress 回 `403` | 啟用了 `from-proxy-only`，但 Traefik 看到的來源 IP 不是反向代理 |
| Ingress 回 `404` | `host` 跟請求的網域不符，或 `ingressClassName` 不對 |
| 憑證一直沒有簽發 | DNS 沒有指向 Traefik，或 ClusterIssuer 名稱錯誤，看 `kubectl -n pterodactyl describe certificate` |
| Argo CD：`unable to resolve revision` | `targetRevision` 指定的 tag 不存在 |
| Argo CD：`no resource matches strategic merge patch` | ConfigMap 的 patch 少了 `namespace: pterodactyl` |
| Argo CD：`doc is missing path: /spec/tls/0/hosts/0` | `path: proxy` 卻沿用了 `https` 的 Ingress patch |