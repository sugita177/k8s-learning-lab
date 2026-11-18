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
