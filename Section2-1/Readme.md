# Section.2-1 Pod

## Table of Contents

* [Objective](#Objective)
* [Guide](#Guide)
* [追加情報](#追加情報)

## Objective
* Pod リソースを理解する。

## Guide

### Step.1 Pod をデプロイする

この Section は Section.1 で作成した Master Node へログインした状態で進めます。Master Node で作業する理由は、Kubernetes クラスタのクレデンシャル情報を持っているためです。  
それでは、さっそく nginx Pod をデプロイしてみます。以下のコマンドを実行してください。
```bash
$ kubectl run test --image=nginx
pod/test created
```

Pod がデプロイされていることを確認してみましょう。以下のコマンドを実行してください。STATUS が Running になっていると思います。Kubernetes に Pod を起動する処理時間は1分もかかりません。数秒で起動することができます。
```bash
$ kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          6s
```

### Step.2 Pod の IP アドレスの確認
先ほど作成した Pod は、他の Pod と通信できるように IP アドレスを設定する必要があります。しかし、Kubernetes は IP アドレスの設定を自動的に行うため、手動による設定は不要です。IP アドレスを確認してみましょう。

```bash
$ kubectl get pod -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
test   1/1     Running   0          7m49s   192.168.5.3   worker01   <none>           <none>
```

IP アドレスが設定されており、Worker Node で動作していることが分かります。このように、Kubernetes では、Pod を起動するという指示を与えるだけで、IP アドレスを自動的に設定し、他の Pod と通信できる状態を構築してくれることが理解できました。この Pod は、Master Node と通信できるでしょうか。それでは確認してみましょう。

```bash
$ export PODIP=$(kubectl get pod test -o jsonpath='{.status.podIP}')
$ curl http://${PODIP}
```

Pod と通信できることが理解できたと思います。

### Step.3 Pod の削除
先ほど作成した Pod を削除してみましょう。

```bash
$ kubectl delete pod test
pod "test" deleted
```

Pod が削除されたことを確認します。

```bash
$ kubectl get pod
No resources found in default namespace.
```

### Step.4 Manifest ファイルから Pod を生成する
次に Pod を Manifest ファイルから生成してみます。まず、以下のコマンドを実行し、Manifest ファイルを作成しましょう。

```bash
$ kubectl run test --image=nginx --dry-run=client -o yaml > test.yaml
$ cat test.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
spec:
  containers:
  - image: nginx
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

この Manifest ファイルは、Pod を生成するための設計書です。本番環境で Kubernetes を運用する場合には、Manifest ファイルを作成し、Manifest ファイルを Kubernetes に適用することで運用するケースがメインとなりますので、やってみましょう。

```bash
$ kubectl apply -f test.yaml
pod/test created
```

Pod を確認してみます。

```bash
$ kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          12s
```

Pod が起動していることを確認できたと思います。


### Step.5 Pod の詳細情報の確認
`kubectl get` コマンドの出力結果だけでは、Pod の状態を詳細に確認することができません。しかし、運用中に Pod や他リソースの詳細を確認したいケースは度々発生します。その場合には、`kubectl describe` コマンドを使って、詳細を確認することができます。それでは、この Pod の状態を確認してみましょう。

```bash
$ kubectl describe pod test
```

image, IP アドレス, 動作している Node, Pod の状態が詳細に確認できることが分かります。運用中に想定外の動作が発生した場合には、`kubectl describe` コマンドにより、リソースの詳細を確認し原因を解析することができます。

### Step.6 Pod のログの確認
Pod の状態だけではなく、Pod の Container のログを確認したいケースもあります。その場合には、`kubectl logs` コマンドを使って確認することができます。以下のコマンドを実行し、まずはログを出力させます。
```bash
$ export PODIP=$(kubectl get pod test -o jsonpath='{.status.podIP}')
$ curl http://${PODIP}
$ curl http://${PODIP}
$ curl http://${PODIP}
```

3回アクセスしたため、Pod には3つのアクセスログが出力されたはずです。ログを確認してみましょう。
```bash
$ kubectl logs test
```

ログを確認できたでしょうか。VM であれば、ログはファイルに出力されますが、コンテナではファイルに出力することはせずに、エフェメラルな領域にログを出力します。そのため、Pod を停止するとログは揮発してしまいます。もし、ログの保存要件がある場合には、Persistent Volume を利用するか、ログをロギングサーバに転送する等の考慮が必要となります。

これで、この Section は終了です。不要になった Pod を削除しましょう。

```bash
$ kubectl delete pod test
$ kubectl get pod
```

## まとめ
`kubectl` コマンドを利用した Pod の一連の操作を学習しました。Pod は簡単に生成することができ、削除も同様です。Pod の詳細やログも確認することができるため、障害解析時に役立ててください。

## 追加情報
Pod の Manifest で image を指定しましたが、他の image を利用したい場合には、Public な Registry や自前の Registry に格納されている image が利用できます。他の image を利用することに興味があれば、[Dokcer Hub](https://hub.docker.com/) にアクセスし、他の image を指定して起動してみてください。

---
お疲れさまでした。次の [Section.2-2](../Section2-2/Readme.md) へ進みましょう。
