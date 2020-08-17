# Section.1 Kubernetes のインストール

## Table of Contents

* [Objective](#Objective)
* [Workshop へのアクセス](#Workshop-へのアクセス)
* [Kubernetes のインストール](#Kubernetes-のインストール)
  * [Step.1 Kubernetes インストール要件の確認](#Step1-Kubernetes-インストール要件の確認)
  * [Step.2 ブリッジを通過するトラフィックの通信許可](#Step2-ブリッジを通過するトラフィックの通信許可)
  * [Step.3 Docker のインストール](#Step3-Docker-のインストール)
  * [Step.4 kubeadm,kubelet,kubectlのインストール](#Step4-kubeadm-kubelet-kubectlのインストール)
  * [Step.5 Master Node の構築](#Step5-Master-Node-の構築)
  * [Step.6 Worker Node をクラスタに追加する](#Step6-Worker-Node-をクラスタに追加する)
  * [Step.7 クラスタ状態を確認](#Step7-クラスタ状態を確認)
  * [Step.8 まとめ](#Step8-まとめ)
* [追加情報](#追加情報)

## Objective
* Kubernetes デプロイツールの kubeadm を利用し、Kubernetes をインストールします。  
* 作成するクラスタ構成は以下の表の通りです。

| Node 種別 | 台数 | 説明 |
| --- | --- | --- |
| Master Node | 1 | Kubernetes の機能に必要なコアコンポーネントを動かすための Node |
| Worker Node | 2 | アプリケーションを動作させるための Node |

> **Note**
> Workshop では、Master Node を1台しか作成しませんが、本番環境で運用する場合には可用性を考慮し、最低3台以上の奇数の構成にしてください。Master Node では、Kubernetes のコアコンポーネントが起動しているため、Master Node が消失すると Kubernetes が正常に機能しなくなります。最低3台とする理由は、コアコンポーネントの一つである etcd が Raft という合意アルゴリズムを採用しているためです。この Workshop では、Raft について詳細を説明しません。

## Workshop へのアクセス
* Workshop へのアクセス情報は、ログイン情報ページに記載があります。
* Teraterm もしくは、Rlogin 等の Terminal ソフトウェアを起動し、Workshop へログインします。
* Workshop には、3つのノードが用意されています。3枚の Terminal を開き、Master Node へログインします。
* Master Node から Worker Node へ、SSH でログインします。
* この時点で、Terminal は以下のノードへそれぞれログインした状態になっているはずです。
  - Master Node
  - Worker Node 1
  - Worker Node 2

## Kubernetes のインストール

### Step.1 Kubernetes インストール要件の確認
Kubernetes の要件に、全ての Node の MAC アドレスおよび product_uuid はユニークであること、Swap がオフであることが挙げられています。  
現在のところ kubelet は SELinux を正式にサポートしていません。そのため、SELinux を無効化しておく必要があります。  
そのため、Kubernetes をインストールする前に、Workshop の環境が動作要件を満たしているのか確認します。  

* 3つの Node で、以下のコマンドを実行し、MAC アドレスを検証します。全ての Node で MAC アドレスはユニークであることが確認できます。

```bash
$ ip a | grep link/ether | awk '{ print $2 }'
```

* 次に、product_uuid を3つのノードでユニークであることを検証します。以下のコマンドを実行し確認してみましょう。

```bash
$ sudo cat /sys/class/dmi/id/product_uuid
```

* 続いて、Swap がオフであることを確認します。Swap 行を確認し、0 であることを確認してください。
```bash
$ free
```

* 最後に、SELinux を無効化します。Permissive モードにするため、以下のコマンドを実行します。
```bash
$ sudo setenforce 0
```

* Permissive モードになっていることを確認します。
```bash
$ getenforce
```

* OS 再起動時に Enforcing モードに戻らないように、SELinux の設定ファイルを編集します。
```bash
$ sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
$ grep SELINUX= /etc/selinux/config
```

### Step.2 ブリッジを通過するトラフィックの通信許可
Linux の iptables がブリッジを通過するトラフィックを正確に処理できるように sysctl コマンドを用いて設定を変更します。

* まず最初に、Bridge Netfilter をロードします。
```bash
$ sudo modprobe br_netfilter
```

* Bridge Netfilter モジュールがロードされていることを確認してみましょう。
```bash
$ sudo lsmod | grep br_netfilter
```

* Kubernetes 用の sysctl 設定ファイルを作成します。
```bash
$ sudo vi /etc/sysctl.d/k8s.conf
$ cat /etc/sysctl.d/k8s.conf
```
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
```bash
$ sudo sysctl --system
```

* sysctl 設定ファイルが反映されたか確認してみましょう。
```bash
$ sudo sysctl -a | grep net.bridge.bridge-nf-call-ip
```

### Step.3 Docker のインストール
* コンテナランタイムの Docker をインストールします。
```bash
$ sudo yum install -y docker
```

* Docker がインストールされていることを確認します。
```bash
$ sudo yum info docker
```

* Docker を起動し、OS 再起動後も自動的に起動するように設定します。
```bash
$ sudo systemctl enable --now docker
```



### Step.4 kubeadm,kubelet,kubectlのインストール
* Kubernetes のパッケージは、OS デフォルトのパッケージリポジトリに含まれていないため、Kubernetes リポジトリを追加し、Kubernetes パッケージを利用できるようにします。
```bash
$ sudo vi /etc/yum.repos.d/kubernetes.repo
$ cat /etc/yum.repos.d/kubernetes.repo
```
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

* Kubernetes リポジトリの GPG キーを追加します。処理の途中で、**Is this ok [y/N]:** と表示されたら、**y** を入力し、GPG キーを登録します。
```bash
$ sudo yum repolist
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: d36uatko69830t.cloudfront.net
 * extras: d36uatko69830t.cloudfront.net
 * updates: d36uatko69830t.cloudfront.net
kubernetes/signature                                        |  454 B  00:00:00
Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
Importing GPG key 0xA7317B0F:
 Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@goo
gle.com>"
 Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Is this ok [y/N]: y
```

* kubelet , kubeadm , kubectl をインストールします。
```bash
$ sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```  

* kubelet を起動し、OS再起動後も自動起動するように設定します。
```bash
$ sudo systemctl enable --now kubelet
```

### Step.5 Master Node の構築
* まず、Kubernetes のコアコンポーネント機能を動作させるための、Master Node を構築します。本手順は、Master Node のみで実行しますので、Worker Node では実行しないでください。
* この Workshop では、Pod Network に Calico を使用するため、--pod-network-cidr=192.168.0.0/16 を指定します。なお、現在の Calico は仕様が変更されており、Pod Network は 192.168.0.0/16 以外も使用可能になっていますが、今回は慣例的に 192.168.0.0/16 を使用します。

```bash
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

5分程処理に時間がかかりますので、少し待ちましょう。`kubeadm init` の処理が完了すると以下のような表示が出力されますので、確認してみてください。  
`kubeadm join` で始まるコマンドは、Worker Node の構築に必要となりますので、メモ帳等に貼り付けて控えておいてください。各個人の環境によりハッシュ値が異なりますので、本手順に記載されているものは利用しないようにしてください。
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join master:6443 --token hr5r0w.0xc9a1apahzyu12p \
    --discovery-token-ca-cert-hash sha256:d7905e07bfb85989bfc21fec2356f887f93ebfaa202682aef8af1b5c09e75e64
```

* kubeadm コマンドから Kubernetes クラスタを操作できるようクレデンシャル情報を登録します。
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> **Note**
> Kubernetes の操作には、`kubectl` コマンドを利用します。概要については、[kubectlの概要](https://kubernetes.io/ja/docs/reference/kubectl/overview/)を参照してください。

* Kubernetes クラスタ情報を確認してみましょう。**Kubernetes master** および **KubeDNS** が動作していることが確認できます。
```bash
$ kubectl cluster-info
```

* 次に、Workshop で作成した Kubernetes クラスタの Version を確認します。2020/07 時点の最新は、Kubernetes v1.18 です。Kubernetes は四半期に一度マイナーアップデートが実施されておりますので、実施時期によっては Version が異なる可能性があります。
```bash
$ kubectl version
```

* それではクラスタノードの情報を確認してみましょう。現在時点で、STATUS が NotReady であることは問題ありません。
```bash
$ kubectl get nodes
NAME       STATUS     ROLES    AGE     VERSION
master     NotReady   master   7m35s   v1.18.6
```

* Kubernetes 上で動作する Pod 同士が通信できるように、CNI の Calico をインストールしましょう。
```bash
$ kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

* Calico Pod がデプロイされたことを確認しましょう。  
  - タイミングによっては、STATUS が Running ではなく、Init である場合があります。その場合には、少し時間をおいてから再度以下のコマンドを実行してみてください。
  - 本手順と表示されている NAME と、実行結果の NAME が違うことに気が付いたかもしれません。NAME の後ろ5文字は乱数が付与されているため、それぞれの実行環境により異なっていることが正常ですので問題ありません。 
```bash
$ kubectl get pods -n kube-system -l k8s-app=calico-node
NAME                READY   STATUS    RESTARTS   AGE
calico-node-7cvmm   1/1     Running   0          39s
```

* クラスターノードの情報を再度確認してみましょう。次は、STATUS が Ready になっているはずです。
```bash
# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   12m   v1.18.6
```

### Step.6 Worker Node をクラスタに追加する
* Step.5 で作成したクラスタに、Worker Node を追加します。本手順は、Worker Node のみで実行しますので、Master Node では実行しないでください。
* Step.5 で `kubeadm join` コマンドが表示されていたことを覚えていますか。本手順では、`kubeadm join` コマンドを使用して、クラスタに Worker Node を追加していきます。
```bash
$ sudo kubeadm join master:6443 --token hr5r0w.0xc9a1apahzyu12p --discovery-token-ca-cert-hash sha256:d7905e07bfb85989bfc21fec2356f887f93ebfaa202682aef8af1b5c09e75e64
```

以下の表示が出力されれば、クラスタへの追加は完了です。
```bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### Step.7 クラスタ状態を確認
この Section の最後に、構築したクラスタの状態を確認して完了にします。この Step の作業は、Master Node で実行してください。  
Worker Node で作業を実施すると、コマンド操作が失敗するはずです。それは、Step.5 で Master Node のみ Kubernetes のクレデンシャル情報を登録しており、Worker Node には登録していないからです。

* クラスタノードの状態を確認します。Master Node と Worker Node がそれぞれ表示され、STATUS が Ready であることを確認します。
```bash
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   23m   v1.18.6
worker01   Ready    <none>   65s   v1.18.6
worker02   Ready    <none>   59s   v1.18.6
```

* クラスタで動作している Pod を確認します。全ての Pod の STATUS が RUNING になっていることを確認しましょう。
```bash
$ kubectl get pod -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-59877c7fb4-plzvd   1/1     Running   0          20m
kube-system   calico-node-9b6cl                          1/1     Running   0          20m
kube-system   calico-node-hz4fp                          1/1     Running   0          7m27s
kube-system   coredns-66bff467f8-l7bxt                   1/1     Running   0          31m
kube-system   coredns-66bff467f8-wpb6t                   1/1     Running   0          31m
kube-system   etcd-master                                1/1     Running   0          32m
kube-system   kube-apiserver-master                      1/1     Running   0          32m
kube-system   kube-controller-manager-master             1/1     Running   0          32m
kube-system   kube-proxy-7794r                           1/1     Running   0          31m
kube-system   kube-proxy-zzcpf                           1/1     Running   0          7m27s
kube-system   kube-scheduler-master                      1/1     Running   0          32m
```

* クラスタで動作しているオブジェクトを全て確認したい場合には、以下のコマンドを実行します。表示されている内容が分からなくても、問題ありません。Section.2 で説明します。
```bash
$ kubectl get all -A
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-59877c7fb4-plzvd   1/1     Running   0          24m
kube-system   pod/calico-node-9b6cl                          1/1     Running   0          24m
kube-system   pod/calico-node-hz4fp                          1/1     Running   0          11m
kube-system   pod/coredns-66bff467f8-l7bxt                   1/1     Running   0          35m
kube-system   pod/coredns-66bff467f8-wpb6t                   1/1     Running   0          35m
kube-system   pod/etcd-master                                1/1     Running   0          36m
kube-system   pod/kube-apiserver-master                      1/1     Running   0          36m
kube-system   pod/kube-controller-manager-master             1/1     Running   0          36m
kube-system   pod/kube-proxy-7794r                           1/1     Running   0          35m
kube-system   pod/kube-proxy-zzcpf                           1/1     Running   0          11m
kube-system   pod/kube-scheduler-master                      1/1     Running   0          36m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  36m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   36m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/calico-node   2         2         2       2            2           beta.kubernetes.io/os=linux   24m
kube-system   daemonset.apps/kube-proxy    2         2         2       2            2           kubernetes.io/os=linux        36m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           24m
kube-system   deployment.apps/coredns                   2/2     2            2           36m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-59877c7fb4   1         1         1       24m
kube-system   replicaset.apps/coredns-66bff467f8                   2         2         2       35m
```

### Step.8 まとめ
Section.1 では、kubernetes デプロイツールである kubeadm を利用して、Kubernetes をインストールしました。  
おおよそ1～2時間程で構築できたのではないでしょうか。初期の頃は Kubernetes を一から手動で構築していたため、非常に時間のかかる作業でしたが、現在は Kubernetes デプロイツールを利用することで構築作業を簡略化し短時間で構築することができるようになりました。  
今後、Kubernetes を構築する場合には、積極的に Kubernetes デプロイツールを利用するにしましょう。

## 追加情報
Kubernetes を一から手動で構築してみたい方には、以下の Github ページへアクセスし、手動での構築に挑戦してみてください。  
[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---
お疲れさまでした。次の [Section.2-1](../Section2-1/Readme.md) へ進みましょう。
