# 2. リソースの状態を確認する (Observing Resources)

`01-kubernetes-basics.md`では、Kubernetesの基本的な概念とアプリケーションの動かし方を学びました。ここでは、クラスタ上で動いているリソース（PodやDeploymentなど）が「今、どのような状態なのか」を正確に把握するためのコマンドを学びます。

何か問題が起きたとき、まず最初に行うのがこの「状態確認」です。これから紹介するコマンドを使いこなせることが、Kubernetesを扱う上での第一歩となります。

---

### 2.1. リソースを一覧表示する `kubectl get`

`kubectl get`は、指定したリソースの一覧を表示するための最も基本的なコマンドです。Podが動いているか、Serviceは作られているかなど、様々な情報を取得できます。

#### ■ 基本的な使い方

```bash
# Podの一覧を表示する
kubectl get pods

# Deploymentの一覧を表示する
kubectl get deployments

# Serviceの一覧を表示する
kubectl get services

# すべてのリソースを表示する（Namespace内の）
kubectl get all
```

**実行結果の見方 (podsの場合)**

```
NAME                                   READY   STATUS    RESTARTS   AGE
my-nginx-deployment-66b6c48dd5-abcde   1/1     Running   0          2m10s
my-nginx-deployment-66b6c48dd5-fghij   1/1     Running   0          2m10s
```

*   **NAME**: Podの名前です。Deploymentによって作られたPodは、Deployment名の後ろにランダムな文字列が付きます。
*   **READY**: `1/1`のように表示され、「Pod内に定義されたコンテナ数 / 起動準備が完了したコンテナ数」を意味します。ここが`1/1`なら正常です。
*   **STATUS**: Podの現在の状態です。`Running`は正常に動作中であることを示します。他に`Pending`（起動準備中）、`Completed`（正常終了）、`Error`（異常終了）などがあります。
*   **RESTARTS**: Podが再起動した回数です。この回数が多い場合、コンテナ内で何らかの問題が発生している可能性があります。
*   **AGE**: Podが作成されてからの経過時間です。

---

#### ■ オプションでもっと詳しく見る

`kubectl get`には、表示する情報をカスタマイズするための便利なオプションがあります。

##### **-o wide: より詳細な情報を表示する**

`-o wide` (`--output wide`の略) を付けると、標準の表示に加えて、Podがどのワーカーノードで動いているか(`NODE`)や、Podに割り当てられたIPアドレス(`IP`)などを確認できます。

```bash
kubectl get pods -o wide
```

**実行結果**

```
NAME                                   READY   STATUS    RESTARTS   AGE     IP           NODE
my-nginx-deployment-66b6c48dd5-abcde   1/1     Running   0          5m30s   10.244.1.2   kind-worker
my-nginx-deployment-66b6c48dd5-fghij   1/1     Running   0          5m30s   10.244.2.3   kind-worker2
```

👉 **使いどころ**:
複数のPodが、意図した通りに異なるワーカーノードに分散して配置されているか確認したい場合や、PodのIPアドレスを直接知りたい場合などに非常に役立ちます。

##### **-o yaml: マニフェスト形式で全情報を表示する**

`-o yaml` (`--output yaml`の略) を付けると、そのリソースの**すべての設定情報**を、私たちが普段書いているマニフェストと同じYAML形式で表示します。

```bash
# 特定のPodの情報をYAML形式で表示
kubectl get pod <Podの名前> -o yaml
```

**実行結果（一部抜粋）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-deployment-66b6c48dd5-abcde
  namespace: default
  labels:
    app: my-nginx
    pod-template-hash: 66b6c48dd5
  ...
spec:
  containers:
  - image: nginx:latest
    name: nginx-container
    ports:
    - containerPort: 80
      protocol: TCP
  ...
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-01-01T12:00:00Z"
    status: "True"
    type: Initialized
  ...
  containerStatuses:
  - containerID: containerd://...
    image: docker.io/library/nginx:latest
    ready: true
    restartCount: 0
    started: true
  ...
  hostIP: 172.18.0.3
  podIP: 10.244.1.2
  ...
