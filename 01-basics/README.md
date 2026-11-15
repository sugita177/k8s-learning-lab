# 01-basics: 基礎リソースと通信

Kubernetes の最も基礎となるリソースと、クラスタ内通信の仕組みを学ぶディレクトリです。

---

## Step 1-1: クラスタ接続確認と Namespace による論理分離

### 1. 学んだ概念
- **Kubernetes クラスター**:
  - 複数台のコンピュータ（ノード）を束ねて1台の仮想的な巨大マシンとして動かす仕組み。
  - **コントロールプレーン（司令塔）**: 全体の監視とスケジューリング・指示出しを行う。
  - **ワーカーノード（作業員）**: 指示を受けて実際にコンテナを動かす。
- **Context（コンテキスト）**:
  - `kubectl` が「どのクラスタの、どのユーザーで、どの空間を操作しているか」の設定セット。
- **Namespace（名前空間）**:
  - 1つのクラスタを仮想的な「部屋」に論理分割するリソース。
  - `kube-system`（システム用）や `default`（標準）などがあるが、実務ではプロジェクトや環境ごとに Namespace を切って運用する。
- **命令的（Imperative）と宣言的（Declarative）**:
  - `create`（命令的）: 「作れ」という一度限りの指示。すでに存在するとエラーになる。
  - `apply`（宣言的）: 「クラスタをこの YAML の状態にせよ」という指示。何回実行しても同じ結果に収束する（冪等性）。
- **`last-applied-configuration` アノテーションとステータス変化**:
  - `create` で作ったリソースに対して後から `kubectl apply` を行うと、前回の適用履歴がないため Warning と共に `configured`（自動パッチ）される。
  - 差分がない状態で再度 `apply` すると `unchanged`（変更なし）となる。

### 2. 実行したコマンドと検証結果
```bash
# 接続中のコンテキスト確認
kubectl config current-context

# ノード（サーバー）の状態確認
kubectl get nodes

# Namespace 一覧の確認
kubectl get namespaces
# 短縮形
kubectl get ns

# 1. 命令的に作成
kubectl create namespace k8s-lab

# 2. 宣言的に YAML を適用（初回: アノテーション付与の Warning と configured）
kubectl apply -f 00-namespace.yaml
# => Warning: resource namespaces/k8s-lab is missing the kubectl.kubernetes.io/last-applied-configuration annotation...
# => namespace/k8s-lab configured

# 3. 再度同じ YAML を適用（差分なしのため unchanged を確認）
kubectl apply -f 00-namespace.yaml
# => namespace/k8s-lab unchanged
```

### 3. 作成したマニフェスト
- [00-namespace.yaml](./00-namespace.yaml)

---

## Step 1-2: 最小単位 Pod の作成とライフサイクル

### 1. 学んだ概念
- **Pod（ポッド）とは**:
  - Kubernetes における最小のデプロイ単位（コンテナを直接デプロイするのではなく、Pod というカプセルに包む）。
  - 1つの Pod 内に複数のコンテナ（運命共同体のメインコンテナとサイドカー等）を同居させることができ、Pod 内のコンテナ同士は同一 IP アドレスと `localhost` を共有して通信できる。
- **マニフェストの4大基本要素**:
  - `apiVersion`: リソースの API バージョン（Pod は `v1`）
  - `kind`: リソース種別（`Pod`）
  - `metadata`: 名前（`name`）や所属する部屋（`namespace`）
  - `spec`: あるべき仕様（コンテナイメージ、ポート番号等）
- **Namespace によるアクセスの壁**:
  - `kubectl get pods` はデフォルトで `default` 名前空間しか見ないため、別 Namespace のリソース確認には `-n <namespace>` の指定が必須。
- **Pod へのアクセスと閉じたネットワーク**:
  - Pod はクラスタ内部の仮想ネットワーク上にあり、そのままでは外部（ホスト Mac）から直接アクセスできない。
  - 開発・デバッグ時は `kubectl port-forward` を使って一時的なトンネルを開通させて疎通確認を行う。
- **Pod 単体の限界（なぜ実務で Pod を直接使わないのか）**:
  - Pod は一時的な（ephemeral）存在であり、手動削除やノード障害等で停止しても自己修復（セルフヒーリング）されない。
  - 継続的な運用には、Pod を監視・管理する上位リソース（Deployment）が不可欠。

