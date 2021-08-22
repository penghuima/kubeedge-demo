# 基于KubeEdge框架的demo开发

### 仓库介绍

本仓库主要记录项目的开发过程，仓库目录有

- [基于KubeEdge框架的demo](./samples) 
- [开发demo过程中所有文档记录](./doc)
- [实验结果截图](./Screenshot of running result)
- [kubeedge 环境搭建教程](./Kubeedge installation tutorial)

下面详细介绍 [Temperature demo on Raspberry PI](./samples/temperature-demo) 部署实例

公有云环境访问链接  [139.198.19.139:30813/](http://139.198.19.139:30813/)

### 项目概述

#### 项目目标

基于 Ubuntu 虚拟机、树莓派以及温度传感器实现在云端对边缘侧环境温度的实时采集。

#### 项目背景

温度监测在不同的应用场景下都具备广泛的需求，本项目基于华为开源的边缘计算框架 KubeEdge 来实现对边缘设备的监测控制，验证功能的可行性，在本实验基础上可以对集群规模进行扩展，实现大规模环境温度实时监测，并根据边缘侧负载、网络情况进行任务的动态调度，在实现云边协同的同时增强边缘节点的离线自治能力，满足多场景应用需求。

#### 项目详情

实验使用kubeadm搭建集群并初始化主节点，并配置golang环境可实现灵活的业务定制，使用 keadm 工具分别在云端和树莓派节点部署 cloudcore 服务和 edgecore 服务，完成云边协同架构的搭建，在云端生成设备实例，边缘侧在业务代码基础上构建镜像，云端通过 kube-scheduler 服务结合 nodeSelector 的标签选择机制将温度监测 pod 调度至树莓派节点，树莓派在本地构建的镜像基础上拉起容器，在云端可实时显示温度状态。

### 本地环境

#### 软件包版本

- Kernel version: 5.11.0-25-generic
- Kubelet version: v1.19.3
- KubeEdge version： v1.7.2
- Golang version: go1.14.4 linux/amd64 for Ubuntu 21.04，go1.14.4 linux/arm64 for Raspberry PI
- Docker version: v20.10.7

#### 硬件型号与配置

Raspberry PI 4B

- CPU(s): 4
- Memory: 8G
- Disk: 128G
- Vendor ID: ARM
- Model name: Cortex-A72
- CPU max MHz: 1500.0000
- CPU min MHz: 600.0000

温度传感器: DHT11

### 工具依赖

实验过程中需安装 docker 环境并配置 golang 环境变量，当前 KubeEdge 支持的 kubernets 最高版本是 v1.19.X以下，建议保持各节点软件包版本一致，本次实验 Vmware 虚拟机使用的是 amd64 位架构，树莓派为 arm64 位架构，需使用对应架构版本的软件。

#### 安装 docker

使用命令来安装 docker

```sh
$ sudo apt install docker.io
```

配置docker 镜像加速器

编辑配置文件

```sh
$ sudo vim /etc/docker/daemon.json
```

文件内容如下

```sh
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com",
    "https://quay-mirror.qiniu.com",
    "https://dh7kdsrx.mirror.aliyuncs.com"
  ],
}
```

内容保存后重启docker

```sh
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

#### 安装Finalshell 配置ssh服务

在各节点上设置免密登录

修改配置文件内容

```shell
$ sudo vim /etc/ssh/sshd_config
```

将配置文件内容 PermitRootLogin 、PasswordAuthentication 设置为 yes

```shell
PermitRootLogin  yes
PasswordAuthentication  yes
```

Finalshell 配置

![image-20210822163202028](https://cdn.jsdelivr.net/gh/penghuima/ImageBed@master/img/blog_file/PicGo-Github-ImgBedimage-20210822163202028.png)

类似的操作分别配置节点 master 、node1、node2、raspberrypi 的 SSH连接，后续所有节点的操作都将在Finalshell中实现。

#### 配置Goland环境

下载golang

```sh
[root@master ~]# wget https://golang.google.cn/dl/go1.14.4.linux-amd64.tar.gz
[root@master ~]# tar -zxvf go1.14.4.linux-amd64.tar.gz -C /usr/local
```

配置golang环境

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

### 集群搭建

#### 安装 kubeadm、kubelet 和 kubectl

为K8s配置国内阿里源

添加 public key

```sh
$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add 
```

添加源

```sh
$ sudo cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ sudo apt-get update
```

安装kubeadm、kubelet、kubectl

```sh
# 安装指定版本,目前kubesphere支持k8s 1.15.x, 1.16.x, 1.17.x, or 1.18.x版本，不指定版本会默认安装20版本
sudo apt-get install -y kubeadm=1.19.3-00 kubelet=1.19.3-00 kubectl=1.19.3-00
sudo apt-mark hold kubelet kubeadm kubectl
```

用命令查看版本当前kubeadm对应的k8s组件镜像版本

```sh
$ kubeadm config images list
```

#### 拉取镜像

使用 kubeadm config images pull 命令拉取上述镜像。 一般，国内无法直接下载 k8s.gcr.io 的镜像，故可使用下面这个脚本下载k8s所需要的镜像

```sh
$ cd ~ && touch pullk8s.sh 
$ vim pullk8s.sh		
```

内容如下

```sh
for  i  in  `kubeadm config images list`;  do
    imageName=${i#k8s.gcr.io/}
    docker pull registry.aliyuncs.com/google_containers/$imageName
    docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.aliyuncs.com/google_containers/$imageName
done;
```

给脚本赋予权限，并执行脚本

```sh
$ sudo chmod +x pullk8s.sh 
$ sh pullk8s.sh 
```

查看所需的镜像是否安装完毕

```sh
$ docker images
```

#### 初始化集群

使用命令初始化集群

```sh
#注意第二个ip要使用自己master主机的ip 192.168.43.110
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.43.110 --ignore-preflight-errors=Swap
```

初始完毕后会给出如下提示后续信息

```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.43.110:6443 --token gcvj8c.ajd2osq1x2agp4it \
    --discovery-token-ca-cert-hash sha256:b053d393444d079d24588993d22605298b4475339ac32219fc803d5ae0da1536
```

#### 配置kubelctl

使用提示信息首先来配置kubectl的环境path

```sh
$ rm -rf $HOME/.kube
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看kubelet状态

```sh
# systemctl status kubelet
```

**查看node节点**

```
$ systemctl status kubelet
```

#### 配置网络插件

下载flannel插件的yaml文件

```sh
$ cd ~ && mkdir flannel && cd flannel
$ curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

启动

```sh
[root@master ~]# kubectl apply -f ~/flannel/kube-flannel.yml
```

> 注意：只有网络插件也安装配置完成之后，node才能会显示为ready状态。

### KubeEdge的安装与配置

#### cloud端配置

下载keadm工具

```sh
wget https://github.com/kubeedge/kubeedge/releases/download/v1.7.2/keadm-v1.7.2-linux-amd64.tar.gz
```

解压然后进入解压后的目录,运行

```sh
./keadm init
```

CloudCore成功启动，使用以下命令获取token

```sh
./keadm gettoken
```

#### edge端配置

和 cloud 端配置类似，下载 keadm 工具，解压进入解压后的目录，运行

```sh
./keadm join --cloudcore-ipport=192.168.43.110:10000 --token=
```

加入成功后会自动安装edgecore服务，提示edgecore is running说明安装成功

此时在云端节点可以查看集群节点状态

![image-20210822170147973](https://cdn.jsdelivr.net/gh/penghuima/ImageBed@master/img/blog_file/PicGo-Github-ImgBedimage-20210822170147973.png)

### 业务部署

#### 硬件准备

- 树莓派外接显示器、鼠标和键盘
- 温度传感器和杜邦线
- 读卡器、SD卡（烧录树莓派操作系统）

![image-20210822182759512](https://cdn.jsdelivr.net/gh/penghuima/ImageBed@master/img/blog_file/PicGo-Github-ImgBedimage-20210822182759512.png)

#### 系统烧录

在树莓派官网根据硬件型号下载专用镜像

使用官方提供的Raspberry Pi Imager工具烧录镜像至树莓派SD卡，工具可在[官网](https://www.raspberrypi.org/blog/raspberry-pi-imager-imaging-utility/)下载

#### 树莓派添加至集群

下载 keadm 工具，解压进入解压后的目录，运行

```sh
./keadm join --cloudcore-ipport=192.168.43.110:10000 --token=
```

加入成功后会自动安装edgecore服务，提示edgecore is running说明安装成功

通过以下命令查看是否安装成功

```sh
kubectl get node
```

#### 创建设备实例

根据yaml文件部署 CRD 自定义资源

```sh
cd $GOPATH/src/github.com/kubeedge/examples/temperature-demo/crds
kubectl apply -f model.yaml
sed -i "s#edge-node#raspberrypi#g" instance.yaml
kubectl apply -f instance.yaml
```

修改后的 instance.yaml 文件为

```yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: Device
metadata:
  name: temperature
  labels:
    description: 'temperature'
    manufacturer: 'test'
spec:
  deviceModelRef:
    name: temperature-model
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: ''
            operator: In
            values:
              - raspberrypi
status:
  twins:
    - propertyName: temperature-status
      desired:
        metadata:
          type: string
        value: ''
```

#### 构建本地业务镜像

在树莓派节点执行一下操作

```sh
cd $GOPATH/src/github.com/kubeedge/examples/temperature-demo
docker build -t kubeedge/temperature-mapper-demo:arm64 .
```

#### 部署业务容器

在云端节点部署业务容器，云端通过kube-scheduler服务结合nodeSelector的标签选择机制将温度监测pod调度至树莓派节点。

```sh
cd $GOPATH/src/github.com/kubeedge/examples/temperature-demo/
kubectl create -f deployment.yaml
```

#### 获取传感器数据

```sh
kubectl get device temperature -w -o go-template --template='{{ range .status.twins }} {{.reported.value}} {{end}}'
```

实验结果如下：

![image-20210822184135038](https://cdn.jsdelivr.net/gh/penghuima/ImageBed@master/img/blog_file/PicGo-Github-ImgBedimage-20210822184135038.png)

### Q & A

问题1:云端创建pod后无法将pod调度到指定的树莓派节点，查看日志提示集群中没有符合nodeSelector条件的节点。

解决办法：查看节点标签

kubectl get node --show-labels

发现树莓派节点标签格式为kubernetes.io/hostname=raspberrypi，deployment.yaml文件中nodeSelector条件为name:raspberryp，两者格式不统一，修改yaml文件后成功将pod调度到边缘节点。修改后的文件为：

```yaml
spec:
      hostNetwork: true
      nodeSelector:
        kubernetes.io/hostname=raspberrypi
      containers:
      - name: temperature
        image: kubeedge/temperature-mapper-demo:arm32
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
```

问题2: pod成功调度到边缘节点，但容器无法启动，一直处于Error状态。

解决办法：在镜像构建环节最初是在云端节点构建Docker镜像，通过scp方式传到边缘节点并加载镜像，从云端构建的镜像传到边缘端后无法成功创建容器，pod处于Error状态。

分析后发现问题出在云、边节点架构不同，云端Ubuntu节点为amd64位架构，边缘raspberrypi节点为arm64架构，所以导致镜像在异构节点上无法创建容器。直接在raspberrypi节点创建镜像，并保证镜像名称和tag与deployment.yaml文件中一致，否则创建的pod将无法识别镜像，重新创建镜像后成功拉起容器，问题解决。

### 参考资料

> - https://blog.51cto.com/u_10006690/2727626
> - https://learnku.com/articles/29209
> - https://www.freesion.com/article/11691054943/
> - https://blog.csdn.net/weixin_43168190/article/details/107223600
> - https://blog.csdn.net/weixin_43168190/article/details/107227550
> - https://www.yuque.com/sunxiaping/yg511q/gqxqgg
> - https://www.it610.com/article/1281557696918601728.htm
> - https://cloud.tencent.com/developer/article/1661687
> - https://github.com/kubeedge/examples/tree/master/temperature-demo
> - https://blog.csdn.net/Obese_Tiger/article/details/104802926?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-18.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-18.control
> - https://mp.weixin.qq.com/s?search_click_id=8336705840306415996-1628507703106-740319&sub=&__biz=MzIzNzU5NTYzMA==&mid=2247490514&idx=1&sn=66e8500384a5c2a36c1de23ea3b32f93&chksm=e8c76553dfb0ec458e1fd89dc5b506e2f47277553b564537de017b88103e793a017b41e36ccf&scene=3&subscene=10000&clicktime=1628507703&enterid=1628507703&ascene=0&devicetype=android-29&version=2800093b&nettype=WIFI&abtest_cookie=AAACAA%3D%3D&lang=zh_CN&exportkey=ASqKN4QWA8wMMvUa%2B0bx1f0%3D&pass_ticket=1EpYoHipQ3rxuVT0lY9l4VSM22UXsT4OAvXxoWBm%2B%2Bdlewg%2F%2BalUT%2B7QT1tXKv8S&wx_header=1