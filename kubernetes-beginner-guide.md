### はじめてのKubernetes入門 🔰

この資料は、Kubernetesを初めて学ぶ方が、基本的な仕組みを理解し、実際に手を動かしてアプリケーションを動かせるようになることを目指します。

### 1\. Kubernetesとは？ 🤔

Kubernetes（クバネティス、略して**K8s**）は、たくさんの「コンテナ」を自動で管理・運用してくれる**オーケストラの指揮者**のようなソフトウェアです。

Dockerなどで作ったアプリケーションの実行環境を「コンテナ」と呼びます。サービスが大きくなると、このコンテナの数がどんどん増えていきます。

  * 「コンテナが急に止まってしまった！」
  * 「アクセスが増えたから、コンテナの数を3個から10個に増やしたい！」

こうした作業を手動でやるのはとても大変です。Kubernetesを使うと、このようなコンテナの管理をすべて自動化できます。

👉 **一言でいうと**：コンテナ化されたアプリケーションの**デプロイ、スケーリング、管理を自動化する**ためのオープンソースプラットフォームです。

-----

### 2\. Kubernetesの仕組み（アーキテクチャ）

Kubernetesは、全体として「**クラスタ**」という大きなまとまりで動きます。このクラスタは、役割の違う2種類のコンピュータ（**ノード**）で構成されています。ここでは、まずその土台となる物理的な仕組みを見ていきましょう。

#### ■ コントロールプレーン (Control Plane)

クラスタ全体の司令塔です 🧠。
クラスタの状態を常に監視し、「アプリケーションをいくつ起動するか」「どこで動かすか」などを決定して指示を出します。私たちが`kubectl`コマンドで命令を送る先は、このコントロールプレーンです。

#### ■ ワーカーノード (Worker Node)

実際にアプリケーション（コンテナ）を動かす作業員です 💪。
コントロールプレーンからの指示に従って、コンテナを起動したり停止したりします。アプリケーションのプログラムが実際に動いているのは、このワーカーノードの上です。

👉 **関係性のイメージ**
司令塔であるコントロールプレーンが、複数の作業員（ワーカーノード）を管理・監督しているイメージです。

```mermaid
graph TD
    subgraph クラスタ
        subgraph コントロールプレーン（司令塔）
            A[APIサーバ]
        end
        subgraph "複数のワーカーノード（作業員）"
            WN1[ワーカーノード1]
            WN2[ワーカーノード2]
            WN3[...]
        end
    end

    User[あなた] -- kubectlコマンドで指示 --> A
    A -- 命令 --> WN1
    A -- 命令 --> WN2
```

-----

### 3\. Kubernetesを構成する主要な「部品」たち 🧩

ワーカーノードの上では、アプリケーションを動かすために様々な\*\*部品（リソース）\*\*が使われます。ここでは、最も基本的で重要な3つの部品を紹介します。

#### ■ ポッド (Pod)

Kubernetesが管理するアプリケーションの**最小単位**です。
1つ以上のコンテナをまとめたカプセルのようなもので、Pod単位でネットワーク設定などが共有されます。Kubernetesでは、コンテナを直接操作するのではなく、この「Pod」という単位で扱います。
👉 **例えるなら**：アプリケーションが住む「**家**」のようなものです。

#### ■ デプロイメント (Deployment)

Podの数やバージョンを管理する**監督役**です。
「Podを常に3つ起動した状態に保ってください」「アプリを新しいバージョンに更新してください」といった指示をDeploymentに出しておけば、あとは自動でPodの状態を維持・管理してくれます。Podが壊れたら新しいPodを自動で作り直す**自己修復機能**も担います。
👉 **例えるなら**：家の数を管理し、壊れたら直してくれる「**大工さんの親方**」です。

#### ■ サービス (Service)

