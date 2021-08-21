# 	kubeedge 本地环境安装

# 一、系统配置

## 1.1 集群环境

| 主机名      | 角色   | IP             | 工作负载               |
| ----------- | ------ | -------------- | ---------------------- |
| devad-cloud | 云端   | 192.168.56.101 | k8s、docker、cloudcore |
| devad-edge1 | 边缘端 | 192.168.56.104 | docker、edgecore       |
| devad-edge2 | 边缘端 | 192.168.56.105 | docker、edgecore       |

虚拟机安装ssh服务

```bash
$ sudo apt-get install openssh-server
```

###  安装docker

Docker 安装可参考[这里](https://docs.docker.com/engine/install/ubuntu/)

### Uninstall old versions

Older versions of Docker were called `docker`, `docker.io`, or `docker-engine`. If these are installed, uninstall them:

```sh
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

#### SET UP THE REPOSITORY

1. Update the `apt` package index and install packages to allow `apt` to use a repository over HTTPS:

   ```sh
   $ sudo apt-get update
   
   $ sudo apt-get install \
       apt-transport-https \
       ca-certificates \
       curl \
       gnupg-agent \
       software-properties-common
   ```

2. Add Docker’s official GPG key:

   ```sh
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   ```

   Verify that you now have the key with the fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, by searching for the last 8 characters of the fingerprint.

   ```sh
   $ sudo apt-key fingerprint 0EBFCD88
   
   pub   rsa4096 2017-02-22 [SCEA]
         9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
   uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
   sub   rsa4096 2017-02-22 [S]
   ```

3. Use the following command to set up the **stable** repository. To add the **nightly** or **test** repository, add the word `nightly` or `test` (or both) after the word `stable` in the commands below. [Learn about **nightly** and **test** channels](https://docs.docker.com/engine/install/).

   > **Note**: The `lsb_release -cs` sub-command below returns the name of your Ubuntu distribution, such as `xenial`. Sometimes, in a distribution like Linux Mint, you might need to change `$(lsb_release -cs)` to your parent Ubuntu distribution. For example, if you are using `Linux Mint Tessa`, you could use `bionic`. Docker does not offer any guarantees on untested and unsupported Ubuntu distributions.

   ```sh
   $ sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
   ```

   

#### INSTALL DOCKER ENGINE

1. Update the `apt` package index, and install the *latest version* of Docker Engine and containerd, or go to the next step to install a specific version:

   ```sh
    $ sudo apt-get update
    $ sudo apt-get install docker-ce docker-ce-cli containerd.io
   ```
   
2. Verify that Docker Engine is installed correctly by running the `hello-world` image.

   ```sh
   $ sudo docker run hello-world
   ```

   This command downloads a test image and runs it in a container. When the container runs, it prints an informational message and exits.

   Docker Engine is installed and running. The `docker` group is created but no users are added to it. You need to use `sudo` to run Docker commands. Continue to [Linux postinstall](https://docs.docker.com/engine/install/linux-postinstall/) to allow non-privileged users to run Docker commands and for other optional configuration steps.

#### 安装docker后配置

1. 创建 docker组

   ```bash
   $ sudo groupadd docker
   ```

2. 将您的用户添加到 group.docker

   ```sh
   $ sudo usermod -aG docker $USER
   ```

3. 在 Linux 上，你也可以运行以下命令来激活对组的更改:

   ```bash
   $ newgrp docker 
   ```

4. 验证您可以在没有sudo 的情况下运行命令

   ```sh
   $ docker run hello-world
   ```

5. docker配置国内镜像加速

   ```sh
   $ sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://dh7kdsrx.mirror.aliyuncs.com"]
   }
   EOF
   
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```


# 二、cloud节点部署K8s

### 使用kubeadm安装k8s

#### 关闭swap

编辑 /etc/fstab 文件，注释掉swap分区挂载的行，示例：

```
# swap was on /dev/sda1 during installation
# UUID=42d32c46-d6ba-4a0d-86f4-18532069a8c1 none            swap    sw              0       0
```

再执行 ` swapoff -a`

```sh
$ swapoff -a
```

> 安装[kubeadm](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

#### 配置源

注意点：k8s添加国内源，这里选择的是阿里云的：

```sh
$ sudo cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

$ sudo apt-get update
```

如果出现

```sh
The following signatures couldn't be verified because the public key is not available
```

则执行下面命令，添加 key。

```sh
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add 
```

#### 安装 kubeadm、kubelet 和 kubectl

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
# 安装指定版本
# sudo apt-get install -y kubeadm=1.18.12-00 kubelet=1.18.12-00 kubectl=1.18.12-00
sudo apt-mark hold kubelet kubeadm kubectl
```

