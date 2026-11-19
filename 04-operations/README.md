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
