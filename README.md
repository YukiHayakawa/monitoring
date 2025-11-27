# Monitoring Stack

Prometheus、Loki、Grafana をセットにした監視スタックです。`edge`（nginx）で社内 DNS / `jwilder/nginx-proxy` からの HTTPS 要求を受け、各サービスにプロキシします。テンプレート化された設定を使って最短で監視基盤を立ち上げられることを目的にしています。

## 構成

| サービス       | 役割                                          | Exposed ポート                | 備考                                                 |
| -------------- | --------------------------------------------- | ----------------------------- | ---------------------------------------------------- |
| Prometheus     | メトリクス収集（node-exporter / cAdvisor 等） | `${PROMETHEUS_INTERNAL_PORT}` | `prometheus/prometheus.yml` をベースにカスタム       |
| Loki           | ログ集約（promtail などから Push）            | `${LOKI_INTERNAL_PORT}`       | `loki/local-config.yml`                              |
| Grafana        | ダッシュボード可視化                          | `${GRAFANA_INTERNAL_PORT}`    | `grafana/provisioning/**` で自動プロビジョニング     |
| Node Exporter  | Docker ホストの OS メトリクス収集             | `${NODE_EXPORTER_PORT}`       | `/proc` / `/sys` をマウントしてホスト情報を収集      |
| cAdvisor       | Docker ホスト上のコンテナメトリクス収集       | `${CADVISOR_PORT}`            | Docker デーモンの統計を公開                          |
| Edge(Nginx)    | HTTPS 終端／リバースプロキシ                  | `${EDGE_PORT}` (expose のみ)  | jwilder/nginx-proxy と同じ `proxy-net` に接続        |

## 必要条件

- Docker / Docker Compose v2
- 監視スタック用ホストで `proxy-net`（jwilder/nginx-proxy と共有する外部ネットワーク）が作成済み
- 監視対象サーバーの node-exporter / cAdvisor / NGINX ログ転送が可能
- `BASE_DOMAIN` 配下で `local.prometheus.…` などの DNS が解決できること（社内 DNS or hosts ファイル）

## セットアップ

1. **リポジトリ取得**
   ```bash
   git clone <this-repo>
   cd monitoring
   ```
2. **環境変数ファイルを準備**

   ```bash
   cp .env.sample .env
   ```

   `.env` 内で特に必要な項目:

   | 変数                                                           | 説明                                                                      |
   | -------------------------------------------------------------- | ------------------------------------------------------------------------- |
   | `ENV_PREFIX` / `BASE_DOMAIN`                                   | `local.` や `tool.example.com` など環境別のホスト名を組み立てるために使用 |
   | `PROMETHEUS_HOST` / `LOKI_HOST` / `GRAFANA_HOST` / `EDGE_HOST` | 実際に jwilder/nginx-proxy に渡す完全修飾ホスト名                         |
   | `PROMETHEUS_INTERNAL_PORT` ほか                                | 各サービスのコンテナ内ポート。基本的にはデフォルトのままで OK             |
   | `GF_SECURITY_ADMIN_*`                                          | Grafana 初期ログイン情報                                                  |
   | `NODE_EXPORTER_*` / `CADVISOR_*`                               | ホストメトリクス用サービスのコンテナ名・ポート                            |

3. **Prometheus のスクレイプ設定**

   - `prometheus/prometheus.yml` にはデフォルトで `prometheus` / `node-exporter` / `cadvisor` / `nginx-logs` のジョブが定義されています。Docker ホスト以外のメトリクスが必要な場合はここにジョブを追加してください。

4. **Loki・Grafana の設定**

   - Loki: `loki/local-config.yml` にリテンションなどのオプションを記述しています。必要に応じて S3 連携や永続化ボリュームを追記してください。
   - Grafana: `grafana/provisioning/datasources/datasources.yml` に Prometheus / Loki のデータソースが環境変数ベースで登録されます。追加のデータソースを作りたい場合はここに追記します。

5. **外部ネットワークを作成（未作成の場合）**
   ```bash
   docker network create proxy-net
   ```

## 使い方

1. **起動**

   ```bash
   # Loki の初回起動時は Tempslate を読み込むので少し時間がかかります
   docker compose up -d
   ```

2. **状態確認**

   ```bash
   docker compose ps
   docker compose logs -f loki
   docker compose logs -f edge
   ```

   `loki` コンテナは `/ready` ヘルスチェックに通らないと `edge` が起動しない設計です。`STATUS` が `healthy` になるまで待ってください。

3. **アクセス**

   - `https://local.grafana.yourdomain.com` など `.env` で設定したホスト名をブラウザで開きます。リクエストは外側の `jwilder/nginx-proxy` → `edge` → 各サービスに流れます。
   - Grafana ログイン: `.env` の `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`
   - Prometheus / Loki も同じドメインパターン（`local.prometheus.…`, `local.loki.…`）でアクセスできます。

4. **監視対象を増やす**

   - `prometheus/prometheus.yml` 内にジョブを追加します。ホストメトリクス用の `node-exporter` / `cadvisor` は Compose に同梱されているため、Docker ホスト自身は追加作業なしで取得できます。
   - promtail から Loki へ送る場合は各アプリサーバーに promtail を配置し、`clients[].url` に `http(s)://<EDGE_HOST>/loki/api/v1/push` を指定してください。

5. **停止 / クリーンアップ**
   ```bash
   docker compose down
   # ボリュームも消す場合
   docker compose down -v
   ```

## よくある問題と対処

- **`host not found in upstream "loki"`**  
  Loki コンテナが `proxy-net` に参加していないか、ヘルスチェックに通っていません。`docker compose ps` で `loki` が `healthy` になっているか確認し、必要なら `docker compose up -d loki` → `docker compose up -d edge` の順で再起動してください。

- **HTTPS でアクセスできない**  
  `edge` の `environment.VIRTUAL_HOST` が未解決だと jwilder/nginx-proxy が vhost を生成できません。.env で実際のホスト名を指定し、`jwilder/nginx-proxy` と同じ `proxy-net` を使っているか確認します。

- **Grafana にデータソースが出てこない**  
  `grafana/provisioning/datasources/datasources.yml` の環境変数展開は Grafana 起動時に行われます。`.env` の `PROMETHEUS_INTERNAL_PORT` 等が正しいか確認し、Grafana を再起動してください。

## ディレクトリ構成

```
monitoring/
├─ .env / .env.sample
├─ docker-compose.yml
├─ edge/
│   ├─ nginx/conf.d/*.conf          # 実際にコンテナで使用される設定
│   └─ nginx/templates/*.template   # envsubst で生成するテンプレート
├─ grafana/
│   ├─ data/                        # 永続化データ（初回空でOK）
│   └─ provisioning/datasources/
├─ loki/local-config.yml
└─ prometheus/
    ├─ prometheus.sample.yml         # 参考用の設定
    └─ prometheus.yml
```

この README をベースに、チームの事情に合わせて各設定を拡張してください。