複数のPodにまとめてアクセスするための**窓口**です。
Podはいつ壊れて作り直されるか分からないため、IPアドレスなどが変わりやすく不安定です。Serviceは、それらのPodの前に立って、常に変わらない一つの窓口（IPアドレスやDNS名）を提供します。これにより、外部から安定してアプリケーションにアクセスできるようになります。
👉 **例えるなら**：複数の家（Pod）への荷物をまとめて受け取ってくれる「**マンションの受付**」です。

-----

### 4\. 実践！Kubernetesを動かしてみよう

ここからは、実際にコマンドを打ちながらKubernetesを体験してみましょう。

#### Step 1: 学習用クラスタを構築する (kind)

**kind (Kubernetes in Docker)** を使って、手元のPCに学習用クラスタを構築します。

**前提**: Dockerと`kubectl`がインストールされている必要があります。

```bash
# kindのインストール (macOS / Linux)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# クラスタの作成
kind create cluster --name my-cluster
```

#### Step 2: クラスタの状態を確認する

`kubectl`コマンドで、クラスタ（コントロールプレーンとワーカーノード）が正しく動いているか確認します。

```bash
# 司令塔の状態を確認
kubectl cluster-info

# 作業員の状態を確認
kubectl get nodes
```

#### Step 3: 【はじめの一歩】Podを動かしてみる

まずは、Kubernetesにおける最小単位である`Pod`を1つだけ、シンプルに動かしてみましょう。

##### **1. Podマニフェストの作成**

最も基本的なWebサーバー（nginx）のPodの設計図です。

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

##### **2. マニフェストの適用と確認**

```bash
# Podを作成
kubectl apply -f pod.yaml

# Podが動いているか確認 (STATUSがRunningになればOK)
kubectl get pods
```

これで、Kubernetesクラスタ上で1つのPodが動作している状態になりました。

**しかし、このままでは2つの大きな問題があります。**

1.  **外部からアクセスできない**: クラスタの外（あなたのPCのブラウザなど）から見る方法がありません。
2.  **壊れたらおしまい**: もしこのPodが何らかの原因で停止したら、誰かが手動で作り直さない限り復旧しません。

この問題を解決してくれるのが、次に出てくる`Deployment`と`Service`です。

#### Step 4: 【実践編】DeploymentとServiceでアプリを本格運用する

次に、先ほどのPodを`Deployment`に管理させ、`Service`を使って外部に公開します。

##### **1. マニフェストの作成**

`Deployment`と`Service`の設計図を1つのファイルにまとめます。

`nginx-app.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 2 # Podを2つに増やす
  selector:
    matchLabels:
      app: my-nginx
  template: # ★ここがPodの設計図。先ほどのPod定義とよく似ています
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-service
spec:
  type: NodePort
  selector:
    app: my-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

👉 `Deployment`の中の`.spec.template`以下が、管理対象となるPodの設計図になっています。ここにラベル(`labels`)を付けて、ServiceがどのPodに接続すればよいかの目印にしています。

##### **2. マニフェストの適用と確認**

```bash
# 先ほど作ったPodは削除します
kubectl delete -f pod.yaml

# 新しいマニフェストを適用
kubectl apply -f nginx-app.yaml

# Podが2つ起動していることを確認
kubectl get pods

# Serviceが作られていることを確認
kubectl get service
```

##### **3. ブラウザでアクセス**

`http://localhost:30007` にアクセスし、nginxのページが表示されれば成功です。

#### Step 5: 自己修復機能を試す

Podを1つ削除して、Deploymentが自動で新しいPodを起動する様子を観察します。

```bash
# Podの名前を確認
kubectl get pods

# Podを1つ削除 (名前は自分の環境のものに置き換えてください)
kubectl delete pod <Podの名前>

# すぐにPodの状態を監視！
kubectl get pods -w
```

1つが`Terminating`（終了中）になると同時に、新しいPodが自動で作成されるのがわかります。

-----

### 5\. お片付け（クラスタ削除）

学習が終わったら、以下のコマンドでクラスタをきれいに削除できます。

```bash
kind delete cluster --name my-cluster
```
