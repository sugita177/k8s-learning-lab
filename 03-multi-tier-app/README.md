# 03-multi-tier-app: 複数コンテナ・多層アーキテクチャと信頼性設計

多層アーキテクチャの構築（Web + DB）、クラスタ内 DNS（CoreDNS）、ヘルスチェック（Probes）、Ingress ルーティングを学ぶディレクトリです。

---

## Step 3-1: Web + DB の2層アーキテクチャとクラスタ内 DNS（CoreDNS）

### 1. 学んだ概念
- **多層アーキテクチャと Service タイプの使い分け**:
  - **Web 層（フロント）**: 外部（ブラウザ）からアクセスできるように `LoadBalancer`（または Ingress）で公開する。
  - **DB 層（バックエンド）**: セキュリティ上、外部へ直接公開せず `ClusterIP` にしてクラスタ内部からのみ接続させる。
- **クラスタ内 DNS（CoreDNS）による名前解決**:
  - `kube-system` 名前空間で稼働している `coredns` が、クラスタ内の全 Service に自動で DNS レコードを登録する。
  - **FQDN（完全修飾ドメイン名）**: `<Service名>.<Namespace名>.svc.cluster.local`
  - 同一 Namespace 内からの通信であれば、`<Service名>` のみ（例: `postgres-svc`）で名前解決が可能。これにより、Pod の IP 変動に影響されない疎結合なアプリ連携が実現する。
- **Secret の `type: Opaque` と `envFrom`**:
  - `type: Opaque` は任意のキー・バリュー形式を格納する汎用的な Secret タイプ（デフォルト）。
  - `envFrom` を使うと、Secret や ConfigMap に定義されたすべてのキーを一括で環境変数としてコンテナへ注入できる。

### 2. 実行したコマンドと検証結果
```bash
# 1. kube-system 内の CoreDNS 稼働確認
kubectl get pods -n kube-system
# => coredns-..., local-path-provisioner-..., svclb-... が稼働中であることを確認

# 2. DB 層（Secret, PVC, Deployment, ClusterIP Service）のデプロイ
kubectl apply -f 01-postgres.yaml

# 3. Web 層（Adminer Deployment, LoadBalancer Service）のデプロイ
kubectl apply -f 02-adminer.yaml

# 4. ブラウザ（http://k8s.orb.local:8081）から Adminer へアクセス
# サーバ名に IP ではなく「postgres-svc」を指定して PostgreSQL へのログインに成功！
```

### 3. 作成したマニフェスト
- [01-postgres.yaml](./01-postgres.yaml)
- [02-adminer.yaml](./02-adminer.yaml)

---

## Step 3-2: ヘルスチェックの実装（`livenessProbe` / `readinessProbe` / `startupProbe`）

### 1. 学んだ概念
- **なぜ `STATUS: Running` だけでは不十分なのか**:
  - プロセスは生きているがフリーズ（デッドロック）して応答しない状態や、起動直後の準備中でまだリクエストを受けられない状態を検知するため、アプリケーションレベルの健康診断（Probe）が必要。
- **3種類の Probe の役割と失敗時のアクション**:
  - **`livenessProbe`（生存確認）**: 「生きているか？」をチェック。失敗時は **コンテナを強制再起動（Killing → Restart）** して治療する。
  - **`readinessProbe`（受信準備確認）**: 「トラフィックを受け入れられるか？」をチェック。失敗時は再起動せず、**Service の Endpoints から切り離してトラフィックを遮断** する。
  - **`startupProbe`（起動完了確認）**: 起動が遅いアプリ用。これが成功するまで `liveness` と `readiness` のチェックを待機させる。
- **チェック方式**:
  - `httpGet`（HTTP 200 確認）、`tcpSocket`（ポート疎通確認）、`exec`（コマンド実行結果の終了コード確認）の 3 パターン。
- **主なチューニングパラメータ**:
  - `initialDelaySeconds`（初回チェックまでの待機秒数）、`periodSeconds`（チェック間隔）、`failureThreshold`（異常と判定する連続失敗回数）。

### 2. 実行したコマンドと検証結果
```bash
# 1. ヘルスチェック実験用マニフェストの適用
kubectl apply -f 03-probes-demo.yaml

# 2. readinessProbe 失敗実験（/tmp/ready を削除）
kubectl exec -n k8s-lab <Pod名> -- rm /tmp/ready
kubectl get pods -n k8s-lab -l app=probe-demo
# => READY: 0/1, STATUS: Running, RESTARTS: 0（再起動されず維持）
kubectl get ep probe-demo-svc -n k8s-lab
# => ENDPOINTS が空になり、トラフィックから安全に切り離されたことを確認

# 3. livenessProbe 失敗実験（/tmp/alive を削除）
kubectl exec -n k8s-lab <Pod名> -- rm /tmp/alive
kubectl describe pod <Pod名> -n k8s-lab | tail -n 15
# => Events に "Readiness probe failed" の後に
#    "Container nginx failed liveness probe, will be restarted" (Killing -> Created -> Started) が記録され自動復旧を確認
```

### 3. 作成したマニフェスト
- [03-probes-demo.yaml](./03-probes-demo.yaml)

---

## Step 3-3: Ingress による L7 ルーティング（パス・ホストベース振り分け）

### 1. 学んだ概念
- **LoadBalancer Service の課題と Ingress の役割**:
  - マイクロサービスごとに `LoadBalancer` を作るとコスト（外部 IP・LB 課金）が爆発し、L4（IP/Port）のみのため URL パスやドメイン名による振り分けができない。
  - **Ingress** を使うことで、外部への入口（LoadBalancer）を 1 つに集約し、HTTP（L7）のパスやホスト名に応じて背後の無料な `ClusterIP` Service へルーティングできる。
- **Ingress（ルール表）と Ingress Controller（実行役）の分離**:
  - `Ingress` マニフェスト単体では動作せず、クラスタ内に `NGINX Ingress Controller` などの実行役（`IngressClass`）が必要。
- **`annotations` と `rewrite-target`**:
  - `annotations` はコントローラー固有のオプション設定を記述するメタデータ。
  - `nginx.ingress.kubernetes.io/rewrite-target: /` により、外向きのパス（`/hello`, `/api`）を受け取った後、バックエンドのアプリへ転送する直前に `/` へ書き換えて渡す（アプリ側の 404 エラーを防ぐ）。

### 2. 実行したコマンドと検証結果
```bash
# 1. IngressClass の有無確認（初期状態はなし）
kubectl get ingressclass

# 2. NGINX Ingress Controller のインストールと稼働確認
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml
kubectl get ingressclass
# => nginx (k8s.io/ingress-nginx) が登録
kubectl get pods -n ingress-nginx
# => ingress-nginx-controller が 1/1 Running

# 3. 振り分け先アプリ2つ（ClusterIP）と Ingress ルールの適用
kubectl apply -f 04-ingress-apps.yaml
kubectl apply -f 05-ingress.yaml

# 4. 同一入口・同一ポート（80番）へのパスベースルーティング確認
curl http://k8s.orb.local/hello
# => Hello from App 1 (Frontend)!
curl http://k8s.orb.local/api
# => Response from App 2 (Backend API)!
```

### 3. 作成したマニフェスト
- [04-ingress-apps.yaml](./04-ingress-apps.yaml)
- [05-ingress.yaml](./05-ingress.yaml)


