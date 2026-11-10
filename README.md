# Kubernetes Learning Lab (`k8s-learning-lab`)

macOS + OrbStack 環境を用いて、Kubernetes の基礎概念から実践的なマニフェスト設計・運用・トラブルシューティングまでを体系的に学ぶハンズオンリポジトリです。

## 環境要件
- **OS**: macOS (Apple Silicon)
- **Kubernetes 環境**: [OrbStack](https://orbstack.dev/) (Built-in Kubernetes Cluster)
- **CLI ツール**: `kubectl`

## ディレクトリ構成（予定）
各フェーズ・テーマごとにディレクトリを分け、検証用マニフェストと学習ログ（`README.md`）を格納します。

```text
k8s-learning-lab/
├── 01-basics/              # Phase 1: Namespace, Pod, Deployment, Service, ConfigMap, Secret
├── 02-storage-and-state/   # Phase 2: PV/PVC, StatefulSet, Job/CronJob
├── 03-multi-tier-app/      # Phase 3: Web+DB連携, CoreDNS, Probes, Ingress
├── 04-operations/          # Phase 4: Troubleshooting, Resource管理, RBAC, Kustomize
└── README.md
```

## 学習ロードマップ

### Phase 1: 基礎リソースと通信（DockerからKubernetesへのパラダイムシフト）
- **Step 1-1: クラスタ接続確認と Namespace による論理分離**
  - コンテキストの確認、リソースの論理的な境界線（Namespace）の理解と作成
- **Step 1-2: 最小単位 Pod の作成とライフサイクル**
  - なぜコンテナ直接ではなく「Pod」なのか、命令的（Imperative）と宣言的（Declarative）の違い
- **Step 1-3: Deployment と ReplicaSet**
  - セルフヒーリング（自己修復）、スケーリング、ローリングアップデートとロールバック
- **Step 1-4: Service によるネットワーク抽象化**
  - 使い捨てられる Pod IP の課題解決、`ClusterIP` と OrbStack の `LoadBalancer` 連携
- **Step 1-5: ConfigMap と Secret**
  - コンテナイメージと設定・機密情報の分離（環境変数注入・Volumeマウント）

### Phase 2: データの永続化とステート管理
- **Step 2-1: PersistentVolume (PV) と PersistentVolumeClaim (PVC)**
  - コンテナ再起動時のデータ消失確認、StorageClass を用いた動的プロビジョニング
- **Step 2-2: StatefulSet によるステートフル管理**
  - Deployment との設計思想の違い、安定したネットワーク識別子と個別ストレージの維持
- **Step 2-3: Job と CronJob**
  - 常駐しないワークロード（1回限りのバッチ処理・定期実行タスク）の管理

### Phase 3: 複数コンテナ・多層アーキテクチャと信頼性設計
- **Step 3-1: Web + DB の2層構成連携とクラスタ内 DNS**
  - CoreDNS による Service 名での名前解決、複数コンポーネントの疎結合な連携
- **Step 3-2: ヘルスチェック（Probes）の実装**
  - `livenessProbe`（死活監視と再起動）と `readinessProbe`（トラフィック受信可否制御）の使い分け
- **Step 3-3: Ingress による L7 ルーティング**
  - HTTP/HTTPS レイヤーでのホスト・パスベースルーティング

### Phase 4: 運用・セキュリティ・トラブルシューティング実践
- **Step 4-1: トラブルシューティング実践**
  - `CrashLoopBackOff`、`ImagePullBackOff`、`Pending`、Service 通信不全の原因特定アプローチ（`describe` / `logs` / `events`）
- **Step 4-2: リソース管理と QoS**
  - CPU/Memory の `requests` と `limits`、リソース枯渇時の挙動（CPU スロットリングと `OOMKilled`）
- **Step 4-3: セキュリティと RBAC（ロールベースアクセス制御）**
  - `ServiceAccount`、`Role`、`RoleBinding` による最小権限の原則、`SecurityContext` による非 root 実行
- **Step 4-4: Kustomize によるマニフェスト管理**
  - ベースマニフェストとオーバーレイ（`dev` / `prod`）を用いた DRY な構成管理

