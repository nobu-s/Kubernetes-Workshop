# Section.2-4 Service

## Table of Contents

* [Objective](#Objective)
* [Guide](#Guide)
* [追加情報](#追加情報)

## Objective
* Service の仕組みを理解する。

## Guide

### Step.1 Service の概要

Service リソースは、Kubernetes クラスタ上の Pod に対するエンドポイントを提供し、L4 ロードバランシング機能を内包しています。

### Step.2 Service Manifest の作成

Service の Manifest ファイルを作成します。
```bash
$ vi servicetest.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: servicetest
  name: servicetest
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    run: deploymenttest
  type: NodePort
status:
  loadBalancer: {}
```

### Step.3 Service の生成
Service を生成します。以下のコマンドを実行してください。
```bash
$ kubectl apply -f servicetest.yaml
```

Service が生成されたことを確認します。以下のコマンドを実行し確認してみましょう。  
```bash
$ kubectl get service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP        6d19h
servicetest   NodePort    10.104.105.16   <none>        80:30080/TCP   4s
```

Service を生成したことにより、Pod Network に存在する Pod は、Node Network や外部ネットワークと接続が可能になります。

> **Note**  
> **CLUSTER-IP** とは、～～～　ここは後で図を付けて説明する予定。

### Step.4 Service の情報確認
Service リソースである servicetest を生成しましたが、この Service は、どの Pod に振り分けるのか確認してみましょう。
```bash
$ kubectl describe service servicetest
Name:                     servicetest
Namespace:                default
Labels:                   app=servicetest
Annotations:              Selector:  run=deploymenttest
Type:                     NodePort
IP:                       10.104.105.16
Port:                     80-80  80/TCP
TargetPort:               80/TCP
NodePort:                 80-80  30080/TCP
Endpoints:                192.168.5.24:80,192.168.5.25:80,192.168.5.26:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Endpoints の行を見てください。IP アドレスが3つ記載されています。この Endpoints に記載されている IP アドレスへ振り分けることを示しています。  
Service リソースは、Selector により指定した Label が付与されている Pod を探し出し、該当 Label が付与されている Pod に対し振り分け制御を行います。  
Pod の Label と IP アドレスを確認してみましょう。

```bash
$ kubectl get pod -o wide -L run
NAME                              READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES   RUN
deploymenttest-59b8b56748-7v8zs   1/1     Running   1          23h   192.168.5.26   worker01   <none>           <none>            deploymenttest
deploymenttest-59b8b56748-gdf4f   1/1     Running   1          23h   192.168.5.24   worker02   <none>           <none>            deploymenttest
deploymenttest-59b8b56748-jgx4m   1/1     Running   1          23h   192.168.5.25   worker01   <none>           <none>            deploymenttest
```

Deployment を更新し、Pod の replica 数を変更することにより、動的に Service の振り分け先も変更されるため、運用者による手動の変更作業は必要ありません。

### Step.5 Service にアクセスする
PC のブラウザを開き、アドレスバーに **http://< Elastic IP アドレス >>:30080** を入力し、アクセスしてください。nginx の Welcome ページが表示されることが確認できます。  
アクセスするたびに振り分け先が変わるため、`kubectl logs` コマンドを利用し、アクセスが振り分けられていることを確認してみてください。

## まとめ
Service リソースを利用し、Kubernetes クラスタから外部ネットワークへサービスを公開する方法を理解しました。Service リソースは、Selector に指定された Label を探しだし、Endpoints に自動的に加えてくれます。これは、運用者の手動操作を必要としないことが理解できたと思いますので、Kubernetes は人手を介することなくスケールしやすいことが分かりました。

---
お疲れ様でした。8/31 のもくもく会は以上で終了となります。  
Workshop の環境は、もくもく会終了後に削除しますので、そのままログアウトして頂いて問題ありません。