```

この中には、私たちがマニフェストに書いた設定(`spec`)だけでなく、Kubernetesが自動で追加した情報（ラベルやIPアドレスなど）や、現在のリソースの状態(`status`)も含まれています。

👉 **使いどころ**:
*   「適用したはずのマニフェストの設定が、正しく反映されているか」を正確に確認したい場合。
*   リソースがどのような状態で動いているのか、詳細な生データを確認したい場合。
*   既存のリソースを元に、新しいマニフェストを作成したい場合（この出力をコピーして編集するなど）。

##### **--v: kubectlコマンド自体のログレベルを変更する**

`--v`オプションは、`kubectl`コマンドがAPIサーバと通信する際の、より詳細なログを出力させたいときに使います。数字が大きいほど、より詳細な情報が表示されます（通常は6〜8がよく使われます）。

```bash
# Podを取得する際の通信ログを詳細に表示する
kubectl get pods --v=8
```

**実行結果（一部抜粋）**

```
I0101 12:05:00.123456   12345 loader.go:372] Config loaded from file: /home/user/.kube/config
I0101 12:05:00.234567   12345 round_trippers.go:445] GET https://<APIサーバのアドレス>/api/v1/namespaces/default/pods 200 OK in 10 milliseconds
...
```

このように、どの設定ファイルを読み込み、どのAPIエンドポイントにリクエストを送り、どのような結果が返ってきたか、といった内部的な動作が詳細に表示されます。

👉 **使いどころ**:
*   `kubectl`コマンド自体がなぜかうまく動かない、エラーになる、といった場合に、その原因を調査するためのデバッグ情報として使います。
*   普段の運用で使うことは稀ですが、「`kubectl`の裏側ではAPIサーバとこんな通信が行われているんだな」という学習目的で見てみるのも面白いでしょう。

---

#### ■ 特定のリソースだけを指定して見る

リソースがたくさんある中で、特定の1つだけを見たい場合は、リソース名の後ろにその名前を指定します。

```bash
# "my-nginx-deployment-66b6c48dd5-abcde" という名前のPodだけを表示
kubectl get pod my-nginx-deployment-66b6c48dd5-abcde

# "my-nginx-service" という名前のServiceだけを表示
kubectl get service my-nginx-service
```

これにより、大量の情報の中から目的のリソースだけを素早く見つけることができます。

---

### 2.2. リソースの詳細情報を表示する `kubectl describe`

`kubectl get`がリソースの一覧を**表形式**でシンプルに表示するのに対し、`kubectl describe`は特定のリソースに関する**詳細な情報や関連イベント**を、人間が読みやすい形式で表示してくれます。

特に、Podがうまく起動しない（`Pending`や`Error`状態になる）など、問題が発生したときの原因調査に絶大な効果を発揮します。

#### ■ 基本的な使い方

```bash
# 特定のPodの詳細情報を表示
kubectl describe pod <Podの名前>

# 特定のDeploymentの詳細情報を表示
kubectl describe deployment <Deploymentの名前>
```

#### ■ 実行結果の見方とポイント

`kubectl describe pod <Pod名>` の実行結果は、いくつかのセクションに分かれています。特に重要なのが一番下の `Events` セクションです。

```
Name:         my-nginx-deployment-66b6c48dd5-abcde
Namespace:    default
Priority:     0
Node:         kind-worker/172.18.0.3
Start Time:   Tue, 01 Jan 2024 12:00:00 +0900
Labels:       app=my-nginx
              pod-template-hash=66b6c48dd5
Annotations:  <none>
Status:       Running
IP:           10.244.1.2
IPs:
  IP:  10.244.1.2
Containers:
  nginx-container:
    Container ID:   containerd://...
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:...
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 01 Jan 2024 12:00:05 +0900
    Ready:          True
    Restart Count:  0
    ...
Volumes:
  ...
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/my-nginx-deployment-66b6c48dd5-abcde to kind-worker
  Normal  Pulling    10m   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     10m   kubelet            Successfully pulled image "nginx:latest" in 4s
  Normal  Created    10m   kubelet            Created container nginx-container
  Normal  Started    10m   kubelet            Started container nginx-container
