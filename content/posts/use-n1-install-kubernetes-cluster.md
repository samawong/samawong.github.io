---
title: "Use Phicomm N1 install Kubernetes Cluster"
date: 2020-08-27T23:52:56+08:00
description: "本文使用斐讯N1搭建Kubernetes集群，以及在搭建过程中遇到的几处问题"
tags: 
- N1
- Kubernetes Cluster
categories:
- 旁通杂粮
draft: false
---
手里面有两台斐讯N1，一台一直做TV盒子使用，另一台吃灰中，最近做TV盒子的那台播放电视家的时候，动不动就返回主页，同时，开启HDMI CEC后，尽管索尼的遥控器可以直接无缝匹配N1，但遥控起来轻飘飘的， 所以还是找出了吃灰中的小米盒子3拿来做TV盒子使用，然后将这两台N1安装armbian，尝试着安装kubernetes cluster，以下为详细安装步骤： 
## 一、安装前准备
* N1机器，并安装Armbian，我这里安装的是 Armbian_5.77_Aml-s905_Ubuntu_bionic_default_5.0.2_20190401.img，都说稳定性好，U盘安装后并写入eMMC;
系统下载地址就不放了，网上有；安装armbian的步骤简单捋一下：
1. 先制作Armbian U盘，Mac下直接使用dd命令即可，制作完后，重新挂载U盘，修改U盘下的uEnv.ini文件内的dtb文件路径（U盘目录下有N1的dtb文件，找到后修改即可）
2. 降级N1系统并打开USB调试功能；
3. adb 连接N1，并使用 ` adb shell reboot update` 命令重启盒子，在盒子断电瞬间，拔掉电源线，然后插U盘（U盘插入靠近HDMI接口的那个上），不出意外的话应该就可以进入Armbian系统了；
4. root默认密码是1234，输入后会让你再次输入并连续两次输入新密码的操作来重置该root密码，在这一步你也可以选择添加一个普通用户来进行日常登录；
5. 修改root目录下的install.sh权限，` chmod a+x install.sh `，然后执行 ` nand-sata-install`命令来将系统写入eMMC，写入成功后` reboot `下就可拔掉U盘了；
6. 如果在上述步骤时N1使用的有线网络连接，后续可直接使用ssh登录N1来进行后续步骤，不是的话需要在登录系统后，使用` armbian-config`命令来调试并连接Wi-Fi网络；
7. 网络连接后，使用root用户ssh登录系统，修改源，可以使用` armbian-config`命令来直接修改，并修改host名称，改完后，` apt update && apt upgrade -y && apt autoremove -y`命令来一下，更新一下系统软件，若遇error错误
提示，重启下系统再次执行即可。

截止目前，我们对N1设备已安装来Armbian系统，并固定来IP地址，修改了主机名称，修改了软件源，更新了系统，差不多准备就结束了，开始下面的正式工作。

