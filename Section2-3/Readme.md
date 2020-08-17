# Section.2-3 Deployment

## Table of Contents

* [Objective](#Objective)
* [Guide](#Guide)
* [追加情報](#追加情報)

## Objective
* Deployment の仕組みを理解する。

## Guide

### Step.1 Deployment の概要

Replicaset リソースを管理し、Pod のローリングアップデートやロールバックをするためのリソースです。

### Step.2 Replicaset Manifest の作成

Deployment の Manifest ファイルを作成します。
```bash
$ vi deploymenttest.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploymenttest
  labels:
    run: deploymenttest
spec:
  replicas: 3
  selector:
    matchLabels:
      run: deploymenttest
  template:
    metadata:
      labels:
        run: deploymenttest
    spec:
      containers:
      - name: nginx
        image: nginx:1.18
```

### Step.3 Deployment の生成
Deployment を生成します。以下のコマンドを実行してください。
```bash
$ kubectl apply -f deploymenttest.yaml
```

Deployment が生成されたことを確認します。以下のコマンドを実行し確認してみましょう。  
```bash
$ kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
deploymenttest   3/3     3            3           11s
```

**Note**
| Key | Description |
| --- | --- |
| NAME | オブジェクト名 |
| READY | 現在リクエストを引き受けることができる Pod 数 |
| UP-TO-DATE | Desired State を満たすためにアップデートが完了した Pod 数 |
| AVAILABLE | 現在利用可能な Pod 数 |
| AGE | オブジェクトが生成からの経過時間 |

Deployment は Replicaset を生成し、生成された Replicaset が Pod を生成するという親子関係があります。  
Deployment, Replicaset, Pod を確認してみましょう。Replicaset 名に、ランダム文字列が付与されていることがわかります。さらに、Pod 名には、Replicaset 名に更にランダム文字列が付与されています。Deployment により、Replicaset, Pod と親子関係であることが名前からも推測することができるようになります。

```bash
$ kubectl get all
```

Pod のアップデートをするため、以下のように Manifest ファイルを修正してください。
* spec.template.spec.containers.image を nginx:1.18 -> nginx:1.19 へ修正します。
```bash
$ vi deploymenttest.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploymenttest
  labels:
    run: deploymenttest
spec:
  replicas: 3
  selector:
    matchLabels:
      run: deploymenttest
  template:
    metadata:
      labels:
        run: deploymenttest
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
```

それでは、Manifest ファイルを適用してみます。
```bash
$ kubectl apply -f deploymenttest.yaml
```

```bash
$ kubectl get pod
```

Pod が一つ増えては、一つ減りを繰り返していることが分かります。全ての Pod のアップデートが完了するまで繰り返します。全ての Pod が停止することなく、縮退せずにローリングアップデートが実施されるため、サービスへの影響を小さくローリングアップデートできることが分かりました。

> **Note**
> アップデート中は、古いバージョンの Pod と新しいバージョンの Pod が一時的に混在する環境になります。そのため、アプリケーションをアップデートする場合には、異なるバージョンが混在しても問題ないようにしておく必要があります。異なるバージョンが混在することを許容できないアプリケーションの場合には、アップデート戦略を変更してください。


## まとめ
Deployment を利用することで、アップデート時に複雑な手順を踏むことなく、Kubernetes に任せることで自動的にアップデートできることを理解しました。Pod リソースのスケールを管理するために Replicaset があり、Replicaset を管理し、Pod のアップデートを管理するために、Deployment があることを理解できたと思います。本番運用では、Pod や Replicaset リソースを生成するのではなく、Deployment リソースを生成するようにしましょう。

---
お疲れ様でした。それでは最後の Section に進みましょう。[Section.2-4](../Section2-4/Readme.md)
