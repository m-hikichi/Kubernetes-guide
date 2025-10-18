# 05 JobとCronJobによるタスク実行

DeploymentやReplicaSetは、コンテナを継続的に実行し続けるためのリソースでした。しかし、アプリケーションの要件によっては、一度だけ実行して終了するタスクや、定期的に実行したいタスクもあります。Kubernetesでは、`Job`と`CronJob`というリソースがこれらのユースケースに対応します。

## 1. Jobとは

`Job`は、1つ以上のPodを正常に完了させることを目的としたリソースです。Pod内のプロセスが正常終了（終了コード0）すると、そのPodは「完了（Completed）」状態と見なされます。Jobは、指定された数のPodが完了するまでPodの作成を試み続けます。

**主な用途:**
*   データ移行や初期化などの一度きりのバッチ処理
*   重い計算処理
*   バックアップタスクの実行

### Deployment/ReplicaSetとの違い
DeploymentやReplicaSetは、Podが何らかの理由で終了した場合、新しいPodを起動して指定されたレプリカ数を維持しようとします（自己修復）。一方、JobはPodが**完了**することを目指しており、完了したPodを再起動することはありません。

## 2. Jobのマニフェスト

Jobのマニフェストは、Podのテンプレート（`spec.template`）に加えて、Job固有の振る舞いを制御するフィールドを持ちます。

**例：円周率を2000桁計算して出力するJob**
`pi-job.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never # Jobでは Never または OnFailure が一般的
  backoffLimit: 4 # 失敗時のリトライ回数
```
*   `spec.template`: Deploymentなどと同様に、作成するPodの定義を記述します。
*   `spec.template.spec.restartPolicy`: Pod内のコンテナが失敗した際の再起動ポリシーです。Jobでは、Pod自体の再試行はJobコントローラーが行うため、`Never`または`OnFailure`を指定します。`Always`（デフォルト）は指定できません。
*   `spec.backoffLimit`: Jobが失敗と判断されるまでのリトライ回数を指定します。

### 並列実行

Jobでは、複数のPodを並列で実行させることも可能です。
*   `spec.completions`: 成功させるPodの総数を指定します。この例では1つのPodが完了すればJobは成功です。
*   `spec.parallelism`: 同時に実行するPodの数を指定します。

## 3. CronJobとは

`CronJob`は、Linuxのcronのように、スケジュールに基づいてJobを定期的に作成・実行するためのリソースです。

**主な用途:**
*   定期的なバックアップ
*   日次、週次、月次のレポート生成
*   定期的なデータ同期

CronJobは、指定されたスケジュールになると、自身の`jobTemplate`に基づいて新しいJobリソースを作成します。

## 4. CronJobのマニフェスト

CronJobのマニフェストでは、実行スケジュールと、作成するJobのテンプレートを定義します。

**例：1分ごとに"Hello from the Kubernetes cluster"と表示するCronJob**
`hello-cronjob.yaml`
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *" # 毎分実行
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from the Kubernetes cluster"
          restartPolicy: OnFailure
```
*   `spec.schedule`: Jobを実行するスケジュールをcron形式で指定します。`分 時 日 月 曜日`の順です。
*   `spec.jobTemplate`: スケジュールに基づいて作成されるJobのテンプレートです。Jobのマニフェストの`spec`フィールドと同じ内容を記述します。

### 同時実行ポリシー (`concurrencyPolicy`)

前のJobがまだ実行中に次のスケジュール時刻が来た場合の振る舞いを制御します。
*   `Allow` (デフォルト): 新しいJobを常に作成します。Jobが並行して実行される可能性があります。
*   `Forbid`: 前のJobが完了していない場合、新しいJobの作成をスキップします。
*   `Replace`: 実行中のJobをキャンセルし、新しいJobで置き換えます。
