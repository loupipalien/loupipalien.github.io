---
layout: post
title: "ArgoCD 入门"
date: "2019-04-28"
description: "ArgoCD 入门"
tag: [argo, kubernates]
---

### 入门
要求
- 安装了 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 命令行工具
- 有一个 [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) 文件 (默认位置是 `~/.kube/config`)

#### 安装 Argo CD
```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
这将会创建一个新的命名空间 `argocd`, Argo CD 服务和应用资源都会部署在这里  
如果再 GKE 上, 你还需要授权你的账号能够创建一个新集群角色

#### 下载 Argo CD CLI
从 `https://github.com/argoproj/argo-cd/releases/latest` 下载最新版本 Argo CD
```
$ curl -LO https://github.com/argoproj/argo-cd/releases/download/v0.12.2/argocd-linux-amd64
$ chmod +x argocd-linux-amd64
$ sudo mv argocd-linux-amd64 /usr/local/bin/argocd
```
Mac 可使用 Homebrew 安装
```
$ brew tap argoproj/tap
$ brew install argoproj/tap/argocd
```

#### 访问 Argo CD API 服务器
默认的, Argo CD API 服务器将不会暴露一个外部 IP; 为了访问 API 服务, 可以选择以下其中一种技术来暴露 Argo CD API 服务

##### LoadBlancer 服务类型
修改 argocd-server 的服务类型为 `LoadBalancer` (此方案在 Minikube 上也不能暴露一个外部 IP)
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
##### Ingress
如何为 Argo CD 配置 ingress 见 [ingress 文档](https://argoproj.github.io/argo-cd/operator-manual/ingress/)
##### 端口转发
kubectl port-forwarding 可以用于连接没有暴露服务的 API server
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
可以使用 `localhost:8080` 来访问 API 服务器 (如果遇到浏览器报安全问题, 将其添加为例外网址即可, 可见这个  [issue#828](https://github.com/argoproj/argo-cd/issues/828))

#### 使用 CLI 登录
使用 `admin` 用户登录; 初始化密码是自动生成的 Argo CD API 服务器 pod 名; 可以使用以下命令获得
```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
使用以上密码登录到 Argo CD 的 IP 或 hostname
```
argocd login <ARGOCD_SERVER>
```
使用以下命令修改密码
```
argocd account update-password
```

#### 注册一个集群来部署 Apps (可选)
TODO

#### 从一个 Git 仓库创建一个应用
示例仓库包含了一个 guestbook 应用来展示 Argo CD 是如何工作的, 仓库地址为 `https://github.com/argoproj/argocd-example-apps.git`  
##### 从 CLI 创建 Apps
```
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```
##### 从 UI 创建 Apps
从浏览器打开 Argo CD UI, 使用第四步中的用户密码登录; 连接到 `https://github.com/argoproj/argocd-example-apps.git` 仓库
![image](https://argoproj.github.io/argo-cd/assets/connect_repo.png)
在连接仓库后, 选择 guestbook 应用来创建
![image](https://argoproj.github.io/argo-cd/assets/select_app.png)
![image](https://argoproj.github.io/argo-cd/assets/create_app.png)

#### 同步 (部署) 应用
一旦 guestbook 应用被创建, 你可以使用以下命令看到它的状态
```
$ argocd app get guestbook
```
应用的状态初始是 `OutOfSync` 的, 因为应用还没有被部署, 并且没有 Kubernetes 资源被创建; 运行以下命令同步 (部署) 应用
```
$ argocd app sync guestbook
```
这个命令会从仓库中获取 manifests 并执行 `kubectla apply` 这个 manifests; guestbook 应用现在已经运行, 可以看到他的资源组件, 日志, 事件, 健康状态等
![image](https://argoproj.github.io/argo-cd/assets/guestbook-app.png)
![image](https://argoproj.github.io/argo-cd/assets/guestbook-tree.png)

>**参考:**
[Getting Started](https://argoproj.github.io/argo-cd/getting_started)