## 二、开始安装kunernetes 主节点软件
** 以下操作均在主节点机器上执行，可暂时将另一台主机关闭 **
* 修改hosts文件（/etc/hosts），在hosts内添加IP地址信息与主机名称对应信息，如：
```
127.0.0.1     locahost,host-m
192.168.0.100 locahost,host-m
192.168.0.101 host-02
```
* 配置设置docker的cgroup-driver为systemd,创建` /etc/docker/daemon.json `文件，文件内容如下：
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```
完成后保存，接着执行` sudo systemd daemon-reload`命令来重载，如果机器已经安装docker了，重启docker（` sudo systemctl restart docker `）

* 配置kubelet的cgroup-driver为systemd，同时需要关闭swap，但是，Armbian使用的是zram,不建议关闭，所以创建` /etc/default/kubelet `，内容如下：
```
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --fail-swap-on=false
```
ps:kubelet在启动时，会检测cgroup-driver的参数选项，要求均一致，所以在kubelet启动后，如果启动失败，可以通过启动日志查看原因；

* 安装docker,kubeadm,kubelet,kubectl
```
apt install -y docker.io kubeadm kubelet kubectl
```
*** 暂不考虑网络，假设网络非常好，我是在那啥路由器下操作的 ***
安装完成后，继续往下走。。。
## 初始化k8s集群
初始化命令` sudo kubeadm init `,但因为swap没有关闭，所以还需要添加一个选项：` --ignore-preflight-errors Swap`，所以完整的是：
` sudo kubeadm init --ignore-preflight-errors Swap`
初始化成功后，可以看到类似的提示消息：
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

kubeadm join 192.168.1.200:6443 --token gffngy.ihk640zujif06483 \
    --discovery-token-ca-cert-hash sha256:367fc272849a3786d741e2056db6336b4ac7c643e05a030e33f9e5157603b9e1 
```
然后可以通过以下命令查看集群信息：
```
sama@host-m:~$ sudo kubectl cluster-info   //获取集群信息
Kubernetes master is running at https://192.168.1.200:6443
KubeDNS is running at https://192.168.1.200:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

sama@host-m:~$ sudo kubectl get nodes   //获取节点
NAME     STATUS   ROLES    AGE   VERSION
host-m   Ready    master   39h   v1.18.8

sama@host-m:~$ sudo kubectl get pods --all-namespaces //获取所有namespaces的pod
```
在该步骤可能会出现问题，我把我遇到的问题列下来，如遇到可做参考：
1. `The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.`
类似如下信息：
```
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get http://localhost:10248/healthz: dial tcp 127.0.0.1:10248: connect: connection refused.
```
使用`sudo systemctl status kubectl` 查看`kubectl`的状态，发现：
```
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Thu 2020-08-26 17:44:39 CST; 2s ago
     Docs: https://kubernetes.io/docs/
  Process: 14476 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 14476 (code=exited, status=255)

Mar 28 09:44:39 k8s-node systemd[1]: kubelet.service: main process exited, code=exited, status=255/n/a
```
状态为`activating`,没有起来。Google一下，让检查`native-cgroupdriver`，所以我们查看一下下面几个地方：
```
sama@host-m:~$ sudo cat /var/lib/kubelet/kubeadm-flags.env 
KUBELET_KUBEADM_ARGS="--cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.2 --resolv-conf=/run/systemd/resolve/resolv.conf"
 
sama@host-m:~$ sudo cat /etc/default/kubelet
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --fail-swap-on=false


sama@host-m:~$ docker info
Client:
 Debug Mode: false

Server:
......
 Logging Driver: json-file
 Cgroup Driver: systemd
 Plugins:
  Volume: local
......

```
看配置里的 `Cgroup Driver` 的值是否一致，不一致的话修改上述说明中的两处配置文件。

2. Kubelet启动报错：node "node-m" not found
类似如下提示:
```
sama@host-m# sudo systemctl status kubelet -l

● kubelet.service - kubelet: The Kubernetes Node Agent
......

Aug 26 17:39:16 master kubelet[6161]: E0731 05:39:16.008683    6161 kubelet.go:2248] node "node-m" not found
Aug 26 17:39:16 master kubelet[6161]: E0731 05:39:16.109488    6161 kubelet.go:2248] node "node-m" not found
Aug 26 17:39:16 master kubelet[6161]: E0731 05:39:16.210368    6161 kubelet.go:2248] node "node-m" not found
```
先检查`kubelet.conf`的配置文件，查看配置中的地址是否为正确地址：
` sama@host-m# sudo cat /etc/kubernetes/kubelet.conf`
若以上地址正确，检查`/etc/hosts`内主机名称是否对应一致。若均无毛病，直接重启该设备后查看启动是否正常。
若地址不正确，需要修改一下几个配置文件：`/etc/kubernetes/admin.conf`,`/etc/kubernetes/controller-manager.conf`,`/etc/kubernetes/kubelet.conf`,`/etc/kubernetes/scheduler.conf`
`journalctl -xeu kubelet` 命令可以查看kubelet的日志。
## 安装flannel 
Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

1.安装前修改`/etc/kubernetes/manifests/kube-controller-manager.yaml`文件，添加两处参数，详细如下：
```
sama@host-m:~$ sudo cat /etc/kubernetes/manifests/kube-controller-manager.yaml 
[sudo] password for sama: 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --port=0
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --use-service-account-credentials=true
    - --allocate-node-cidrs=true             //新添内容
    - --cluster-cidr=10.244.0.0/16           //新添内容
```
2. 安装使用如下命令,最新版怼起来，
`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`


