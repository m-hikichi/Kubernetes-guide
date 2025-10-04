### 第4章: Service詳解 🌐

`01-kubernetes-basics.md`では、Serviceが「複数のPodへのアクセスをまとめる窓口」であると学びました。この章では、Serviceの仕組みをさらに深く掘り下げ、その種類や使い方を実践的に学んでいきます。

-----

### 1\. Serviceとは？（復習）

Deploymentによって管理されているPodは、障害発生時やスケールアウト時に作り直される「揮発的」な存在です。Podが作り直されるたびにIPアドレスが変わってしまうため、クライアントがPodに直接アクセスするのは現実的ではありません。

そこで登場するのが**Service**です。Serviceは、複数のPodの前に立つ**永続的な単一の窓口**として機能します。

*   **不変のIPアドレス**: Serviceは自身に割り当てられたIPアドレス（ClusterIP）を持ち、これが変わることはありません。
*   **Podの自動追跡**: Serviceは**ラベルセレクタ**という仕組みを使って、どのPodにトラフィックを転送すべきかを常に把握しています。Podが増減しても、Serviceが自動で追跡してくれます。

これにより、クライアントはPodの個々の状態を気にすることなく、常にServiceのIPアドレスやDNS名にアクセスすればよくなります。

-----

### 2\. Serviceマニフェストの書き方

Serviceを定義するマニフェストの主要な項目を見ていきましょう。

`service-example.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app  # (1) どのPodにトラフィックを送るかの目印
  ports:
    - protocol: TCP
      port: 80         # (2) Serviceが公開するポート
      targetPort: 8080 # (3) Pod側のコンテナが待ち受けるポート
  type: ClusterIP      # (4) Serviceの種類
```

#### (1) `spec.selector`
Serviceがトラフィックを転送する対象のPodを選択するための**ラベル**を指定します。この例では、`app: my-app`というラベルを持つすべてのPodが、このServiceの転送対象になります。DeploymentのPodテンプレート（`.spec.template.metadata.labels`）と一致させる必要があります。

#### (2) `spec.ports.port`
Service自体がクラスタ内で公開するポート番号です。クラスタ内の他のPodは、このポート番号を使ってServiceにアクセスします。

#### (3) `spec.ports.targetPort`
転送先となるPod内のコンテナが、実際にリクエストを待ち受けているポート番号です。`port`と`targetPort`は同じ番号でも構いません。

#### (4) `spec.type`
Serviceの公開範囲や振る舞いを決定する種類です。これについては後ほど詳しく解説します。

-----

### 3\. 実践：Serviceを作成してPodに接続する

実際にDeploymentとServiceを作成し、`port-forward`を使って動作確認を行いましょう。

#### Step 1: サンプルアプリケーション（Deployment）の作成

まず、`app: my-app`というラベルを持つPodを3つ起動するDeploymentを作成します。

`app-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app # ★Serviceのselectorと一致させる
    spec:
      containers:
      - name: my-app-container
        image: "httpd:2.4" # Apache Web Server
        ports:
        - containerPort: 80 # コンテナは80番ポートで待機
```

このマニフェストを適用します。
```bash
kubectl apply -f app-deployment.yaml
kubectl get pods
# NAME                                 READY   STATUS    RESTARTS   AGE
# my-app-deployment-5f5f47d7f6-abcde   1/1     Running   0          10s
# my-app-deployment-5f5f47d7f6-fghij   1/1     Running   0          10s
# my-app-deployment-5f5f47d7f6-klmno   1/1     Running   0          10s
```

#### Step 2: Serviceリソースの作成

次に、このDeploymentが管理するPod群に対して、Serviceを作成します。

`app-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
```

マニフェストを適用し、Serviceが作成されたことを確認します。
```bash
kubectl apply -f app-service.yaml
kubectl get service my-app-service
# NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# my-app-service   ClusterIP   10.100.200.30   <none>        80/TCP    5s
```
`CLUSTER-IP`に、クラスタ内でのみ有効なIPアドレスが割り当てられていることがわかります。

#### Step 3: `port-forward`を用いた動作確認

`kubectl port-forward`コマンドを使うと、ローカルPCのポートとクラスタ内のリソース（PodやService）のポートを一時的につなぐことができます。これにより、クラスタ外部から安全に動作確認ができます。

以下のコマンドを実行して、ローカルPCの`8080`番ポートを`my-app-service`の`80`番ポートに転送します。
```bash
# service/<サービス名> <ローカルポート>:<サービスポート>
kubectl port-forward service/my-app-service 8080:80
# Forwarding from 127.0.0.1:8080 -> 80
# Forwarding from [::1]:8080 -> 80
```

この状態で、ブラウザや`curl`コマンドで `http://localhost:8080` にアクセスします。
```bash
curl http://localhost:8080
# <html><body><h1>It works!</h1></body></html>
```
Apacheのデフォルトページが表示されれば成功です。リクエストはローカルPCの8080番ポートからServiceに転送され、Serviceが3つのPodのいずれかに自動で振り分けています。

-----

### 4\. ServiceのTypeを理解する

Serviceの`type`フィールドは、そのServiceをどのように公開するかを定義します。

#### ■ `ClusterIP` (デフォルト)
*   **用途**: クラスタ内部のアプリケーション間通信
*   **特徴**:
    *   クラスタ内部でのみ有効なIPアドレス（ClusterIP）が割り当てられます。
    *   クラスタの外部からは直接アクセスできません。
    *   `type`を省略した場合のデフォルト値です。マイクロサービスアーキテクチャで、バックエンドサービス間の通信によく使われます。

#### ■ `NodePort`
*   **用途**: 開発や検証目的で、一時的にクラスタ外部へサービスを公開
*   **特徴**:
    *   `ClusterIP`の機能に加えて、**すべてのワーカーノード**の特定のポート（デフォルトでは30000-32767番）を外部に公開します。
    *   `<いずれかのNodeのIP>:<NodePort>` というアドレスで、クラスタ外部からアクセスできるようになります。
    *   本番環境での利用は、セキュリティやポート管理の観点からあまり推奨されません。

`app-service.yaml`の`type`を`NodePort`に変更してみましょう。
```yaml
spec:
  type: NodePort # ここを変更
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      # nodePort: 30007 # 省略すると自動で空きポートが割り当てられる
```
```bash
kubectl apply -f app-service.yaml
kubectl get service my-app-service
# NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# my-app-service   NodePort    10.100.200.30   <none>        80:31234/TCP   1m
```
`PORT(S)`の表示が `80:31234/TCP` のようになりました。これは、Serviceの80番ポートが、全ノードの31234番ポートにマッピングされていることを意味します。

-----

### 5\. Serviceを利用したDNS

Kubernetesクラスタには、**CoreDNS**というDNSサーバーが標準で組み込まれています。これにより、クラスタ内のPodは、IPアドレスの代わりに**サービス名で他のサービスにアクセス**できます。

*   Serviceを作成すると、`<サービス名>.<名前空間>.svc.cluster.local` という形式のDNS名が自動で登録されます。
*   同じ名前空間内のPodからは、名前空間部分を省略して単に `<サービス名>` だけでアクセスできます。

例えば、`default`名前空間にあるPodから`my-app-service`にアクセスしたい場合、
`http://10.100.200.30` の代わりに `http://my-app-service` というアドレスを使えます。

これにより、ServiceのClusterIPを直接知る必要がなくなり、より柔軟で管理しやすいアプリケーション構成が実現できます。

---

### 6\. お片付け

学習が終わったら、作成したリソースを削除します。
```bash
kubectl delete -f app-service.yaml
kubectl delete -f app-deployment.yaml
