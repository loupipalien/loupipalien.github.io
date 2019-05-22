---
layout: post
title: "Argo 入门"
date: "2019-04-26"
description: "Argo 入门"
tag: [argo, kubernates]
---

### Argo 入门
前提要求
- 安装 Kubernates 1.9 或以上 (可以安装 Minikube, 本文操作即在 Minikube 上完成)
- 安装 kubectl
- 有一个 kubeconfig 配置文件 (默认路径在 `~/.kube/config`)

#### 下载 Argo
在 Linux 上, 可以使用以下语句
```
$ curl -sSL -o  https://github.com/argoproj/argo/releases/download/v2.2.1/argo-linux-amd64
$ sudo mv argo /usr/local/bin/argo
$ sudo chmod +x /usr/local/bin/argo
```

#### 安装 Controller 和 UI
在 k8s 上安装其服务
```
$ kubectl create ns argo  # 在 ks8 上创建 argo 的命令空间
$ kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.2.1/manifests/install.yaml
```

#### 配置服务账号以运行工作流
要运行指南中的所有示例, "默认" 服务帐户太受限制, 无法支持构件, 输出, 访问机密等功能... 出于演示目的, 请运行以下命令以授予 `default` 命名空间中的 `default` 服务帐户管理员的权限
```
$ kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default
```
对于工作流需要运行的最小权限集, 见 [Workflow RBAC](https://argoproj.github.io/docs/argo/docs/workflow-rbac.html); 还可以使用其他服务帐户提交工作流运行

#### 运行简单的示例工作流
```
$ argo submit --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/hello-world.yaml
$ argo submit --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/coinflip.yaml
$ argo submit --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/loops-maps.yaml
$ argo list
$ argo get xxx-workflow-name-xxx
$ argo logs xxx-pod-name-xxx  # 从上一句命令中输出获得
```
你也可以直接使用 kubectl 来创建工作流; 然而, Argo CLI 提供了 kubectl 没有的额外特性, 例如 YAML 校验, 工作流可视化, 参数传递, 重试, 重提交, 暂停, 恢复等
```
$ kubectl create -f https://raw.githubusercontent.com/argoproj/argo/master/examples/hello-world.yaml
$ kubectl get wf
$ kubectl get wf hello-world-xxx
$ kubectl get po --selector=workflows.argoproj.io/workflow=hello-world-xxx --show-all
$ kubectl logs hello-world-yyy -c main
```
其他示例见 [这里](https://github.com/argoproj/argo/blob/master/examples/README.md)

#### 安装一个构件仓库
Argo 支持 S3 (AWS, GCS, Minio) 以及 Artifact 作为构件仓库, 为了尽可能的方便本教程使用 Minio; 如何配置其他构件仓库的指南见 [这里](https://github.com/argoproj/argo/blob/master/ARTIFACT_REPO.md)  
Linux 可以先安装 [Snap](https://snapcraft.io/), 再使用 Snap 安装 [Helm](https://helm.sh), 再使用 Helm 安装 [Minio](https://min.io/) (这里需要 FQ)
```
$ helm init  # 初始化 helm 服务端和客户端
$ helm install stable/minio --name argo-artifacts --set service.type=LoadBalancer --set persistence.enabled=false
```
可以使用 `kubectl` 获取外部 IP 后, 使用浏览器登录 Minio UI (端口: 9000)
```
$ kubectl get service argo-artifacts -o wide
```
如果是 Minikube 的 Kubernates, 可以使用以下语句
```
$ minikube service --url argo-artifacts
```
在实际操作中发现 Minikube 上部署的 argo-artifacts-minio 的 service 的外部 IP 永远都是 pending 状态(因为类型为 LoadBalancer, 所以会有外部 IP), 但 minikube 中貌似不能支持 LoadBalancer 类型的服务分配外部 IP, 所以服务不能够成功启动, 可见 [issue#384](https://github.com/kubernetes/minikube/issues/384) 和这个 [question](https://stackoverflow.com/questions/44110876/kubernetes-service-external-ip-pending); [issue#220](https://github.com/kubernetes/kube-deploy/issues/220) 中 @DawidCh 提供了方法, 即手动将 externalIPs 设置到 service 的配置中, 更新配置后 service 会显示为启动成功 (但外部 IP 并不生效); 运行 `helm status argo-artifacts` 可查看运行状态;
```
$ export POD_NAME=$(kubectl get pods --namespace default -l "release=argo-artifacts" -o jsonpath="{.items[0].metadata.name}")
$ kubectl port-forward $POD_NAME 9000 --namespace default
```
再访问 `127.0.0.1:9000` 便可打开 Minio UI
>注意: 当使用 Helm 安装 minio 时, 将使用以下硬编码的默认凭据, 你可以使用其登录 UI 界面:
AccessKey: AKIAIOSFODNN7EXAMPLE  
SecretKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

接着, 在 Minio UI 中创建一个名为 `my-bucket` 的 bucket

#### 使用 Minio 构件库重新配置工作流控制器
编辑 workflow-controller 配置映射, 并引用由 helm 安装的 argo-artifacts 的服务名和密钥
```
$ kubectl edit cm -n argo workflow-controller-configmap
...
data:
  config: |
    artifactRepository:
      s3:
        bucket: my-bucket
        endpoint: argo-artifacts.default:9000
        insecure: true
        # accessKeySecret and secretKeySecret are secret selectors.
        # It references the k8s secret named 'argo-artifacts'
        # which was created during the minio helm install. The keys,
        # 'accesskey' and 'secretkey', inside that secret are where the
        # actual minio credentials are stored.
        accessKeySecret:
          name: argo-artifacts
          key: accesskey
        secretKeySecret:
          name: argo-artifacts
          key: secretkey
```
>注意: 当你运行工作流时, Minio 的密钥将从命名空间中搜索; 如果 Minio 被安装在其他的命名空间, 那么你需要在你运行工作流的命名空间中拷贝这个密钥

#### 使用构件运行一个工作流
默认的 Argo UI 不会带着外部 IP 暴露, 可以使用以下其中一个方法来访问 UI
```
argo submit https://raw.githubusercontent.com/argoproj/argo/master/examples/artifact-passing.yaml
```
实际操作中遇到了以下两个报错
- failed to save outputs: secrets "argo-artifacts" not found
这是因为在提交的命令空间中找不到名为 "argo-artifacts" 的密钥, 使用上面的 AccessKey 和 SecretKey 的值创建即可
- failed to save outputs: Get http://argo-artifacts.default:9000/my-bucket/?location=: dial tcp: lookup argo-artifacts.default on 10.96.0.10:53: no such host
这表示解析不了 argo-artifacts.default 此主机名, 这里将之前 workflow-controller-configmap 中 endpoint 的 argo-artifacts.default 替换为对应 pod 的 ip

以上报错的原因可能是, 安装构件仓库时 --name argo-artifacts 未生效

#### 访问 Argo UI
##### 方法一: kubeclt port-forward
```
$ kubectl -n argo port-forward deployment/argo-ui 8001:8001
```
然后访问: http://127.0.0.1:8001

##### 方法二: kubectl 代理
```
$ kubectl proxy
```
然后访问: http://127.0.0.1:8001/api/v1/namespaces/argo/services/argo-ui/proxy/
>注意: 使用这种方法, 构件下载和 webconsole 是不支持的

##### 方法三: 暴露一个 LoadBalancer
更新 argo-ui 服务的类型为 `LoadBalancer`
```
$ kubectl patch svc argo-ui -n argo -p '{"spec": {"type": "LoadBalancer"}}'
```
然后等待外部 IP 可用
```
$ kubectl get svc argo-ui -n argo
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
argo-ui   LoadBalancer   10.19.255.205   35.197.49.167   80:30999/TCP   1m
```
>注意: 在 Minikube 上, 在更新服务后你也不能获得外部 IP, 它将总是 `pending` (这坑为啥不早说...)

运行以下命令来确定 Argo UI URL
```
$ minikube service -n argo --url argo-ui
```

#####
>**参考:**
[Argo Getting Started](https://argoproj.github.io/docs/argo/demo.html)   
[Configuring Your Artifact Repository](https://github.com/argoproj/argo/blob/master/ARTIFACT_REPO.md)  
[Installing snap on CentOS](https://docs.snapcraft.io/installing-snap-on-centos/10020)  
[Installing Helm](https://helm.sh/docs/using_helm/#installing-helm)
