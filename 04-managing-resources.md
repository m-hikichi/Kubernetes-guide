# 4. リソースを操作・管理する (Managing Resources)

これまでの章で、リソースの状態を確認し、内部を調査する方法を学びました。この最後の章では、リソースの設定を**変更**したり、不要になったリソースを**削除**したりするための、より直接的な操作コマンドを学びます。

これらのコマンドはクラスタの状態を直接変更するため、実行する際はその影響をよく理解している必要があります。特に本番環境での操作は慎重に行いましょう。

---

### 4.1. 稼働中のリソースを編集する `kubectl edit`

`kubectl edit`は、稼働中のリソースのマニフェストを**その場で直接編集**し、変更を適用するためのコマンドです。

`kubectl apply -f <file.yaml>`では「①YAMLファイルを修正 → ②applyコマンド実行」という2ステップが必要ですが、`kubectl edit`を使えば、このプロセスを1ステップで、より素早く行うことができます。

#### ■ 基本的な使い方

`edit`の後ろにリソースの種類と名前を指定します。Deploymentだけでなく、PodやServiceなど、ほとんどのリソースが編集可能です。

```bash
# "my-nginx-deployment" というDeploymentを編集する
kubectl edit deployment my-nginx-deployment

# "my-first-pod" というPodを編集する
kubectl edit pod my-first-pod
```

このコマンドを実行すると、環境変数 `EDITOR` で設定されたテキストエディタ（デフォルトでは`vi`や`vim`）が起動し、現在のリソースのマニフェストが表示されます。

**Deploymentの編集例**
```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
  ...
spec:
  progressDeadlineSeconds: 600
  replicas: 2  # <-- 例えば、この値を 3 に変更する
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: my-nginx
  ...
```

ここで、例えば `replicas` の値を `2` から `3` に変更して保存・終了すると、`kubectl`がその変更を即座にクラスタに適用します。

```
deployment.apps/my-nginx-deployment edited
```

その後 `kubectl get pods` を実行すれば、Podが3つに増えていることが確認できます。

**注意点：Podの編集について**
`kubectl edit pod <Pod名>`でPodを直接編集することも可能ですが、**変更できるフィールドは限られています**。例えば、Podが使用するコンテナイメージ(`spec.containers[0].image`)など、根本的な定義は後から変更できません。変更しようとして保存しても、APIサーバからエラーが返されます。一般的に、Podの定義を変更したい場合は、そのPodを管理しているDeploymentなどを編集するのが正しいアプローチです。

#### ■ `kubectl edit` vs `kubectl apply`

| | `kubectl edit` | `kubectl apply -f <file.yaml>` |
| :--- | :--- | :--- |
| **手軽さ** | 非常に手軽。コマンド一発で編集・適用できる。 | ファイルの編集とコマンド実行の2段階が必要。 |
| **変更履歴** | **残らない**。誰がいつ何を変更したか追跡が困難。 | **残る**。GitでYAMLファイルを管理すれば変更履歴は明確。 |
| **再現性** | 低い。同じ変更を別のクラスタに適用するのが難しい。 | 高い。同じYAMLファイルを適用すればよい。 |
| **主な用途** | ・一時的なデバッグ（Pod数を増やすなど）<br>・緊急の修正<br>・設定値の意味を学ぶための試行錯誤 | ・本番・開発環境での定常的な運用<br>・CI/CDパイプラインでの自動デプロイ |

`kubectl edit`は非常に便利ですが、**変更が手元やGitのYAMLファイルに反映されない**という大きな欠点があります。これにより、「Gitで管理しているマニフェストと、実際に動いている設定が違う」という状態（**設定ドリフト**）が発生しやすくなります。

👉 **使いどころ**:
原則として、本番環境など、構成管理が重要な環境での恒久的な変更には**使うべきではありません**。
*   開発環境で「このパラメータを変えたらどうなるだろう？」と気軽に試したい場合。
*   本番障害時など、一刻も早く設定を変更する必要がある場合の緊急避難的な手段として。

普段の運用では`git`でYAMLファイルを管理し、`kubectl apply`で変更を適用するフローを徹底し、`kubectl edit`はあくまで一時的なツールとして使い分けるのが賢明です。

---

### 4.2. リソースを削除する `kubectl delete`

`kubectl delete`は、不要になったリソースをクラスタから削除するためのコマンドです。一度削除したリソースは（基本的には）元に戻せないので、特に本番環境では対象をよく確認してから実行する必要があります。

#### ■ 基本的な使い方

削除したいリソースの種類と名前を指定するのが最も基本的な使い方です。

```bash
# "my-first-pod" という名前のPodを削除する
kubectl delete pod my-first-pod

# "my-nginx-service" という名前のServiceを削除する
kubectl delete service my-nginx-service
```

Deploymentを削除すると、そのDeploymentによって管理されていたPodも**すべて一緒に削除される**点に注意してください。

```bash
# Deploymentを削除（管理下のPodもすべて削除される）
kubectl delete deployment my-nginx-deployment
```

#### ■ マニフェストファイルを使って削除する

`kubectl apply -f`でリソースを作成した場合、同じファイルを使って削除することもできます。`-f`オプションは`delete`コマンドでも有効です。

```bash
# nginx-app.yaml に定義されているすべてのリソースを削除する
kubectl delete -f nginx-app.yaml
```

この方法は、複数のリソース（DeploymentとServiceなど）を一度にまとめてクリーンアップしたい場合に非常に便利です。

#### ■ ラベルを使ってまとめて削除する

`-l`オプションでラベルを指定すれば、特定のラベルが付いたリソースをまとめて削除することもできます。

```bash
# "app=my-nginx" というラベルが付いたすべてのPodを削除
kubectl delete pod -l app=my-nginx
```

**警告**: このコマンドは非常に強力で、意図しないリソースまで削除してしまう危険性があります。実行する前に、`kubectl get pods -l app=my-nginx` のように`get`コマンドで対象となるリソースを正確に確認する癖をつけましょう。

#### ■ `--force` オプションについて

リソースが何らかの理由で正常に終了できず、`Terminating`状態のまま止まってしまうことがあります。そのような場合に、`--force`オプションを付けて強制的にリソースを削除することもできますが、これは**最後の手段**です。

```bash
# Podを強制的に削除（非推奨）
kubectl delete pod <Pod名> --force
```

強制削除は、クラスタ内で管理情報に不整合を起こす可能性があり、予期せぬ問題を引き起こすことがあります。まずは`kubectl describe`などでなぜ終了できないのか原因を調査し、どうしても解決しない場合の最終手段としてのみ検討してください。