## 安装kubernetes-dashboard WebUI插件
### 安装
官方安装命令：
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml`
但这样安装有点问题：
1. 安装完后证书原因，除firfox浏览器外，其他浏览器均无法打开；
这个我按照官方说明，使用自签证书依旧无法解决，所以暂时使用火狐浏览器访问。
2. 安装完成后，kubernetes-dashboard service 在集群内部，无法再外部访问，需要将dashboard的443端口暴露出来，所以有四种方法：
- 第一方法：上面的`kubernetes-dashboard.yaml`文件，先保存下来，然后修改该文件，修改内容如下：
```
......
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort            //新加内容
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31111       //新加内容
  selector:
    k8s-app: kubernetes-dashboard

......
```
然后保存，再使用`kubectl apply -f kubernetes-dashboard.yaml`来进行安装；
- 第二种方法：在使用`kubectl apply -f 在线yaml地址`完成安装后，使用以下命令`kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard`来修改已运行的配置，
找到type字段，将ClusterIP修改为NodePort，wq保存后，可使用`kubectl get svc -n kubernetes-dashboard`来进行查看结果。
两种方法各有利弊，比如说安装失败需要重新调试参数时，第一种方法最合适，而且第二种方式所添加的端口不固定，需要使用`kubectl get svc`查看后才可以知晓端口。
- 第三种方法：使用` kubectl -n kubernetes-dashboard port-forward ${dashboard-pod-name} 31111:8443`，该方法应该可行，在后续的其他pod上使用该方法。
- 第四种方法：proxy,使用命令`sudo kubtctl proxy`,然后访问`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`；
### 登录
需要创建dashboard用户
1. 创建服务账户文件，我命名为`admin-sa.yml`，内容如下：
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

```
保存后，使用`sudo kubectl apply -f admin-sa.yml`来创建对象；

2. 创建集群规则绑定，我命名为`admin-rbac.yml`,内容如下：
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```
保存后，使用`sudo kubectl apply -f admin-rbac.yml`命令来创建该规则。

3. 使用` sudo kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')`命令来获取该用户的token：
```
sama@host-m:~$ sudo kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-gj8cb
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 29e446eb-7773-428d-beb6-2e9dc5850b05

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlNBdlVaY0d3NHVQRldKRUFGZ1VJRjhWOG1NUkNmV2gwelBaVy1xdS1mTk0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWdqOGNiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyOWU0NDZlYi03NzczLTQyOGQtYmViNi0yZTlkYzU4NTBiMDUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.AoA91kavZ_CrNp0gzpLagRAKDYml2nP1Eiumd_k0tBlPmbtZD1dN3GNoVFbaSjJm3omZReZ6qq3l-t7r6gWPg88WhAxHCQlWVz0AgwPVQvRGFNE4rHQHaCdqzvn5Hyk0i7W62_NHhvM8VWln-hfpPGKNvxho6hT3HdD2u22bX-F32NOKR15zu7N3MTXeaMflL3nbyzn6DSWlGHHTe1428LMXmPZZQjO7QEyVbh8mK6KxumncGgs6sTw8ubXvTG7fKS32Ydr69Q7NuGWCbstkEkfeH72gvXk79HXycqx32HZqoklZfNdmYehRSxLauTfzbYUPJZulma1zw3-sQ_sTyQ
```
然后使用该token值就可登录dashboard的UI界面了，权限正常。

