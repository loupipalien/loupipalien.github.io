---
layout: post
title: "Argo 工作流模板示例"
date: "2019-04-28"
description: "Argo 工作流模板示例"
tag: [argo, kubernetes]
---

### Welcome!
TODO

#### Argo CLI
在本示例中遵循以下演练, 可以快速浏览最有用的 argo 命令行界面的命令
```
$ argo submit hello-world.yaml    # 提交一个工作流到 kubernetes
$ argo list                       # 列出当前工作流
$ argo get hello-world-xxx        # 获取指定工作流的信息
$ argo logs -w hello-world-xxx    # 获取一个工作流中所有步骤的日志
$ argo logs hello-world-xxx-yyy   # 获取一个工作流中指定步骤的日志
$ argo delete hello-world-xxx     # 删除工作流
```
你也可以直接使用 `kubectl` 来运行工作流, 但是 Argo CLI 提供了语法检查, 更好的输出, 以及更少的必要输入
```
$ kubectl create -f hello-world.yaml
$ kubectl get wf
$ kubectl get wf hello-world-xxx
$ kubectl get po --selector=workflows.argoproj.io/workflow=hello-world-xxx --show-all  # similar to argo
$ kubectl logs hello-world-xxx-yyy -c main
$ kubectl delete wf hello-world-xxx
```

#### Hello World!
让我们通过创建一个非常简单的工作流模板开始, 此摹本会使用从 DockerHub 拉取的 docker/whalesay 容器镜像输出 "hello world"; 你也可以直接从你的 shell 中运行一个简单的 docker 命令行
```
$ docker run docker/whalesay cowsay "hello world"
 _____________
< hello world >
 -------------
    \
     \
      \
                    ##        .
              ## ## ##       ==
           ## ## ## ##      ===
       /""""""""""""""""___/ ===
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
       \______ o          __/
        \    \        __/
          \____\______/


Hello from Docker!
This message shows that your installation appears to be working correctly.
```
以下, 我们在一个 kubernetes 集群上使用一个 Argo 工作流运行相同的容器; 请务必阅读注释, 因为它们提供了非常拥有的解释
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow                  # 新的 k8s 规范类型
metadata:
  generateName: hello-world-    # 工作流的名称
spec:
  entrypoint: whalesay          # 调用 whalesay 模板
  templates:
  - name: whalesay              # 模板的名称
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
      resources:                # 资源限制
        limits:
          memory: 32Mi
          cpu: 100m
```
Argo 添加了一种名为 `Workflow` 的新的 kubernetes 规范类型; 以上规范包含了一个名为 `whalesay` 的模板, 此模板会运行 `docker/whalesay` 容器被调用 `caysay "hello world"`; `whalesay` 模板是规范的 `entrypoint`; 入口点制定了初始化模板, 当工作流规范被 Kubernetes 中执行时调用这个模板; 当有多个模板被定义在 Kubernetes 工作流规范中时, 指定入口点是非常有用的

#### 参数
让我们来看一个带参数的稍微复杂的工作流规范
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
spec:
  # invoke the whalesay template with
  # "hello world" as the argument
  # to the message parameter
  entrypoint: whalesay
  arguments:
    parameters:
    - name: message
      value: hello world

  templates:
  - name: whalesay
    inputs:
      parameters:
      - name: message       # parameter declaration
    container:
      # run cowsay with that message input parameter as args
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```
这次, `whalesay` 模板使用了一个名为 `message` 的输入参数, 此参数将被传递作为 `cowsay` 命令的 `args`; 为了引用参数 (例如, `"{{inputs.parameters.message}}"`), 参数必须使用双引号包裹来转义 YAML 中的花括号  
argo CLI 提供了一个便捷的方式覆盖参数可用于调用入口点; 例如, 以下命令将绑定 `message` 参数为 "goodbye world" 以替换默认的 "hello world"
```
argo submit arguments-parameters.yaml -p message="goodbye world"
```
多个参数也可以被覆盖, argo CLI 提供了一个命令来加载 YAML 和 JSON 格式的参数文件; 以下是参数文件的示例
```
message: goodbye world
```
使用以下命令运行
```
argo submit arguments-parameters.yaml --parameter-file params.yaml
```
命令行参数可以用于覆盖默认的入口点, 并且可以调用在工作流规范中的任意模板; 例如, 如果你添加一个名为 `whalesay-caps` 的新版本 `whalesay` 模板, 但是你不想修改默认的入口点, 你可以调用类似如下的命令
```
argo submit arguments-parameters.yaml --entrypoint whalesay-caps
```
通过联合使用 `--entrypoint` 和 `-p` 参数, 你可以使用你喜欢的任意参数调用在工作流规范中的任意模板  
设置在 `spec.arguments.parameters` 的值是全局范围的, 可以通过 `{{workflow.parameters.parameter_name}}` 访问; 这在工作瑞的多个步骤之前传递信息是非常有用的; 例如, 你想使用设置在每个容器环境变量中的不同日志级别运行你的工作流, 你可以定义一个类似如下的 YAML 文件
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: global-parameters-
spec:
  entrypoint: A
  arguments:
    parameters:
    - name: log-level
      value: INFO

  templates:
  - name: A
    container:
      image: containerA
      env:
      - name: LOG_LEVEL
        value: "{{workflow.parameters.log-level}}"
      command: [runA]
  - name: B
    container:
      image: containerB
      env:
      - name: LOG_LEVEL
        value: "{{workflow.parameters.log-level}}"
      command: [runB]
```
在此工作流中, 步骤 `A` 和 `B` 将会使用相同的日志级别 `INFO`, 并且可以在工作流提交是使用 `-p` 标识轻易的改变

#### 步骤
