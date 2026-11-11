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
- [00-namespace.yaml](file:///Users/moriaki/workspace/k8s-learning-lab/01-basics/00-namespace.yaml)

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
- [01-pod.yaml](file:///Users/moriaki/workspace/k8s-learning-lab/01-basics/01-pod.yaml)



