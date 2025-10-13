# 5. Deploymentによる高度なデプロイ戦略

`01-kubernetes-basics.md`では、DeploymentがReplicaSetを管理し、アプリケーションのデプロイを行うことを学びました。この章ではさらに踏み込み、Deploymentマニフェストの具体的な設定値や、アプリケーションを安全に更新するための様々な戦略（ストラテジー）について詳しく見ていきます。

ここでの知識は、ダウンタイムなしでアプリケーションを更新したり、問題発生時に素早く以前のバージョンに戻したりといった、実務で必須となるオペレーションの基礎となります。

---

### 5.1. Deploymentマニフェストの詳細

`01-kubernetes-basics.md`で使った`nginx-app.yaml`を元に、Deploymentマニフェストの主要なフィールドの役割を詳しく見ていきましょう。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec: # ★ここから下がDeploymentの仕様を定義する部分
  replicas: 2
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

#### ■ `spec.replicas`

このDeploymentが管理するPodの**目標数**を指定します。Kubernetesは、この数と実際のPod数を常に比較し、足りなければ増やし、多ければ減らすことで、Pod数を維持します。このPod数の維持を直接担当しているのが、Deploymentによって作られる**ReplicaSet**です。

#### ■ `spec.selector`

このDeploymentが**どのPodを管理対象とするか**を定義する非常に重要なフィールドです。

*   `matchLabels`: ここで指定されたラベル（例: `app: my-nginx`）を持つPodを、このDeploymentの管理対象とみなします。

`selector`は、DeploymentとPod（を管理するReplicaSet）を結びつける**接着剤**の役割を果たします。

#### ■ `spec.template`

ここには、このDeploymentが新しく作成するPodの**設計図（テンプレート）** を記述します。`template`以下の構造は、`kind: Pod`のマニフェストとほぼ同じです。

*   `metadata.labels`: Podに付けるラベルを定義します。**非常に重要な点として、ここのラベルは必ず`spec.selector.matchLabels`と一致している必要があります。** もし一致していないと、Deploymentは自分が作ったPodを管理対象として認識できず、無限にPodを作り続けてしまうなどの問題が発生します。
*   `spec`: Podの仕様を定義します。コンテナの種類（`containers`）、使用するイメージ（`image`）、公開するポート（`ports`）などを指定します。

---

### 5.2. アップデート戦略 (Strategy)

