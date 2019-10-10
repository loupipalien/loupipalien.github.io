---
layout: post
title: "安装 Minikube"
date: "2019-04-22"
description: "安装 Minikube"
tag: [kubernates]
---

### 安装 Minikube

#### 开始之前
在电脑 BIOS 中必须开启 VT-x 或 AMD-v 虚拟化; 在 Linux 上可以运行以下命令检查, 确认输出是非空的
```
egrep --color 'vmx|svm' /proc/cpuinfo
```
还可使用 `lscpu | grep Virtualization` 查看
>如果使用的是 VMware, 同时确保在虚拟机 ->设置 -> 处理器 -> 虚拟化引擎中选择了对应项

#### 安装 Hypervisor
接下来需要安装一个 Hypervisor; [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) 中有各个操作系统支持的 Hypervisor 列表; Minikube 默认使用 virtualbox, 可从其 [官网](https://www.virtualbox.org) 下载 [Yum 仓库源](https://download.virtualbox.org/virtualbox/rpm/el/virtualbox.repo) 安装; 其他 Driver 安装见 [Driver plugin installation](https://github.com/kubernetes/minikube/blob/master/docs/drivers.md#kvm2-driver)  
这个组件是一个虚拟化的引导, 因为 Minikube 的本质是在一个 VM 里运行 Kubernetes 的各种 Docker 镜像; 可以了解下在 [VirtualBox 中安装 VM](https://www.jianshu.com/p/18207167b1e7) 来理解这概念

#### 安装 kubectl
具体操作见 [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/); 国内可以使用 aliyun 的镜像库
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 安装 Minikube
具体操作见 [Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
> `storage.googleapis.com` 上的各种资源难下载, 没找到可用的国内镜像, FQ 才一点点下载完成

#### 启动前清理一切
如果之前没有安装过 minikube, 运行以下语句(Minikube 默认使用 virtualbox 驱动) `minikube start` (建议加上 `--alsologtostderr -v=9` 参数, 可以看到详细日志); 如果此命令返回 `machine does not exist`, 那么需要先清理掉之前的配置文件 `rm -rf ~/.minikube`; 如果是其他报错执行 `minikube delete` 再执行 `minikube start` 即可, 不建议删除 `~/.minikube`, 因为 `~/.minikube/cache/images/` 里缓存了一些好不容易才下载的 docker 镜像

#### minikube start 时遇到的问题
- The vboxdrv kernel module is not loaded
根据提示解决即可, 安装 `gcc make perl` 和 `kernel-devel`, 其中 `kernel-devel` 必须要报错提示中的版本(Yum 源中没有指定版本, 可下载 RMP 本地安装), 否则还是会报错; 最后看到 `vboxdrv.sh: Building VirtualBox kernel modules` 即成功
- 可以 FQ, 但从 storage.googleapis.com 上的包安装中途失败
例如 `minikube-darwin-amd64, minikube-v1.0.0.iso` 等, 可以先下载到本地再安装 (minikube start 有 --iso-url 参数可用)
- 报错 `The machine 'minikube' is already locked for a session (or being unlocked)`
解决方法见 [这里](http://uzigood.blogspot.com/2016/12/virtualbox-is-already-locked-by-session.html)
- 一堆 `open /root/.docker/config.json: no suchfile or directory` 日志
无关紧要, 如想解决见 [issue#4007](https://github.com/kubernetes/minikube/issues/4007)
- 日志报错 `kube-apiserver: ... TLS handshake timeout`
这通常是由于开了本机的全局代理, 关闭代理解决, 类似情况见 [这里](http://leonlibraries.github.io/2017/06/15/Kubeadm%E6%90%AD%E5%BB%BAKubernetes%E9%9B%86%E7%BE%A4/)
- 启动卡住或创建镜像失败
这往往是一些镜像包下载不下来, 可以在本机开启代理, 也可利用 `--docker-env` 参数设置 VM 内的 docker 代理, 具体见 [这里](https://xkcoding.com/2019/01/14/solve-mac-install-minikube-problem.html), 或者使用 `--registry-mirror` 参数配置镜像库
- 机器配置可能有一些要求
本文使用的 CentOS7 VM 配置: CPU: 2C, Memory: 4G, Disk: 20G

#### 打开 kubernates 的面板
`minikube dashboard` 命令会使用默认浏览器打开面板, 如果使用的非图形化的机器, 可以使用 `kubectl proxy` 添加外部代理 (注意关闭防火墙) 访问, 具体操作见 [这里](https://blog.51cto.com/tryingstuff/2309205)

#### 总结
- 因为我们伟大的祖国繁荣富强, 导致无法下载 k8s 镜像 :joy:
- 有报错先仔细看日志, 如果日志里没提示或线索再 google

>**参考:**
[Install Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)    
[輕鬆解決 VirtualBox is already locked by a session (or being locked or unlocked)](http://uzigood.blogspot.com/2016/12/virtualbox-is-already-locked-by-session.html)   
[Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)  
[centos7安装minikube](http://lc161616.cnblogs.com/p/9192329.html)
[mac上安装minikube及排错记](https://fatfatson.github.io/2018/07/23/mac%E4%B8%8A%E5%AE%89%E8%A3%85mimikube/)  
[使用minikube创建K8S单机环境-填坑指南](https://blog.51cto.com/tryingstuff/2309205)  
[解决 Mac 安装最新版 minikube 出现的问题](https://xkcoding.com/2019/01/14/solve-mac-install-minikube-problem.html)  
[CentOS 7安装配置Shadowsocks客户端](https://www.zybuluo.com/ncepuwanghui/note/954160)
