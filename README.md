<!-- @format -->

## Kubernetes CI&CD Pipline with GithubActions and Flux

### 目次

1. Kubernetes クラスター構築  
   1.1 仮想マシン構築  
   1.2 Kubernetes のクラスター構築（マスターノード）  
   1.3 Kubernetes のクラスター構築（ワーカーノード#1）  
   1.4 Kubernetes のクラスター構築（ワーカーノード#2）
2. Flux 構築  
   2.1 リポジトリ説明  
   2.2 事前準備（Git&Curl インストール）  
   2.3 Flux インストール  
   2.4 イメージタグ監視コンポーネントインストール
3. アプリケーション確認
4. Weave GitOps 構築  
   4.1 Weave GitOps 構築  
   4.2 Weave GitOps GUI 確認

![image](https://github.com/ns1root/FluxApp/blob/main/image/image.png)

### 1.Kubernetes クラスター構築

#### 1.1 仮想マシン構築（詳細は省略、どのクラウドでも OK）

・計３台（マスターノード１台、ワーカーノード２台）  
・OS：CentOS7.9  
・Spec: 2vCPU, 4GB Mem, 40GB Disk  
・マスターノードの Global IP は固定の IP で払い出す  
・全てのノードのプライベート IP は固定の IP で払い出す

#### 1.2 Kubernetes のクラスター構築（マスターノード）

##### 1.2.1 OS Parameter 設定

```bash
#update repository
sudo su -
sudo yum update -y

#swap off
sudo swapoff -a

#check /etc/fstab to make sure permanently disable swap in centos
#sudo vi /etc/fstab

#check hostname, uuid and eth0 network
sudo hostname -s
sudo cat /sys/class/dmi/id/product_uuid
sudo ip link show eth0

#let iptables see bridged traffic
sudo modprobe br_netfilter
sudo modprobe overlay
sudo lsmod | grep -e br_netfilter -e overlay
#CLI result
#overlay                91659  0
#br_netfilter           22256  0
#bridge                151336  1 br_netfilter

#enable settings after reboot
sudo cat >/etc/modules-load.d/containerd.conf <<EOF
br_netfilter
overlay
EOF

#setup kernel parameter
sudo cat >/etc/sysctl.d/99-containerd.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

#apply kernel parameter
sudo sysctl --system
sudo sysctl -a | grep -e bridge-nf-call-ip -e ip_forward
```

##### 1.2.2 Kubernetes インストール

```bash
#Install Containerd
sudo yum install -y yum-utils

sudo yum-config-manager \
--add-repo \
https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install containerd.io -y

sudo rm /etc/containerd/config.toml

sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd
#CLI result
#● containerd.service - containerd container runtime
#   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; vendor preset: disabled)
#   Active: active (running) since Thu 2022-09-22 13:19:49 CST; 941ms ago

#Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#Install Kubectl
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

#Kubeadm Initialization
#PRIVATEIPとGLOBALIPを実際のマスターノードの値に置き換える
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address PRIVATEIP --apiserver-cert-extra-sans GLOBALIP

#Terminal上に表示された以下の行をメモする
#kubeadm join 172.16.0.2:6443 --token 5267cm.ut645x4g1iwzf37d \
#	--discovery-token-ca-cert-hash sha256:5e8b0f1555d852309fbdde0f7f675f02a1533a1bfdeb66cd35e03cdf46d50935

#Exit from root shell
exit

#Tips
#To start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#Alternatively, if you are the root user, you can run:
export KUBECONFIG=/etc/kubernetes/admin.conf

#Flannel Network Deploy
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
#CLI result
#namespace/kube-flannel created
#clusterrole.rbac.authorization.k8s.io/flannel created
#clusterrolebinding.rbac.authorization.k8s.io/flannel created
#serviceaccount/flannel created
#configmap/kube-flannel-cfg created
#daemonset.apps/kube-flannel-ds created
```

##### 1.2.3 インストール確認

```bash
#Check Installation
kubeadm version
#CLI result
#[ecs-user@k-01 ~]$ kubeadm version
#kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:32:18Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}

kubectl version
#CLI result
#[ecs-user@k-01 ~]$ kubectl version
#WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short. Use --output=yaml|json to get the full version.
#Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:33:49Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}
#Kustomize Version: v4.5.7
#Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:27:13Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}

kubelet --version
#CLI result
#[ecs-user@k-01 ~]$ kubelet --version
#Kubernetes v1.25.2

kubectl get nodes
#CLI result
#[ecs-user@k-01 ~]$ kubectl get nodes
#NAME STATUS ROLES AGE VERSION
#k-01 Ready control-plane 87s v1.25.2

kubectl get pods -A
#CLI result
#[ecs-user@k-01 ~]$ kubectl get pods -A
#NAMESPACE NAME READY STATUS RESTARTS AGE
#kube-flannel kube-flannel-ds-2kg56 1/1 Running 0 25s
#kube-system coredns-565d847f94-5lfhv 1/1 Running 0 71s
#kube-system coredns-565d847f94-jw5bn 1/1 Running 0 71s
#kube-system etcd-k-01 1/1 Running 0 86s
#kube-system kube-apiserver-k-01 1/1 Running 0 86s
#kube-system kube-controller-manager-k-01 1/1 Running 0 86s
#kube-system kube-proxy-xnl47 1/1 Running 0 71s
#kube-system kube-scheduler-k-01 1/1 Running 0 86s
```

#### 1.3 Kubernetes のクラスター構築（ワーカーノード#1）

##### 1.3.1 OS Parameter 設定

```bash
#update repository
sudo su -
sudo yum update -y

#swap off
sudo swapoff -a

#check /etc/fstab to make sure permanently disable swap in centos
#sudo vi /etc/fstab

#check hostname, uuid and eth0 network
sudo hostname -s
sudo cat /sys/class/dmi/id/product_uuid
sudo ip link show eth0

#let iptables see bridged traffic
sudo modprobe br_netfilter
sudo modprobe overlay
sudo lsmod | grep -e br_netfilter -e overlay
#CLI result
#overlay                91659  0
#br_netfilter           22256  0
#bridge                151336  1 br_netfilter

#enable settings after reboot
sudo cat >/etc/modules-load.d/containerd.conf <<EOF
br_netfilter
overlay
EOF

#setup kernel parameter
sudo cat >/etc/sysctl.d/99-containerd.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

#apply kernel parameter
sudo sysctl --system
sudo sysctl -a | grep -e bridge-nf-call-ip -e ip_forward
```

##### 1.3.2 Kubernetes インストール

```bash
#Install Containerd
sudo yum install -y yum-utils

sudo yum-config-manager \
--add-repo \
https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install containerd.io -y

sudo rm /etc/containerd/config.toml

sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd
#CLI result
#● containerd.service - containerd container runtime
#   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; vendor preset: disabled)
#   Active: active (running) since Thu 2022-09-22 13:19:49 CST; 941ms ago

#Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#Install Kubectl
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

#Exit from root shell
exit
```

##### 1.3.3 インストール確認

```bash
#Check Installation
kubeadm version
#CLI result
#[ecs-user@k-01 ~]$ kubeadm version
#kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:32:18Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}

kubectl version
#CLI result
#[ecs-user@k-01 ~]$ kubectl version
#WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short. Use --output=yaml|json to get the full version.
#Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:33:49Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}
#Kustomize Version: v4.5.7
#Server Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:27:13Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"linux/amd64"}

kubelet --version
#CLI result
#[ecs-user@k-01 ~]$ kubelet --version
#Kubernetes v1.25.2
```

##### 1.3.4 マスターノード参加

```bash
#Kubeadm Join ルートユーザーとして実行
#先ほどマスターノードにてメモしたKubeadm Join ｘｘｘを貼り付けて実行
#メモ忘れた場合はマスターノードにて以下のコマンドを実行する
#kubeadm token create --print-join-command

#ワーカーノード参加確認（マスターノードにて実行）
kubectl get nodes
#CLI result
#[ecs-user@k-01 cluster]$ kubectl get nodes
#NAME   STATUS   ROLES           AGE     VERSION
#k-01   Ready    control-plane   3h11m   v1.25.2
#k-02   Ready    <none>          130m    v1.25.2
```

#### 1.4 Kubernetes のクラスター構築（ワーカーノード#2）

```
#ワーカーノード１と同じ手順を実施するため、省略
```

### 2.Flux 構築

#### 2.1 リポジトリ説明

| フォルダ          | 説明                                                                                                                             | 備考 |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------- | ---- |
| .github/workflows | GithubActions を動かす YAML 定義ファイル<br>・npm build<br>・create docker image<br>・upload docker image to github packages<br> |
| docker            | Docker Image ビルド用の Dockerfile                                                                                               |      |
| kustomize         | Flux と連携する Kustomization ファイル                                                                                           |      |
| ほかのファイル    | デフォルトの React App ファイル                                                                                                  |

#### 2.2 事前準備（Git&Curl インストール）

##### 2.2.1 Git インストール

```bash
#1.依存関係のあるライブラリをインストール
sudo su -
sudo yum -y install gcc curl-devel expat-devel gettext-devel openssl-devel zlib-devel perl-ExtUtils-MakeMaker autoconf
#2.インストールに適切な場所に移動
cd /usr/local/src/
#3.サイトから Git の圧縮ファイルをダウンロード
#公式サイト https://mirrors.edge.kernel.org/pub/software/scm/git/
sudo wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.9.5.tar.gz
#4.ファイルを解凍
sudo tar xzvf git-2.9.5.tar.gz
#5.圧縮ファイルを削除
sudo rm -rf git-2.9.5.tar.gz
#6.解凍した Git ディレクトリに移動
cd git-2.9.5/
#7.make コマンドでインストール
sudo make prefix=/usr/local all
sudo make prefix=/usr/local install
#8.バージョン確認コマンド
git --version
```

##### 2.2.2 Curl インストール

```bash
#1.依存関係のあるライブラリをインストール
sudo yum update -y
sudo yum install wget gcc openssl-devel -y
#2.インストールに適切な場所に移動
cd /usr/local/src/
#3.サイトから Git の圧縮ファイルをダウンロード
#公式サイト https://curl.se/download.html
sudo wget https://curl.se/download/curl-7.85.0.tar.gz
#4.ファイルを解凍
gunzip -c curl-7.85.0.tar.gz | tar xvf -
#5.圧縮ファイルを削除
sudo rm -rf curl-7.85.0.tar.gz
#6.解凍した Git ディレクトリに移動
cd curl-7.85.0
#7.make コマンドでインストール
./configure --with-ssl
sudo make prefix=/usr/local all
sudo make prefix=/usr/local install
#8.バージョン確認コマンド
curl -V

#Exit from root shell
exit

#最新になっていない場合
#which -a curl
#sudo rm /usr/bin/curl
#sudo ln -s /usr/local/bin/curl /usr/bin/curl
#curl -V
```

#### 2.3 Flux インストール

```bash
#インストール先：マスターノード
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version

#Export Github user&token
export GITHUB_USER=<your-username>
export GITHUB_TOKEN=<your-token>

#インストール前確認
flux check --pre
#[ecs-user@k-01 ~]$ flux check --pre
#► checking prerequisites
#✔ Kubernetes 1.25.2 >=1.20.6-0
#✔ prerequisites checks passed

#flux bootstrap
#--repositoryはfluxコンポーネントデプロイされる予定のリポジトリ名を指定、リポジトリは自動作成される
flux bootstrap github \
 --components-extra=image-reflector-controller,image-automation-controller \
 --owner=$GITHUB_USER \
 --repository=demo-flux \
 --branch=main \
 --path=./cluster \
 --read-write-key \
 --personal

#flux bootstrap失敗また、Fluxをインストールし直す場合は、以下のSecret Keyを削除
#kubectl delete secret -n flux-system flux-system

#Github リポジトリ連携
#前項目で指定したリポジトリ
git clone https://github.com/$GITHUB_USER/demo-flux
cd demo-flux

#Flux監視するアプリケーションリポジトリを追加
#--urlは本リポジトリを利用しています。（別のアプリケションリポジトリを指定しても問題ない）
flux create source git react \
 --url=https://github.com/ns1root/FluxApp \
 --branch=main \
 --interval=30s \
 --export > ./cluster/react-repo-source.yaml

#Kustomizationパスを追加する
flux create kustomization react \
 --target-namespace=default \
 --source=react \
 --path="./kustomize/base" \
 --prune=true \
 --interval=5m \
 --export > ./cluster/react-kustomization.yaml

#Git commit && Flux手動反映
git add -A && git commit -m "-" && git push
flux reconcile kustomization flux-system --with-source

#Kustomizationのリリース状態確認
flux get kustomizations --watch
#CLI result
#[ecs-user@k8s-01 demo-flux]$ flux get kustomizations --watch
#NAME       	REVISION    	SUSPENDED	READY	MESSAGE
#flux-system	main/744892a	False    	True 	Applied revision: main/744892a
#react	main/a44277b	False	True	Applied revision: main/a44277b

kubectl get hpa,deploy,svc,pod
#CLI result
#[ecs-user@k-01 demo-flux]$ kubectl get hpa,deploy,svc,pod
#NAME READY UP-TO-DATE AVAILABLE AGE
#deployment.apps/react 1/1 1 1 15s
#NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE
#horizontalpodautoscaler.autoscaling/react Deployment/react <unknown>/99% 1 3 1 15s
#NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
#service/kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 139m
#service/react NodePort 10.106.46.220 <none> 8080:30001/TCP 15s
#NAME READY STATUS RESTARTS AGE
#pod/react-56c9d8dfd6-qtmz6 1/1 Running 0 15s

#現在のImageバージョン確認
kubectl get deploy react -oyaml | grep 'image:'
#CLI result
#[ecs-user@k-01 demo-flux]$ kubectl get deploy react -oyaml | grep 'image:' - image: ghcr.io/ns1root/react:1.0.0

#手順は省略
#GithubActionsにて新しい1.0.1バージョンのImageを作成

#Fluxのイメージ監視設定を追加
#イメージリポジトリを追加
flux create image repository react \
--image=ghcr.io/ns1root/react \
--interval=1m \
--export > ./cluster/react-image-registry.yaml

#イメージポリシーオブジェクトを追加
flux create image policy react \
--image-ref=react \
--select-semver=1.0.x \
--export > ./cluster/react-image-policy.yaml

#Git commit && Flux手動反映
git add -A && git commit -m "-" && git push
flux reconcile kustomization flux-system --with-source

#Imageリポジトリが正しく作成される
flux get image repository
#CLI result
#[ecs-user@k-01 demo-flux]$ flux get image repository
#NAME LAST SCAN SUSPENDED READY MESSAGE
#react 2022-09-22T10:46:54+08:00 False True successful scan, found 2 tags

#Imageポリシーが正しく作成される
flux get image policy
#CLI result
#[ecs-user@k-01 demo-flux]$ flux get image policy
#NAME LATEST IMAGE READY MESSAGE
#react ghcr.io/ns1root/react:1.0.1 True Latest image tag for 'ghcr.io/ns1root/react' resolved to: 1.0.1

#ImageUpdateAutomation オブジェクト追加
flux create image update flux-system \
--git-repo-ref=flux-system \
--git-repo-path="./cluster" \
--checkout-branch=main \
--push-branch=main \
--author-name=fluxcdbot \
--author-email=fluxcdbot@users.noreply.github.com \
--commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
--export > ./cluster/flux-system-automation.yaml

#react-kustomization.yamlファイルに以下を追加する
#参考サイト
#https://fluxcd.io/flux/guides/image-update/#configure-image-update-for-custom-resources
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
name: podinfo
namespace: default
spec:
  images:
  - name: ghcr.io/ns1root/react
    newName: ghcr.io/ns1root/react # {"$imagepolicy": "flux-system:react:name"}
    newTag: 1.0.0 # {"$imagepolicy": "flux-system:react:tag"}

#Git commit && Flux手動反映
git add -A && git commit -m "-" && git push
flux reconcile kustomization flux-system --with-source

#Image Updateオブジェクトが正しく作成される
kubectl get imageupdateautomation -n flux-system
#CLI result
#[ecs-user@k-01 cluster]$ kubectl get imageupdateautomation -n flux-system
#NAME LAST RUN
#flux-system

#Image Update情報を確認
flux get image update
#CLI result
#[ecs-user@k-01 cluster]$ flux get image update
#NAME LAST RUN SUSPENDED READY MESSAGE
#flux-system 2022-09-22T11:03:32+08:00 False True committed and pushed 110d73a09cba5fad2bb3e23208309d9a0d3bab10 to main
#flux-system 2022-09-22T11:04:42+08:00 False True no updates made; last commit 110d73a at 2022-09-22T03:03:32Z

#DeploymentのImageバージョン確認（1.0.1になっていればOK）
kubectl get deploy react -oyaml | grep 'image:'
#CLI result
#[ecs-user@k-01 cluster]$ kubectl get deploy react -oyaml | grep 'image:' - image: ghcr.io/ns1root/react:1.0.1

#エラー出る時確認するログ
#kubectl logs -n flux-system -l app=image-automation-controller
```

### 3. アプリケーション確認

```bash
#SecurityGroupの30001ポートを開け、マスターノードのPublicIPでアクセスする
[https://](http://masternodepublicip:30001)
```

### 4. Weave GitOps 構築

#### 4.1 Weave GitOps 構築（マスターノード）

```bash
#demo-flux/clusterリポジトリに以下のファイル（weave-installation.yaml）を追加
#username & passwordhashは任意
#passwordHashはbcrypt利用https://bcrypt-generator.com/
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ww-gitops
  namespace: flux-system
spec:
  chart:
    spec:
      chart: weave-gitops
      sourceRef:
        kind: HelmRepository
        name: ww-gitops
  interval: 1h0m0s
  values:
    adminUser:
      create: true
      username: <UPDATE>
      passwordHash: <UPDATE>
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ww-gitops
  namespace: flux-system
spec:
  interval: 1h0m0s
  type: oci
  url: oci://ghcr.io/weaveworks/charts

#Git commit && Flux手動反映
git pull
git add -A && git commit -m "-" && git push
flux reconcile kustomization flux-system --with-source

kubectl get pods -A
#CLI result
#NAMESPACE      NAME                                          READY   STATUS    RESTARTS   AGE
#flux-system    ww-gitops-weave-gitops-5758f7cd44-qq27t       1/1     Running   0          34s

#デフォルトはCluster IPにて公開されいますが、
#Onpremisクラスターのため、NodePortを使ってWeaveを公開する
kubectl describe pod ww-gitops-weave-gitops-5758f7cd44-qq27t -n flux-system
#Label情報をメモする
#CLI result
#Labels:           app.kubernetes.io/instance=ww-gitops
#                  app.kubernetes.io/name=weave-gitops

#マスターノードに以下のweave-service.yamlファイルを作成する
apiVersion: v1
kind: Service
metadata:
  name: weave
  namespace: flux-system
spec:
  type: NodePort
  selector:
    app.kubernetes.io/instance: ww-gitops
  ports:
    - port: 9001
      nodePort: 30009

#serviceを適用する
kubectl apply -f weave-service.yaml
```

#### 4.2 Weave GitOps GUI 確認

```bash
#SecurityGroupの30009ポートを開け、マスターノードのPublicIPでアクセスする
[https://](http://masternodepublicip:30009)
```