Deploymentの最も強力な機能の一つが、アプリケーションのバージョンアップを安全に行うための**アップデート戦略**です。`spec.strategy`フィールドで、どのようにPodを入れ替えるかを指定できます。

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate # "RollingUpdate" または "Recreate" を指定
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  ...
```

#### ■ `type: RollingUpdate` (デフォルト)

`RollingUpdate`は、**サービスを停止することなく（ダウンタイムなしで）**、古いバージョンのPodを新しいバージョンのPodに一つずつ、または少しずつ入れ替えていく方式です。指定しない場合のデフォルトの戦略であり、最も一般的に使われます。

**仕組みのイメージ (`replicas: 3` の場合)**
1.  新しいバージョンのPodを1つ起動します (合計4 Pod)。
2.  新しいPodが起動完了したら、古いバージョンのPodを1つ停止します (合計3 Pod)。
3.  上記を繰り返し、すべてのPodが新しいバージョンに入れ替わったら完了です。

この入れ替えの細かい挙動は、`rollingUpdate`以下のパラメータで制御できます。

*   `maxSurge`: アップデート中に、`replicas`で指定された数を超えて**一時的に存在を許容するPodの最大数**。**整数値**（例: `1`）または**パーセンテージ**（例: `"25%"`）で指定できます。`replicas: 4`, `maxSurge: "25%"` の場合、`4 * 0.25 = 1` となり、一時的に最大 `4 + 1 = 5` 個のPodが存在できます（小数点以下は切り上げ）。
*   `maxUnavailable`: アップデート中に、**同時に利用不可能になってもよいPodの最大数**を定義します。言い換えると「古いPodを最大で何個まで同時にシャットダウンできるか」を指定します。こちらも**整数値**または**パーセンテージ**で指定できます。`replicas: 4`, `maxUnavailable: "25%"` の場合、`4 * 0.25 = 1` となり、最大1つのPodが同時に停止されることを意味します。これにより、常に最低 `4 - 1 = 3` 個のPodがリクエストを処理できる状態が維持されます（小数点以下は切り捨て）。

**利点**:
*   **ゼロダウンタイム**: 常にリクエストを処理できるPodが存在するため、ユーザー影響がありません。
*   **安全な切り替え**: 問題があればすぐに中断・ロールバックできます。

#### ■ `type: Recreate`

`Recreate`は、非常にシンプルな戦略です。まず**古いバージョンのPodをすべて停止**してから、**新しいバージョンのPodをすべて起動**します。

**仕組みのイメージ (`replicas: 3` の場合)**
1.  古いバージョンのPodを3つすべて停止します。
2.  新しいバージョンのPodを3つすべて起動します。

**利点**:
*   **シンプル**: アプリケーションの古いバージョンと新しいバージョンが同時に存在しないため、構成が単純です。
*   **データ移行などに有効**: 新旧バージョンが同時にDBスキーマなどを触ると問題が起きる場合に有効です。

**欠点**:
*   **ダウンタイムが発生する**: 古いPodを停止してから新しいPodが起動するまでの間、サービスが完全に停止します。

👉 **使い分け**:
基本的には、ユーザーへの影響がない`RollingUpdate`を常に第一候補とします。`Recreate`は、ダウンタイムが許容されるバッチ処理や、アーキテクチャ上の制約で新旧バージョンを同時に動かせない、といった特殊な場合にのみ使用を検討します。

---

### 5.3. アップデートの操作と確認 (`kubectl rollout`)

Deploymentのアップデート状況を確認したり、問題があった場合に以前のバージョンに戻したり（ロールバック）するには、`kubectl rollout`コマンド群を使用します。

#### ■ アップデートの仕組みを覗いてみる

Deploymentの`spec.template.spec.containers[0].image`を新しいバージョンに書き換えて`kubectl apply`すると、アップデートが始まります。このとき、裏側では何が起きているのでしょうか。

```bash
# Deploymentによって管理されているReplicaSetを確認
kubectl get replicaset
```

**実行結果（アップデート中）**
```
NAME                           DESIRED   CURRENT   READY   AGE
my-nginx-deployment-564f776b9c   2         2         2       10m  <-- 新しいReplicaSet
my-nginx-deployment-66b6c48dd5   1         1         1       25m  <-- 古いReplicaSet
```
このように、Deploymentは新しいバージョンの**新しいReplicaSetを作成**し、古いReplicaSetのPod数を徐々に減らし、新しいReplicaSetのPod数を徐々に増やしていくことで、ローリングアップデートを実現しています。

#### ■ アップデート状況の確認

`rollout status`を使うと、アップデートが完了するまでその進捗をリアルタイムで監視できます。

```bash
kubectl rollout status deployment/my-nginx-deployment
```

**実行結果**
```
Waiting for deployment "my-nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "my-nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "my-nginx-deployment" rollout to finish: 2 out of 2 new replicas have been updated...
deployment "my-nginx-deployment" successfully rolled out
```
CI/CDパイプラインなどで、デプロイが完了したことを待ってから次の処理に進みたい場合などに非常に便利です。

#### ■ 更新履歴の確認

`rollout history`を使うと、これまでのアップデート履歴（リビジョン）の一覧を確認できます。

```bash
kubectl rollout history deployment/my-nginx-deployment
```

**実行結果**
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
`kubectl apply`時に`--record`オプションを付けるか、マニフェストに`kubernetes.io/change-cause`アノテーションを付けると、`CHANGE-CAUSE`欄に更新理由を記録できます。

#### ■ ロールバック（バージョンの切り戻し）

`rollout undo`を使うと、デプロイしたバージョンに問題があった場合に、**一つ前の安定したバージョンに素早く切り戻す**ことができます。

```bash
# 一つ前のリビジョンに戻す
kubectl rollout undo deployment/my-nginx-deployment
```

**実行結果**
```
deployment.apps/my-nginx-deployment rolled back
```
このコマンドを実行すると、Deploymentは一つ前のリビジョン（例えばリビジョン1）のReplicaSetをスケールアップし、現在のリビジョン（リビジョン2）のReplicaSetをスケールダウンさせることで、安全にロールバックを行います。

特定のバージョンに戻したい場合は、`--to-revision`オプションでリビジョン番号を指定します。

```bash
# リビジョン1に戻す
kubectl rollout undo deployment/my-nginx-deployment --to-revision=1
```
