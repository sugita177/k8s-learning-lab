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


