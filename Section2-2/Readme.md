# Section.2-2 Replicaset

## Table of Contents

* [Objective](#Objective)
* [Guide](#Guide)
* [追加情報](#追加情報)

## Objective
* Repicaset の仕組みを理解する。

## Guide

### Step.1 Replicaset の概要
Replicaset は、Pod のレプリカを生成し、レプリカの数を維持し続けるリソースです。

### Step.2 Replicaset Manifest の作成

Replicaset の Manifest ファイルを作成します。
```bash
$ vi replicatest.yaml
```
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicatest
  labels:
    run: replicatest
spec:
  replicas: 3
  selector:
    matchLabels:
      run: replicatest
  template:
    metadata:
      labels:
        run: replicatest
    spec:
      containers:
      - name: nginx
        image: nginx
```

spec.temlate から下の行が、Section.2-1 で作成した Pod の Manifest と似ていることに気が付くと思います。Replicaset の Manifest は、Pod の定義を含みつつ、Pod をいくつ複製するのかを定義しています。  
spec.replicas の数は、いくつの Pod を作成するのかを示します。
spec.selector.mathLabels は、Pod に付与されたラベルを指定しています。Pod に run: replicaset のラベルが付与されたもののみが、この Replicaset により管理されることになります。

### Step.3 Replicaset の生成
Replicaset を生成します。以下のコマンドを実行してください。
```bash
$ kubectl apply -f replicaset.yaml
```

Replicaset が生成されたことを確認します。以下のコマンドを実行し確認してみましょう。  
```bash
$ kubectl get replicaset
NAME          DESIRED   CURRENT   READY   AGE
replicatest   3         3         3       20s
```

**Note**
| Key | Description |
| --- | --- |
| NAME | オブジェクト名 |
| DESIRED | あるべき Pod 数 |
| CURRENT | 現在の Pod 数 (Pod STATUS が RUNNING 以外のものも含まれる) |
| READY | 現在リクエストを引き受けることができる Pod 数 (Pod STATUS が RUNNING になっているもの) |
| AGE | オブジェクトが生成からの経過時間 |

Replicaset により Pod がいくつ生成されたのか確認します。Section.2-1 で生成した Pod と違い、Pod 名の後ろにランダム文字列が付与されていることが確認できるはずです。  
Replicaset により生成された Pod は、Replicaset 名の後に「-xxxxx」のランダム文字列が付与されます。

```bash
$ kubectl get pod
```

Replicaset により生成された Pod を Terminate してみます。結果はどうなるでしょうか。

```bash
$ kubectl get pod
$ export PODNAME=$(kubectl get pod -o=jsonpath='{.items[0].metadata.name}')
$ kubectl delete pod $PODNAME
$ kubectl get pod
```

Pod を Terminate しても、すぐに新しい Pod が生成されたことが確認できます。次にすべての Pod を Terminate してみましょう。
```bash
$ kubectl get pod
$ kubectl delete pod --all
$ kubectl get pod
```

全ての Pod を Terminate しても、すぐに新しい Pod が生成され、あるべき Pod 数を維持しています。Kubernetes では、Replicaset を利用することで、常にあるべき Pod 数を維持することを自動的に行います。

全ての Pod を停止するには、Replicaset を削除する必要があります。

```bash
$ kubectl delete replicaset replicatest
$ kubectl get replicaset
```

Replicaset が削除されると、Pod も削除されます。確認してみましょう。

```bash
$ kubectl get pod
```

これで、本 Section は完了です。

## まとめ
Replicaset のふるまいを確認しました。Replicaset により、常に spec.replicas に定義した Pod 数が維持されることが理解できたと思います。  
可用性が必要なサービスであっても、Kubernetes は自動的に可用性を保つようにふるまうことで、運用者の操作無しで自動的に可用性が保たれます。

## 追加情報
Kubernetes の重要なアーキテクチャのひとつに、**Reconciliation Loop** があります。**Reconciliation loop** は以下4点の動作を常に繰り返すことで、クラスタの状態を保っています。
* Desired State を取得
* Current State を取得
* Desired State と Current State を比較
* 差分があるものについては、Desired State になるように修正する

---
お疲れさまでした。次の [Section.2-3](../Section2-3/Readme.md) へ進みましょう。
