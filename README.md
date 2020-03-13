# k8s
> 透過kubeadm學習架設k8s cluster


## 環境

* vagrant + Ubuntu 18.04
* docker + kubeadm


## 安裝

```
export ENV APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=DontWarn
export DEBIAN_FRONTEND=noninteractive

apt-get update

apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

sleep 3 && usermod -aG docker vagrant

kubeadm config images pull
```


## 建置cluster
* 設定CNI - 使用flannel並指定 pod-network-cidr 為 10.244.0.0/16
* 設定對外ip - apiserver-advertise-address

```
sudo swapoff -a

sudo kubeadm init --pod-network-cidr=10.244.0.0/16  --apiserver-advertise-address=192.168.2.11

```

安裝完成

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

sudo kubeadm join 192.168.2.11:6443 --token 5chnod.0n5do4kcml8ff8ov \
    --discovery-token-ca-cert-hash sha256:951a812ae32d81c63934f898cfae02188812591c3a1e05d83d1cede0af1fb767
```

### 如何取得kubeadmin join的憑證

#### token

```
kubeadm token list | tail -n1 | awk '{print $1; exit}'
```

#### discovery-token-ca-cert-hash 

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```



### 設定 CNI

```
sudo sysctl net.bridge.bridge-nf-call-iptables=1

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```


## vagrant + virtualbox 踩雷經驗

### Node VM NAT IP：10.0.2.15

```
$ kubectl get node -o wide
NAME         STATUS     ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master   NotReady   master   2m28s   v1.17.4   10.0.2.15     <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
k8s-node   NotReady   <none>   22s     v1.17.4   10.0.2.15     <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
```
vagrant default VM NAT Gateway 相同，將造成網路封包路由異常

#### 修正kubectl

在 kubelet 配置文件 kubeadm-flags.env 加入指定的node-ip

```
KUBELET_KUBEADM_ARGS="--cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1 --resolv-conf=/run/systemd/resolve/resolv.conf --node-ip=192.168.2.11"
```

重新啟動 kubectl

```
systemctl daemon-reload && systemctl restart kubelet
```

```
$ kubectl get node -o wide
NAME         STATUS     ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master   NotReady   master   18m   v1.17.4   192.168.2.11   <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
k8s-node   NotReady   <none>   16m   v1.17.4   192.168.2.12   <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
```

目前  Status 狀況若是 NotReady，只需進行CNI的網路設定即可，如果選用flannel，請參考下方內容

### 網路設定：flannel

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#default-nic-when-using-flannel-as-the-pod-network-in-vagrant

調整方式

* 查看網路卡設定

```
$ ifconfig
...
enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.11  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::a00:27ff:fe97:7526  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:97:75:26  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 13  bytes 1006 (1.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...
```
<span style="color:red">--iface=enp0s8</span>



* 在flannel YAML檔加上指定網路卡名稱

```
...
  containers:
  - name: kube-flannel
    image: quay.io/coreos/flannel:v0.11.0-amd64
    command:
    - /opt/bin/flanneld
    args:
    - --ip-masq
    - --kube-subnet-mgr
    - --iface=enp0s8
...
```

* 檢查pods是否正常啟動
```
kubectl get all --all-namespaces
```


## 確認是否有異常狀況

```
$ kubectl get node -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master   Ready    master   24m   v1.17.4   192.168.2.11   <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
k8s-node-1   Ready    <none>   22m   v1.17.4   192.168.2.12   <none>        Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.8
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-6955765f44-4fhx8             1/1     Running   0          24m
kube-system   pod/coredns-6955765f44-5hrbc             1/1     Running   0          24m
kube-system   pod/etcd-k8s-master                      1/1     Running   0          24m
kube-system   pod/kube-apiserver-k8s-master            1/1     Running   0          24m
kube-system   pod/kube-controller-manager-k8s-master   1/1     Running   0          24m
kube-system   pod/kube-flannel-ds-amd64-62xkl          1/1     Running   0          56s
kube-system   pod/kube-flannel-ds-amd64-jh79h          1/1     Running   0          56s
kube-system   pod/kube-proxy-7q5l8                     1/1     Running   0          22m
kube-system   pod/kube-proxy-k4qld                     1/1     Running   0          24m
kube-system   pod/kube-scheduler-k8s-master            1/1     Running   0          24m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  24m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   24m

NAMESPACE     NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     2         2         2       2            2           <none>                        56s
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        56s
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        56s
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        56s
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        56s
kube-system   daemonset.apps/kube-proxy                2         2         2       2            2           beta.kubernetes.io/os=linux   24m

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   2/2     2            2           24m

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-6955765f44   2         2         2       24m
```


## 測試


### nginx

```
# start the pod running nginx
kubectl run --image=nginx nginx-app --port=80

# expose a port through with a service
kubectl expose deployment nginx-app --port=80 --name=nginx-http

```


### busybox

```
kubectl run busybox -it --image=busybox --restart=Never --rm

```