用命令查看版本当前kubeadm对应的k8s组件镜像版本，如下：

```sh
devad@devad-VirtualBox:~$ kubeadm config images list
W1205 23:45:29.955029    6229 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
k8s.gcr.io/kube-apiserver:v1.19.4
k8s.gcr.io/kube-controller-manager:v1.19.4
k8s.gcr.io/kube-scheduler:v1.19.4
k8s.gcr.io/kube-proxy:v1.19.4
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0

```

#### 拉取镜像 

用命令查看版本当前kubeadm对应的k8s组件镜像版本，如下：

```sh
devad@devad-VirtualBox:~$ kubeadm config images list
W1205 23:56:50.666011    8443 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
k8s.gcr.io/kube-apiserver:v1.19.4
k8s.gcr.io/kube-controller-manager:v1.19.4
k8s.gcr.io/kube-scheduler:v1.19.4
k8s.gcr.io/kube-proxy:v1.19.4
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

> 使用 `kubeadm config images pull` 命令拉取上述镜像。 一般，国内无法直接下载 k8s.gcr.io 的镜像，故可使用下面这个脚本下载k8s所需要的镜像

```shell
$ cd ~ && touch pullk8s.sh # 创建脚本文件
$ vim pullk8s.sh		# 编辑脚本
```

将以下内容复制到脚本中

```shell
for  i  in  `kubeadm config images list`;  do
    imageName=${i#k8s.gcr.io/}
    docker pull registry.aliyuncs.com/google_containers/$imageName
    docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.aliyuncs.com/google_containers/$imageName
done;
```

给脚本赋予权限，并执行脚本。

```sh
$ chmod +x pullk8s.sh #给脚本文件赋权限
$ sh pullk8s.sh #执行脚本
```

执行 `docker images` 命令查看需要的镜像是否都准备好了。

```sh
devad@devad-VirtualBox:~$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.19.4             635b36f4d89f        3 weeks ago         118MB
k8s.gcr.io/kube-apiserver            v1.19.4             b15c6247777d        3 weeks ago         119MB
k8s.gcr.io/kube-controller-manager   v1.19.4             4830ab618586        3 weeks ago         111MB
k8s.gcr.io/kube-scheduler            v1.19.4             14cd22f7abe7        3 weeks ago         45.7MB
k8s.gcr.io/etcd                      3.4.13-0            0369cf4303ff        3 months ago        253MB
k8s.gcr.io/coredns                   1.7.0               bfe3a36ebd25        5 months ago        45.2MB
k8s.gcr.io/pause                     3.2                 80d28bedfe5d        9 months ago        683kB
```

#### 初始化集群

 ```sh
root@devad-VirtualBox:~# kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.131.131 --ignore-preflight-errors=Swap

W1207 09:47:55.650646  375338 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.4
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [devad-virtualbox kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.101]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [devad-virtualbox localhost] and IPs [192.168.56.101 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [devad-virtualbox localhost] and IPs [192.168.56.101 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.001717 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node devad-virtualbox as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node devad-virtualbox as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: dpn9w0.imcy8crk9nefsi99
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.101:6443 --token 3us197.65jugrjnjszfg5ht 
    --discovery-token-ca-cert-hash sha256:e7161225b1d5ab90f6d69fb0b6775f8c92f674a7845758e64eda3cc3dd996b9b
 ```

**进一步配置kubectl**

   ```sh
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
   ```

**查看node节点**

```sh
$ systemctl status kubelet
```



#### 配置网络插件

   **下载flannel插件的yaml文件**

   ```sh
$ cd ~ && mkdir flannel && cd flannel
$ curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

   **启动**

   ```
[root@devad-cloud ~]# kubectl apply -f ~/flannel/kube-flannel.yml
   ```

   **查看**

   ```sh
devad@devad-VirtualBox:~/flannel$ kubectl get node
NAME               STATUS   ROLES    AGE   VERSION
devad-virtualbox   Ready    master   95m   v1.19.4

devad@devad-VirtualBox:~/flannel$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-nr6h4                    1/1     Running   0          96m
coredns-f9fd979d6-sfk4x                    1/1     Running   0          96m
etcd-devad-virtualbox                      1/1     Running   5          96m
kube-apiserver-devad-virtualbox            1/1     Running   5          96m
kube-controller-manager-devad-virtualbox   1/1     Running   5          96m
kube-flannel-ds-dg998                      1/1     Running   0          2m3s
kube-proxy-jh4sc                           1/1     Running   5          96m
kube-scheduler-devad-virtualbox            1/1     Running   5          96m

devad@devad-VirtualBox:~/flannel$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   97m

   ```

> 注意：只有网络插件也安装配置完成之后，node才能会显示为ready状态。

# 三、KubeEdge的安装与配置

## cloud端配置

cloud端负责编译KubeEdge的相关组件与运行cloudcore。

#### 下载golang

```
[root@devad-cloud ~]# wget https://golang.google.cn/dl/go1.15.6.linux-amd64.tar.gz
[root@devad-cloud ~]# tar -zxvf go1.15.6.linux-amd64.tar.gz -C /usr/local
```

#### 配置golang环境

```sh
$ vim /etc/profile
# 文件末尾添加：golang和kubeedge的环境变量

# golang env
export GOROOT=/usr/local/go
export GOPATH=/data/gopath
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

# kubeedge env
export PATH=$PATH:/data/gopath/src/github.com/kubeedge/kubeedge/_output/local/bin


source /etc/profile  #使环境变量配置生效
mkdir -p /data/gopath && cd /data/gopath
$ mkdir -p src pkg bin
```

#### 下载KubeEdge源码

```
$ git clone https://github.com/kubeedge/kubeedge $GOPATH/src/github.com/kubeedge/kubeedge
```

**检测 gcc 版本**

```sh
$ gcc --version
```

**如果没有安装 gcc，则自行安装**

```sh
$ sudo apt update
$ sudo apt install build-essential
```

**编译kubeadm**

```sh
$ cd $GOPATH/src/github.com/kubeedge/kubeedge
$ make all WHAT=keadm

说明：编译后的二进制文件在./_output/local/bin下，单独编译cloudcore与edgecore的方式如下：
$ make all WHAT=cloudcore && make all WHAT=edgecore
```

#### 创建cloud节点

使用`keadm`进行快速部署，由于安装过程需要访问github相关资源，因此选择手动提前下载所需文件：

> g.ioiox.com 为github资源镜像加速，若失效则需自己替换可用镜像
>
> 版本号根据当前kubeedge版本自行替换

```bash
mkdir /etc/kubeedge
cd  /etc/kubeedge && wget https://g.ioiox.com/https://github.com/kubeedge/kubeedge/releases/download/v1.5.0/kubeedge-v1.5.0-linux-amd64.tar.gz
```

同时可能需要添加 raw.githubusercontent.com 的DNS解析：

```bash
151.101.108.133 raw.githubusercontent.com
```

创建cloud节点：

> IP替换为本机IP

```bash
$ keadm init --advertise-address="IP"
# keadm init --advertise-address="192.168.56.101"
```

查看日志：

```bash
# tail -f /var/log/kubeedge/cloudcore.log
```

如果cloudcore正常运行的话，10002端口会被占用，通过命令查看当前10002端口是否被占用

```bash
# netstat -tunlp | grep 10000
```

![image-20201206173301281](C:\Users\cossj\AppData\Roaming\Typora\typora-user-images\image-20201206173301281.png)

## 创建edge节点

在cloud端获取token：

```sh
root@devad-VirtualBox:~# keadm gettoken
ce1a34b2ee25e76a1630c4294912110654fb51dab2d90b44ab93aa66d6641e1a.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MDc5OTg5MjR9.LFu-11csM59ACEsX2ahTgfN3Z0cuCB-jcKICHiS-kq4
```

将cloud端的`./_output/local/bin`二进制文件拷贝至edge端并加入PATH

```bash
# 复制clod端的文件到edge端
$ sudo scp -r devad@192.168.137.131:/data/gopath/src/github.com/kubeedge/kubeedge/_output/ /data/gopath/src/github.com/kubeedge/kubeedge/_output/

# 文件末尾添加：kubeedge的环境变量
$ vim /etc/profile

# kubeedge env
export PATH=$PATH:/data/gopath/src/github.com/kubeedge/kubeedge/_output/local/bin
 
```

随后创建edge节点：

> CLOUD_IP 为前文中的cloud端暴露的IP
>
> TOKEN为获取到的token

```bash
keadm join --cloudcore-ipport=CLOUD_IP:10000 --token=TOKEN

keadm join --cloudcore-ipport=192.168.137.131:10000 --token=2185e19872e16bc2521e0b98c927f9cab5972c4f5f21e207f6883d069e223091.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MDkyMjg3Nzd9.U8B8lApOBNjZ7YoWQVGIHmHpDLAskdZglnincKy8DLg
```

> 国内网络有可能会失败，手动在 `/etc/kubeedge/`目录下创建edgecore.service文件。

```bash
[Unit]
Description=edgecore.service

[Service]
Type=simple
ExecStart=/etc/kubeedge/edgecore
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

![image-20201214102543273](C:\Users\cossj\AppData\Roaming\Typora\typora-user-images\image-20201214102543273.png)

在cloud查看日志：

```bash
tail -f /var/log/kubeedge/cloudcore.log
```

在edge端查看日志:

```bash
journalctl -u edgecore.service -b
journalctl -f -u kubelet
```

## 验证

边缘端在启动edgecore后，会与云端的cloudcore进行通信，K8s进而会将边缘端作为一个node纳入K8s的管控。

```bash
root@devad-cloud:/home/devad# kubectl get node
NAME          STATUS   ROLES        AGE    VERSION
devad-cloud   Ready    master       30h    v1.19.4
devad-edge1   Ready    agent,edge   2m1s   v1.19.3-kubeedge-v1.5.0
devad-edge2   Ready    agent,edge   2m5s   v1.19.3-kubeedge-v1.5.0

root@devad-cloud:/home/devad# kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-p5bzn               1/1     Running   16         30h
coredns-f9fd979d6-zjqbq               1/1     Running   16         30h
etcd-devad-cloud                      1/1     Running   18         30h
kube-apiserver-devad-cloud            1/1     Running   19         30h
kube-controller-manager-devad-cloud   1/1     Running   18         30h
kube-flannel-ds-56rpc                 0/1     Error     4          2m48s
kube-flannel-ds-qxpld                 0/1     Error     3          2m52s
kube-flannel-ds-sjp2k                 1/1     Running   44         30h
kube-proxy-9cct8                      1/1     Running   18         30h
kube-proxy-jn4d6                      1/1     Running   0          2m48s
kube-proxy-ll5j5                      1/1     Running   0          2m52s
kube-scheduler-devad-cloud            1/1     Running   18         30h

说明：如果在K8s集群中配置过flannel网络插件，这里由于edge节点没有部署kubelet，
所以调度到edge节点上的flannel pod会创建失败。这不影响KubeEdge的使用，可以先忽略这个问题。
```

> **如果edgecore节点状态显示为NotReady，可以通过命令 `netstat -tunlp | grep 10000` 排查下cloudcore端的cloudcore是否正常运行。**

如果你不能重启  `edgecore`，检查是否是因为  `kube-proxy`，然后关闭它。`Kubeedge `默认拒绝了它

**注意**： 重要的是避免在  `edgenode` 上部署 **kube-proxy**。有两种方法可以解决这个问题:

1. 通过调用 ：`kubectl edit daemonsets.apps -n kube-system kube-proxy`

```
affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role.kubernetes.io/edge
                operator: DoesNotExist
```

1. 如果你仍然想运行 **kube-proxy**，那么请求 edgecore 不要通过在 edgecore.service 中添加 env 变量来检查环境:

   ```
   sudo vi /etc/kubeedge/edgecore.service
   ```

   - 将以下行添加到**edgecore.service**文件中：

   ```
   Environment="CHECK_EDGECORE_ENVIRONMENT=false"
   ```

   - 最后的文件应该是这样的:

   ```bash
   Description=edgecore.service
   
   [Service]
   Type=simple
   ExecStart=/root/cmd/ke/edgecore --logtostderr=false --log-file=/root/cmd/ke/edgecore.log
   Environment="CHECK_EDGECORE_ENVIRONMENT=false"
   
   [Install]
   WantedBy=multi-user.target
   ```

重新启动所有的**cloudcore**和**edgecore**

```bash
root@devad-cloud:~# pkill cloudcore
root@devad-cloud:~# nohup cloudcore > /var/log/kubeedge/cloudcore.log 2>&1 & #重启cloudcore

root@devad-edge1:~# systemctl restart edgecore #重启edgecore
```

## 删除节点

使用适当的凭证与控制平面节点通信，运行：

```bash
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
```

在删除节点之前，请重置 `kubeadm` 安装的状态：

```bash
kubeadm reset
```

重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

如果要重置 IPVS 表，则必须运行以下命令：

```bash
ipvsadm -C
```

现在删除节点：

```bash
kubectl delete node <node name>
```