### 2. 実行したコマンドと検証結果
```bash
# 1. Pod マニフェストの適用
kubectl apply -f 01-pod.yaml

# 2. default 名前空間での確認（見つからないことを確認）
kubectl get pods
# => No resources found in default namespace.

# 3. 指定した名前空間（k8s-lab）での確認
kubectl get pods -n k8s-lab
# => my-first-pod   1/1   Running   0   69s

# 4. Pod の詳細（健康診断書・イベント履歴）の確認
kubectl describe pod my-first-pod -n k8s-lab
# => default-scheduler によるノード割り当て、kubelet によるイメージプルと起動イベントを確認

# 5. ポートフォワードによるローカル疎通確認（http://localhost:8080 で Nginx 画面確認）
kubectl port-forward pod/my-first-pod -n k8s-lab 8080:80

# 6. Pod の削除と限界の検証（自動復活しないことを確認）
kubectl delete pod my-first-pod -n k8s-lab
kubectl get pods -n k8s-lab
# => No resources found in k8s-lab namespace.
```

### 3. 作成したマニフェスト
- [01-pod.yaml](./01-pod.yaml)

---

## Step 1-3: Deployment と ReplicaSet（宣言的制御・セルフヒーリング・ローリングアップデート）

### 1. 学んだ概念
- **3層ピラミッド構造（親子関係）**:
  - **Deployment（司令官）**: アプリケーション全体の世代交代、ローリングアップデート、ロールバックを管理。
  - **ReplicaSet（現場監督）**: 指定された数（`replicas`）の Pod が常に稼働している状態を維持・監視。
  - **Pod（現場作業員）**: 実際にコンテナが動作する最小単位。
  - リソース名の家系図: `nginx-deployment` (Deploy) -> `nginx-deployment-<hash>` (RS) -> `nginx-deployment-<hash>-<random>` (Pod)
- **ラベル（Labels）とセレクタ（Selector）**:
  - `spec.selector.matchLabels` と `spec.template.metadata.labels` のキー・値が完全一致することで、監督対象の Pod を疎結合に識別・管理する。
- **セルフヒーリング（自己修復 / 調和ループ）**:
  - 「あるべき状態（Desired: 3）」と「現在の状態（Actual）」を常に比較（Reconciliation Loop）。Pod が手動削除や障害で死んでも、即座に新しい Pod を自動生成して理想数を維持する。
- **無停止スケーリング（スケールアウト・イン）**:
  - `replicas` の数値を変更して `kubectl apply` することで、稼働中の既存 Pod を停止させることなく即座に増減できる。
- **ローリングアップデート（無停止更新）**:
  - コンテナイメージ等を更新した際、新旧の ReplicaSet が協調して「新 Pod 起動 -> 旧 Pod 停止」を1台ずつ安全に行い、サービスを止めることなくバージョン更新を行う。
- **ロールバック（緊急巻き戻し）**:
  - 旧 ReplicaSet は `DESIRED: 0` の状態で保持されるため、万が一の障害時には `kubectl rollout undo` で即座に安全な旧バージョンへ切り戻しができる。

### 2. 実行したコマンドと検証結果
```bash
# 1. Deployment マニフェストの適用
kubectl apply -f 02-deployment.yaml

# 2. 3層構造（Deploy, RS, Pod）の一括確認
kubectl get deploy,rs,po -n k8s-lab

# 3. セルフヒーリングの検証（Pod を1つ削除し、即座に新 Pod が誕生することを確認）
kubectl delete pod nginx-deployment-7578bd675d-8rx7h -n k8s-lab
kubectl get pods -n k8s-lab
# => わずか数秒（AGE: 11s）の新 Pod が自動補充され 3 台を維持

# 4. スケーリングの検証（replicas: 5 に変更して apply）
kubectl apply -f 02-deployment.yaml
kubectl get pods -n k8s-lab
# => 既存の 3 台は停止せず、新しく 2 台が追加されて計 5 台へ即時拡張

# 5. ローリングアップデートの検証（image を nginx:1.27-alpine に変更）
kubectl apply -f 02-deployment.yaml
kubectl get rs -n k8s-lab
# => 新しい ReplicaSet が誕生し、旧 RS から新 RS へ 1 台ずつ交代する過渡期を確認

# 6. ロールバックの検証（緊急巻き戻し）
kubectl rollout undo deployment/nginx-deployment -n k8s-lab
kubectl get rs -n k8s-lab
# => 安全に逆回転し、旧 ReplicaSet（nginx:1.27）へ無停止で復帰
```

### 3. 作成したマニフェスト
- [02-deployment.yaml](./02-deployment.yaml)




