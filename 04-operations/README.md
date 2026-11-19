# 04-operations: 運用・セキュリティ・トラブルシューティング実践

Kubernetes の本番運用を見据えた、障害切り分け、リソース管理（requests/limits・OOMKilled）、セキュリティ（RBAC・SecurityContext）、Kustomize による環境別マニフェスト管理を学ぶディレクトリです。

---

## Step 4-1: トラブルシューティング道場

### 1. 4大障害の症状別・切り分けチートシート

| 症状 / ステータス | まず叩くべきコマンド | 主な原因と確認ポイント |
| :--- | :--- | :--- |
| **`ImagePullBackOff`**<br>(`ErrImagePull`) | `kubectl describe pod` | **コンテナイメージの取得失敗**。`Events` 欄でイメージ名・タグのタイプミス、プライベートレジストリの認証エラー（`imagePullSecrets` 漏れ）を確認する。 |
| **`CrashLoopBackOff`**<br>(`Error`) | `kubectl logs` (`--previous`) | **アプリ起動後の異常終了ループ**。コンテナ自体は一度起動しているため、アプリ自身の標準出力・エラー出力（環境変数漏れ、コマンドミス等）を確認する。 |
| **`Pending`** | `kubectl describe pod` | **ノードへのスケジュール不能**。`Events` 欄の `FailedScheduling` を見て、CPU/メモリ不足（`Insufficient cpu/memory`）や `nodeSelector` 不一致、PVC 未バインドを確認する。 |
| **`Running` なのに通信不可** | `kubectl get ep` (`endpoints`) | **Service と Pod の紐付け失敗**。`ENDPOINTS` が空の場合、Service の `selector` と Pod の `labels` の不一致、または `readinessProbe` の失敗を疑う。 |

### 2. 実践検証した 4 つの障害ケース
1. **Case 1 ([01-trouble-case1.yaml](./01-trouble-case1.yaml))**: 存在しないイメージタグ `nginx:1.27-nonexistent` による `ImagePullBackOff`
2. **Case 2 ([02-trouble-case2.yaml](./02-trouble-case2.yaml))**: 必須環境変数 `DATABASE_URL` の欠如による起動即死（`Error` / `CrashLoopBackOff`）
3. **Case 3 ([03-trouble-case3.yaml](./03-trouble-case3.yaml))**: 過剰なリソース要求（CPU 64 / Memory 512Gi）による `FailedScheduling`（`Pending`）
4. **Case 4 ([04-trouble-case4.yaml](./04-trouble-case4.yaml))**: Deployment ラベル（`web-case4`）と Service セレクタ（`web-case-4`）の不一致による `Endpoints` 空問題

---

## Step 4-2: リソース管理（`requests` / `limits`、QoS クラス、`OOMKilled`）

### 1. 学んだ概念
- **リソース制限が必須な理由（Noisy Neighbor 問題）**:
  - 1つの暴走コンテナ（メモリリーク等）がノードの資源を食い潰し、同居している他の Pod やノード全体を道連れにクラッシュさせるのを防ぐ。
- **`requests`（最低保証・予約枠）と `limits`（最大上限）**:
  - **`requests`**: スケジューラがノードへ配置する際の予約枠。空きがなければ `Pending`（`FailedScheduling`）となる。
  - **`limits`**: コンテナが使用できる上限値。
    - **CPU 超過時**: コンテナは終了せず、処理速度にブレーキがかかる（**CPU スロットリング**）。
    - **Memory 超過時**: Linux カーネル（OOM Killer）により即座に強制終了される（**`OOMKilled` / `Exit Code: 137` (`128 + 9 SIGKILL`)**）。
- **QoS（Quality of Service）クラス**:
  - ノードのメモリ逼迫時にどの Pod から追い出すかを決める 3 段階の優先度。
  - **`Guaranteed`**（`requests == limits`: 最優先で保護） > **`Burstable`**（`requests < limits`: 標準） > **`BestEffort`**（未設定: 真っ先に追い出される）。

### 2. 実行したコマンドと検証結果
```bash
# 1. 上限 64Mi に対し 150MB 確保する Pod の適用
kubectl apply -f 05-oom-demo.yaml

# 2. OOMKilled と Exit Code 137 / QoS Class の確認
kubectl get pod oom-demo-pod -n k8s-lab
# => STATUS: OOMKilled

kubectl describe pod oom-demo-pod -n k8s-lab
# => State: Terminated / Reason: OOMKilled / Exit Code: 137
# => QoS Class: Burstable
```

### 3. 作成したマニフェスト
- [05-oom-demo.yaml](./05-oom-demo.yaml)