![dashboard界面](https://res.cloudinary.com/samawong/image/upload/v1598600528/notes/k8s-install/ui_x8ctug.jpg "dashboard界面")


## 节点加入集群
节点主机依照上述文档，固定IP地址，修改主机名，修改hosts文件，apt update && apt upgrade,就可以安装docker,kubeadm,kubectl,kubelet了。
安装完后使用主节点`sudo kubeadm init`命令生成的`sudo kubeadm join 192.168.1.200:6443 --token gffngy.ihk640zujif06483 \
    --discovery-token-ca-cert-hash sha256:367fc272849a3786d741e2056db6336b4ac7c643e05a030e33f9e5157603b9e1 `来加入主节点。
添加成功后，可以在主节点的dashboard内可看到该节点，命令下也可以：
```
sama@host-m:~$ sudo kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
host-m   Ready    master   43h   v1.18.8
host-s   Ready    <none>   22h   v1.19.0
```
上述两台主机的VERSION不一致，host-s比host-m迟一天，正好这天发布了最新的V1.19版，暂时看版本不一致兼容没有问题。

如果上述命令的token忘记了，可以使用`sudo kubeadm token list`来获取token,默认，token有效期24h,如果没有token已过期，你可以使用
`sudo kubeadm token create`来创建一个，比如如下样子：
```
sama@host-m:~$ sudo kubeadm token create 
1azhxq.rbnzt28euxspwjdj
```
若没有后面`--discovery-token-ca-cert-hash`的值，可以使用如下命令来获取：
```
sama@host-m:~$ sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
>    openssl dgst -sha256 -hex | sed 's/^.* //'
367fc272849a3786d741e2056db6336b4ac7c643e05a030e33f9e5157603b9e1
```

## Helm 包管理工具
+ 安装版本
```
sama@host-m:~$ helm version
version.BuildInfo{Version:"v3.3.0", GitCommit:"8a4aeec08d67a7b84472007529e8097ec3742105", GitTreeState:"dirty", GoVersion:"go1.14.7"}
```
+ 安装方式：
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 -o get_helm.sh && sudo chmod +x get_helm.sh && sudo ./get_helm.sh
```
+ 使用：
` helm create hello-helm ` //创建名为hello-helm的Chart
` helm install helm-example hello-helm ` //使用hello-helm来安装一个名为helm-example的应用,如下所示：
```
sama@host-m:~$ helm install helm-example hello-helm
NAME: helm-example
LAST DEPLOYED: Fri Aug 28 14:24:30 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=hello-helm,app.kubernetes.io/instance=helm-example" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```
根据上述提示，执行下面的命令：
```
sama@host-m:~$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=hello-helm,app.kubernetes.io/instance=helm-example" -o jsonpath="{.items[0].metadata.name}")
sama@host-m:~$ kubectl --namespace default port-forward $POD_NAME 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```
然后在浏览器内输入http://127.0.0.1:8080，可以访问我们刚新建的chart了。（N1没有浏览器，所以，两个窗口，curl来一下）
![helm 转发端口](https://res.cloudinary.com/samawong/image/upload/v1598600526/notes/k8s-install/helm_zwxzhz.jpg "helm 转发端口")

其他命令请参阅[helm官网](https://github.com/helm/helm).

## 部分常用命令
+ kubectl replace --force -f kubernetes-dashboard-arm64.yaml  //重新导入yaml配置，可作为集群重启pod方式
+ kubectl get pods --all-namespaces //获取所有pod列表 
+ kubectl -n <namespace> get/edit/delete pod (pod-name) //获取该namespace下pod列表
+ kubectl -n <namespace> get/edit svc (pod-name) //获取该namespace下网络情况
+ kubectl describe pod (pod-name) -n <namespace> //获取该namespace 下（pod）信息
+ kubectl logs -n <namespace> (pod-name)  //获取该namespace下pod-name的日志


至此，k8s的集群简易说明到此结束。

参考链接：
1. [How To Create Admin User to Access Kubernetes Dashboard](https://computingforgeeks.com/create-admin-user-to-access-kubernetes-dashboard/)
2. [kubernetes/dashboard](https://github.com/kubernetes/dashboard)
3. [user/access-control/creating-sample-user.md](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)
4. [Kubernetes Dashboard的安装与坑](https://www.jianshu.com/p/c6d560d12d50)
5. [v1.14.0 problem : It seems like the kubelet isn't running or healthy](https://github.com/kubernetes/kubernetes/issues/75803)
6. [安装笔记K8S 1.15.0版本遇到的一些错误汇总（部分参考网络上经验，基本可以解决问题）](https://www.cnblogs.com/taoweizhong/p/11545953.html)
7. [Kubernetes 部署失败的 10 个最普遍原因（Part 1）](http://dockone.io/article/2247)
8. [构建 arm64 架构 k8s 集群（phicomm-n1）](https://sbilly.github.io/post/howto-setup-kubernetes-cluster-on-armbian-linux-based-on-phicomm-n1-arm64-sbc/)
9. [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)
10. [Unable to update cni config: No networks found in /etc/cni/net.d](https://github.com/kubernetes/kubernetes/issues/54918)
11. [重启 Kubernetes Pod的几种方式](https://segmentfault.com/a/1190000020675199)
12. [helm/helm](https://github.com/helm/helm)