```

*   **基本情報**: `Name`, `Namespace`, `Node`など、Podの基本的な情報が表示されます。
*   **Containers**: Pod内で動いているコンテナの詳細情報です。使用しているDockerイメージ、ポート設定、現在の状態(`State`)などがわかります。
*   **Events**: **ここが最も重要です。** このリソースに関連して発生したイベントが時系列で表示されます。
    *   `Scheduled`: Podがどのワーカーノードに割り当てられたか
    *   `Pulling`: コンテナイメージのダウンロードを開始したか
    *   `Pulled`: イメージのダウンロードが成功したか
    *   `Created`: コンテナが作成されたか
    *   `Started`: コンテナが起動したか

👉 **使いどころ**:
PodのSTATUSが`Pending`のまま進まない場合、`Events`セクションを見れば、「イメージのダウンロードに失敗している(`ImagePullBackOff`)」や「配置できるノードが見つからない(`FailedScheduling`)」といった具体的な原因が記録されています。`kubectl get`で異常を見つけたら、まず`kubectl describe`で詳細を確認する、というのがトラブルシューティングの定石です。

---

### 2.3. コンテナのログを確認する `kubectl logs`

アプリケーションが正常に動作しているように見えても、内部でエラーが発生していることがあります。`kubectl logs`コマンドを使うと、コンテナ内で実行されているアプリケーションが出力する**標準出力・標準エラー出力（ログ）** を直接確認することができます。

#### ■ 基本的な使い方（Podを指定）

最も基本的な使い方は、ログを見たいPodの名前を直接指定する方法です。

```bash
# 特定のPodのログを表示
kubectl logs <Podの名前>
```

もしPod内に複数のコンテナがある場合は、`-c`オプションでコンテナ名を指定する必要があります。

```bash
# Pod内の特定のコンテナのログを表示
kubectl logs <Podの名前> -c <コンテナ名>
```

#### ■ 便利なオプション

##### **-f, --follow: ログをリアルタイムで表示する**

`-f`オプションを付けると、`tail -f`コマンドのように、新しく出力されるログをリアルタイムで表示し続けます。アプリケーションの現在の動作を確認したい場合に非常に便利です。

```bash
# ログをストリーミング表示
kubectl logs -f <Podの名前>
```
（終了するには `Ctrl + C` を押します）

##### **--previous: 前回のコンテナのログを表示する**

Podが何らかの理由でクラッシュし、再起動した場合（`RESTARTS`が1以上の場合）、`kubectl logs`は**現在動作しているコンテナ**のログを表示します。クラッシュする直前のログ、つまり**前回起動していたコンテナ**のログを見るには、`--previous`オプションを使います。

```bash
# クラッシュする前のコンテナのログを表示
kubectl logs --previous <Podの名前>
```

👉 **使いどころ**:
「Podが頻繁に再起動しているが、理由がわからない」という場合に、このコマンドでクラッシュ直前のエラーメッセージを確認することで、問題解決の糸口が見つかります。

---

#### ■ ラベルを使って複数のPodのログをまとめて見る

Deploymentによって管理されているPodなど、同じ役割を持つ複数のPodのログを一度に確認したい場合があります。その場合は、`-l` (`--selector`) オプションを使って**ラベル**を指定します。

`01-kubernetes-basics.md`で作成した`nginx-app.yaml`では、Deploymentが作るPodに `app: my-nginx` というラベルを付けていました。このラベルを利用します。

```bash
# "app=my-nginx" というラベルが付いたすべてのPodのログを表示
kubectl logs -l app=my-nginx
```

**実行結果**

```
my-nginx-deployment-66b6c48dd5-abcde: 192.168.0.1 - - [01/Jan/2024:12:10:00 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.81.0"
my-nginx-deployment-66b6c48dd5-fghij: 192.168.0.1 - - [01/Jan/2024:12:10:01 +0900] "GET / HTTP/1.1" 200 612 "-" "curl/7.81.0"
...
```
このように、どのPodが出力したログなのかが先頭に付与された形で表示されます。

👉 **使いどころ**:
負荷分散しているWebサーバーのアクセスログをまとめて監視したい場合など、複数のPodの動向を一度に把握したいときに非常に強力です。`-f`オプションと組み合わせることもできます (`kubectl logs -f -l app=my-nginx`)。
