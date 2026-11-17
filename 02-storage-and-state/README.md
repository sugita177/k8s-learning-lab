# 02-storage-and-state: データの永続化とステート管理

Kubernetes におけるデータの永続化（PV / PVC）、ステートフルアプリケーション管理（StatefulSet）、バッチ処理（Job / CronJob）を学ぶディレクトリです。

---

## Step 2-1: コンテナの揮発性と PersistentVolume (PV) / PersistentVolumeClaim (PVC)

### 1. 学んだ概念
- **コンテナの揮発性（Ephemeral）**:
  - Pod 内のファイルシステムは使い捨てであり、Pod の削除や再作成が行われると保存したデータは完全に消失する。
  - クラスタ環境（複数ノード）では、単なるホストのローカルディレクトリマウントでは別ノードへの避難時にデータへアクセスできないため、Pod の寿命と切り離されたストレージ抽象化が必要。
- **PV / PVC / StorageClass の役割分担**:
  - **PV（PersistentVolume）**: クラスタ全体で共有される実際のストレージ実体（Namespace に属さないクラスタスコープ）。
  - **PVC（PersistentVolumeClaim）**: 特定の Namespace 内でアプリが「容量（例: `1Gi`）」と「アクセスモード（`ReadWriteOnce` 等）」を指定して提出する要求チケット。
  - **StorageClass**: PVC の要求に応じて裏側で自動的に PV をプロビジョニングする仲介役（OrbStack では `local-path` がデフォルト）。
- **アクセスモード（AccessModes）**:
  - `ReadWriteOnce (RWO)`: 1つのノードから読み書き可能（最も一般的なブロックストレージ）。
  - `ReadOnlyMany (ROX)`: 複数ノードから読み取り専用。
  - `ReadWriteMany (RWX)`: 複数ノードから同時に読み書き可能。
- **`WaitForFirstConsumer` バインディングモード**:
  - PVC を作成した直後は `Pending` 状態で待機し、実際にその PVC を使用する Pod が起動したタイミングで初めて PV が動的作成されて `Bound` になる仕組み。

### 2. 実行したコマンドと検証結果
```bash
# 1. コンテナ揮発性の検証（PVC なしの Pod でファイル作成 -> Pod 削除）
kubectl exec -n k8s-lab <旧Pod名> -- sh -c "echo '絶対に消えてはいけない売上データ' > /tmp/important-data.txt"
kubectl delete pod <旧Pod名> -n k8s-lab
kubectl exec -n k8s-lab <新Pod名> -- cat /tmp/important-data.txt
# => No such file or directory（データ消失を確認）

# 2. StorageClass の確認
kubectl get storageclass
# => local-path (default) / VOLUMEBINDINGMODE: WaitForFirstConsumer

# 3. PVC の適用と待機状態（Pending）の確認
kubectl apply -f 01-pvc.yaml
kubectl get pvc -n k8s-lab
# => STATUS: Pending
kubectl get pv
# => No resources found

# 4. PVC をマウントする Deployment の適用とバインド（Bound）の確認
kubectl apply -f 02-deployment-with-pvc.yaml
kubectl get pvc -n k8s-lab
# => STATUS: Bound, VOLUME: pvc-2b2d8f99-...
kubectl get pv
# => STATUS: Bound, CLAIM: k8s-lab/my-first-pvc

# 5. データ永続化の検証（/data へファイル書き込み -> Pod 削除 -> 復活後の読み込み）
kubectl exec -n k8s-lab <Pod名> -- sh -c "echo 'PV/PVCのおかげで生き残る売上データ' > /data/important-data.txt"
kubectl delete pod <Pod名> -n k8s-lab
kubectl exec -n k8s-lab <新Pod名> -- cat /data/important-data.txt
# => PV/PVCのおかげで生き残る売上データ（永続化に成功！）
```

### 3. 作成したマニフェスト
- [01-pvc.yaml](./01-pvc.yaml)
- [02-deployment-with-pvc.yaml](./02-deployment-with-pvc.yaml)

---

## Step 2-2: StatefulSet によるステートフルアプリケーション管理

### 1. 学んだ概念
- **Deployment（牛の群れ / ステートレス）vs StatefulSet（ペット / ステートフル）**:
  - **Deployment**: Pod 名はランダムで互換性あり。一斉起動・一斉削除。複数台で同じディスクを共有する用途には向かない。
  - **StatefulSet**: Pod 名は厳格な 0 始まりの連番（`web-stateful-0`, `-1`, `-2`）。起動は `0 → 1 → 2` の順、削除は `2 → 1 → 0` の逆順。
- **`volumeClaimTemplates` による個別ストレージ割り当て**:
  - StatefulSet の各 Pod に対して専用の PVC（例: `www-data-web-stateful-0`）が 1 対 1 で自動生成・紐付けされる。
  - Pod が削除されて再生成されても、同じ名前（番号）で復活し、必ず自分専用の元のディスクに再接続される。
- **Headless Service（`clusterIP: None`）**:
  - StatefulSet と連携する「固定 IP を持たない」特殊な Service（StatefulSet の中に包まれているのではなく、独立したリソースとして横並びで連携する）。
  - `web-stateful-0.web-headless` のように特定の番号の Pod をピンポイントで DNS 名前解決するために使用する。
- **リソース容量の単位表記**:
  - Kubernetes YAML では `GiB` ではなく `Gi`（2進数 1024 ベース）または `G`（10進数 1000 ベース）と表記する。

### 2. 実行したコマンドと検証結果
```bash
# 1. Headless Service と StatefulSet の適用
kubectl apply -f 03-statefulset.yaml

# 2. 順番起動（4秒ごとの時間差）と連番命名の確認
kubectl get pods -n k8s-lab -l app=web-stateful
# => web-stateful-0 (27s) -> web-stateful-1 (23s) -> web-stateful-2 (19s)

# 3. 個別 PVC の自動量産を確認
kubectl get pvc -n k8s-lab
# => www-data-web-stateful-0, -1, -2 の 3 つの PVC が個別に生成・Bound

# 4. 個性・永続性の検証（0号機と1号機に別データを書き込み、0号機を削除）
kubectl exec -n k8s-lab web-stateful-0 -- sh -c "echo 'I am Pod 0 (Primary)' > /usr/share/nginx/html/index.html"
kubectl exec -n k8s-lab web-stateful-1 -- sh -c "echo 'I am Pod 1 (Replica)' > /usr/share/nginx/html/index.html"
kubectl delete pod web-stateful-0 -n k8s-lab

# 5. 復活後の確認
kubectl get pods -n k8s-lab -l app=web-stateful
# => web-stateful-0 が AGE: 7s として同じ名前で復活
kubectl exec -n k8s-lab web-stateful-0 -- cat /usr/share/nginx/html/index.html
# => I am Pod 0 (Primary)（専用ディスクへ再接続されたことを確認）
```

### 3. 作成したマニフェスト
- [03-statefulset.yaml](./03-statefulset.yaml)

