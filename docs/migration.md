# 從主機安裝遷移到 Kubernetes

把用官方文件直接安裝在主機上的 Pterodactyl Panel（`/var/www/pterodactyl`），遷移到這個 repo 部署的 Panel。

Wings 和遊戲伺服器的檔案都在 Wings 主機上，**不需要搬動**。要遷移的只有 Panel 的資料庫和三個金鑰。

## 流程總覽

1. [準備](#1-準備)：確認版本，取得舊 Panel 的金鑰
2. [部署新 Panel](#2-部署新-panel)：使用舊的金鑰和相同的版本
3. [演練](#3-演練建議)（建議）：匯入資料庫複本，確認資料正確，不影響正式環境
4. [正式切換](#4-正式切換)：停機、匯入最新資料、切換 DNS
5. [驗證](#5-驗證)
6. [回滾](#6-回滾)：有問題時切回舊 Panel

## 1. 準備

### 確認舊 Panel 的版本

在舊主機上：

```bash
cd /var/www/pterodactyl
grep "'version'" config/app.php
```

**新 Panel 必須先使用相同的版本**，遷移完成、確認沒問題後再升級。Panel 啟動時會自動跑 migration，版本不同的話，匯入的資料庫會直接被升級，而且不能降版。

### 確認舊 Panel 的資料庫

```bash
mysql --version
```

如果舊主機用的是 **MySQL 8**（不是 MariaDB），匯入時可能會遇到排序規則不相容的問題，見[疑難排解](#疑難排解)。

### 取得金鑰

```bash
grep -E '^(APP_KEY|HASHIDS_SALT|HASHIDS_LENGTH|APP_TIMEZONE|APP_URL)=' /var/www/pterodactyl/.env
```

| 值 | 用途 | 遺失的後果 |
| --- | --- | --- |
| `APP_KEY` | 加密 2FA、API key、Wings token | 全部解不開，使用者無法登入 2FA，Wings 連不上 |
| `HASHIDS_SALT`、`HASHIDS_LENGTH` | 產生伺服器網址上的短 ID | 舊連結和 API 呼叫全部失效 |
| `APP_TIMEZONE` | 時區 | 排程時間可能偏移 |
| `APP_URL` | Panel 網址 | — |

> [!CAUTION]
> 這些值不要貼到聊天室、issue 或任何公開的地方。

## 2. 部署新 Panel

### 建立 Secret（使用舊的金鑰）

照 README 建立 namespace、`mariadb-secret` 和 `valkey-secret`。`panel-secret` 則改用舊 Panel 的值，用 `read` 輸入，值不會留在 shell history 裡：

```bash
read -rsp 'APP_KEY: ' P_KEY; echo
read -rsp 'HASHIDS_SALT: ' P_SALT; echo
read -rp  'HASHIDS_LENGTH: ' P_LEN
read -rp  'MAIL_USERNAME: ' P_MUSER
read -rsp 'MAIL_PASSWORD: ' P_MPASS; echo

kubectl -n pterodactyl create secret generic panel-secret \
  --from-literal=APP_KEY="$P_KEY" \
  --from-literal=HASHIDS_SALT="$P_SALT" \
  --from-literal=HASHIDS_LENGTH="$P_LEN" \
  --from-literal=MAIL_USERNAME="$P_MUSER" \
  --from-literal=MAIL_PASSWORD="$P_MPASS"

unset P_KEY P_SALT P_LEN P_MUSER P_MPASS
```

輸入時**不要包含引號**。`.env` 裡的值如果是 `"abc"`，就輸入 `abc`。

確認 `APP_KEY` 跟舊 Panel 完全一樣，兩邊的輸出要相同：

```bash
# Kubernetes
kubectl -n pterodactyl get secret panel-secret -o jsonpath='{.data.APP_KEY}' | base64 -d | sha256sum

# 舊主機
grep '^APP_KEY=' /var/www/pterodactyl/.env | cut -d= -f2- | tr -d '"' | tr -d '\n' | sha256sum
```

### 設定相同的版本與時區

**手動部署**：修改 `base/panel.yaml` 的 image tag，以及 `base/configmap.yaml` 的 `APP_TIMEZONE`。

**Argo CD**：在 Application 加上：

```yaml
spec:
  source:
    kustomize:
      images:
        - ghcr.io/pterodactyl/panel:v1.x.x     # 舊 Panel 的版本
      patches:
        - patch: |-
            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: panel-config
              namespace: pterodactyl
            data:
              APP_TIMEZONE: "Asia/Taipei"      # 舊 Panel 的時區
              # ...其他設定
```

### 部署

照 README 部署。Panel 第一次啟動會在空的資料庫建立資料表，這是正常的，之後匯入時會整個覆蓋掉。

`APP_URL` 和 Ingress 的網域：

- **要演練**：先用一個測試網域，正式切換時再改成舊的網址
- **不演練**：直接用舊 Panel 的網址，DNS 先不要改

## 3. 演練（建議）

匯入一份 prod 資料庫的複本，確認資料正確，再正式切換。

> [!WARNING]
> 複本裡有 Node 的 daemon token，而 `APP_KEY` 又跟舊 Panel 相同，所以**新 Panel 能直接操作正式環境的 Wings**。有人在新 Panel 上按下「停止伺服器」，或是排程觸發，正式環境的遊戲伺服器就真的會停。演練前一定要先封鎖新 Panel 的對外連線。

### 封鎖 Panel 的對外連線

建立 `panel-egress-rehearsal.yaml`，讓 Panel 只能連 MariaDB、Valkey 和 DNS：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: panel-egress-rehearsal
  namespace: pterodactyl
spec:
  podSelector:
    matchLabels:
      app: panel
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels: { app: mariadb }
      ports:
        - { protocol: TCP, port: 3306 }
    - to:
        - podSelector:
            matchLabels: { app: valkey }
      ports:
        - { protocol: TCP, port: 6379 }
    - to:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: kube-system }
          podSelector:
            matchLabels: { k8s-app: kube-dns }
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

```bash
kubectl apply -f panel-egress-rehearsal.yaml

# 確認：連不到外面（exit code 1），但連得到資料庫（open）
kubectl -n pterodactyl exec deploy/panel -c panel -- nc -zv -w3 1.1.1.1 443
kubectl -n pterodactyl exec deploy/panel -c panel -- nc -zv -w3 mariadb 3306
```

這個檔案**不要**放進 kustomization。使用 Argo CD 時，`prune: false` 不會刪除它，但也不會管理它。

### 匯出並匯入

照[正式切換](#4-正式切換)的「匯出」和「匯入」步驟操作，但**不需要**在舊 Panel 執行 `php artisan down`。`--single-transaction` 匯出時不會鎖表，舊 Panel 可以繼續運作。

匯入後，**只在演練的複本上**執行：

```bash
kubectl -n pterodactyl exec mariadb-0 -- sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" panel -e "
  UPDATE schedules SET is_active = 0;
  UPDATE settings SET value = \"false\" WHERE \`key\` = \"settings::recaptcha:enabled\";
"'

kubectl -n pterodactyl rollout restart deploy/panel
```

- **關閉排程**：雙重保險，避免排程在演練期間觸發。
- **關閉 reCAPTCHA**：Panel 驗證 reCAPTCHA 時要連到 Google，被 egress 封鎖後**會無法登入**。而且 reCAPTCHA 綁定的是舊網域，在測試網域上本來也驗證不過。

### 檢查

- [ ] 用舊 Panel 的帳號密碼能登入
- [ ] 有開 2FA 的帳號能通過驗證（`APP_KEY` 正確）
- [ ] 伺服器網址的 ID 跟舊 Panel 相同（`HASHIDS_SALT` 正確）
- [ ] 使用者、伺服器、Node、Location、Nest、Egg 的數量跟舊 Panel 一致
- [ ] **常用 Egg 的啟動指令、Docker image、安裝腳本跟舊 Panel 一致**（見[內建 Egg 會被覆蓋](#內建-egg-會被覆蓋)）
- [ ] Node 顯示紅色、連不上：這是預期的結果，代表封鎖有效

開啟伺服器頁面時出現 `cURL error 28` 或 `Connection timed out` 也是預期的，因為 Panel 連不到 Wings。

## 4. 正式切換

### 停止舊 Panel

在舊主機上：

```bash
cd /var/www/pterodactyl
php artisan down
```

使用者會看到維護頁面，避免切換期間有人修改資料。Wings 和遊戲伺服器**不受影響**，會繼續運作。

### 匯出

在舊主機上，用 `.env` 裡的資料庫帳號：

```bash
mysqldump -u pterodactyl -p \
  --single-transaction --quick --routines --triggers --no-tablespaces \
  panel | gzip > panel-migration.sql.gz
```

把檔案複製到能執行 `kubectl` 的機器上。

### 匯入

```bash
# 停止新 Panel
kubectl -n pterodactyl scale deploy/panel --replicas=0

# 清空資料庫
kubectl -n pterodactyl exec mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -e "DROP DATABASE panel; CREATE DATABASE panel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"'

# 匯入
zcat panel-migration.sql.gz | kubectl -n pterodactyl exec -i mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" panel'

# 確認筆數
kubectl -n pterodactyl exec mariadb-0 -- sh -c \
  'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" panel -e "SELECT COUNT(*) AS users FROM users; SELECT COUNT(*) AS servers FROM servers;"'

# 清空 Valkey 裡的 session 和 cache
kubectl -n pterodactyl exec valkey-0 -- sh -c \
  'valkey-cli -a "$VALKEY_PASSWORD" --no-auth-warning FLUSHALL'
```

使用者和伺服器的數量要跟舊 Panel 一致。

> [!NOTE]
> 使用 Argo CD 時，`selfHeal` 會把 Panel 的 replicas 改回 1。匯入期間請先在 Argo CD 暫停自動同步。

**正式切換時不要關閉排程和 reCAPTCHA**，它們要維持舊 Panel 的狀態。

### 切換設定

做過演練的話：

1. 刪除演練用的封鎖：
   ```bash
   kubectl -n pterodactyl delete networkpolicy panel-egress-rehearsal
   ```
2. 把 `APP_URL` 和 Ingress 的網域改成舊 Panel 的網址，套用或 push

啟動新 Panel：

```bash
kubectl -n pterodactyl scale deploy/panel --replicas=1
kubectl -n pterodactyl logs deploy/panel -c panel -f
```

版本相同的話，log 會顯示 `Nothing to migrate`。

### 切換 DNS

把 Panel 網址的 DNS 指向新的 Ingress。使用 `https/` 時，等 cert-manager 簽好憑證：

```bash
kubectl -n pterodactyl get certificate
# READY 欄位變成 True
```

> [!NOTE]
> cert-manager 使用 HTTP-01 驗證時，DNS 切換之前簽不到憑證，切換後會自動重試，中間會有幾分鐘沒有 HTTPS。使用 DNS-01 的話，可以在切換前先簽好。

### Wings

- **網址不變**：Wings 不需要任何修改，會自動連到新的 Panel。
- **網址改變**：每台 Wings 主機都要修改 `/etc/pterodactyl/config.yml` 的 `remote:`，或從新 Panel 後台的 Node → Configuration 重新複製設定檔，然後執行 `systemctl restart wings`。

## 5. 驗證

- [ ] 用舊帳號登入，2FA 能通過驗證
- [ ] 所有 Node 都是綠色的心跳
- [ ] 抽查幾台伺服器：console、啟動 / 停止、檔案管理、SFTP
- [ ] 寄一封忘記密碼信，確認寄信正常
- [ ] 舊的 API key 還能使用（如果有在用）
- [ ] 排程有正常觸發

確認沒問題後，就可以把 Panel 升級到新版本（見 README 的「升級 Panel」）。

舊主機建議**保留一段時間**再移除，作為回滾的退路。

## 6. 回滾

新 Panel 有問題時：

1. 把 DNS 指回舊主機
2. 在舊主機上執行 `php artisan up`
3. 網址有改過的話，把 Wings 的 `remote:` 改回來並重啟

> [!WARNING]
> 切換後在新 Panel 上產生的變更（新的使用者、伺服器、設定），回滾後會遺失。

## 注意事項

### 內建 Egg 會被覆蓋

Docker 版的 Panel **每次啟動都會執行 seeder**，把官方內建的 Egg（Paper、Forge、Vanilla 等）更新成映像檔內附的版本。主機安裝的 Panel 只有在升級時才會執行 seeder，所以你可能從來沒遇過這個問題。

如果舊 Panel 上**直接修改過官方內建的 Egg**，這些修改在新 Panel 每次重啟後都會消失。遷移前請：

1. 在舊 Panel 匯出修改過的 Egg
2. 在新 Panel 匯入成新的 Egg（UUID 不同，seeder 就不會覆蓋）
3. 把使用這些 Egg 的伺服器改用新匯入的 Egg

### 後台的設定會一起搬過來

後台 Settings 頁面的設定（寄信、reCAPTCHA 等）存在資料庫裡，會跟著遷移。因為 `APP_ENVIRONMENT_ONLY` 是 `"false"`，這些設定會**覆蓋** ConfigMap 的值。

### 檔案與備份

遊戲伺服器的檔案和備份存在 Wings 主機上，遷移 Panel 不會影響它們。

## 疑難排解

| 狀況 | 原因與處理 |
| --- | --- |
| 匯入時出現 `Unknown collation: 'utf8mb4_0900_ai_ci'` | 舊主機是 MySQL 8。把排序規則換掉再匯入：`zcat panel-migration.sql.gz \| sed 's/utf8mb4_0900_ai_ci/utf8mb4_unicode_ci/g' \| kubectl ...` |
| 匯入時出現 `Access denied; you need (at least one of) the PROCESS privilege` | 舊主機是 MySQL 8，匯出時沒有加 `--no-tablespaces` |
| 登入時出現 `The MAC is invalid` | `APP_KEY` 不正確：檢查是否少了 `base64:` 前綴、多了引號，並比對 sha256 |
| 伺服器網址的 ID 跟舊 Panel 不同 | `HASHIDS_SALT` 或 `HASHIDS_LENGTH` 不正確 |
| 演練時登入出現 `An unexpected error was encountered` | reCAPTCHA 被 egress 封鎖，照[演練](#匯出並匯入)關閉 reCAPTCHA |
| log 出現一連串 migration，而不是 `Nothing to migrate` | 新舊 Panel 的版本不同，資料庫已經被升級 |
| 切換後 Node 都是紅色 | Wings 連不到新 Panel：確認 DNS、憑證，以及演練用的 NetworkPolicy 已經刪除 |