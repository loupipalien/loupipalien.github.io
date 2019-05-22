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
以下, 我们在一个 kubernetes 集群上使用一个 Argo 工作流运行相同的容器; 请务必阅读注释, 因为它们提供了非常有用的解释
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
在本示例中, 我们将会看到如何创建多步骤的工作流, 如何在工作流规范中定义多个模板, 以及如何创建内嵌的工作流; 请务必阅读注释, 因为它们提供了非常有用的解释
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: hello-hello-hello

  # This spec contains two templates: hello-hello-hello and whalesay
  templates:
  - name: hello-hello-hello
    # Instead of just running a container
    # This template has a sequence of steps
    steps:
    - - name: hello1            # hello1 is run before the following steps
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2a           # double dash => run after previous step
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2a"
      - name: hello2b           # single dash => run in parallel with previous step
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2b"

  # This is the same template as from the previous example
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```
以上工作流规范打印了三个不同的 "hello"; `hello-hello-hello` 模板由三步组成; 名为 `hello1` 的第一步将会串行运行, 接下来的 `hello2a` 和 `hello2b` 的两步将会并行运行; 使用 argo CLI 命令, 我们可以图形化展示这个工作流规范的执行历史, `hello2a` 和 `hello2b` 的两步是相互并行运行的
```
STEP                                     PODNAME
 ✔ arguments-parameters-rbm92
 ├---✔ hello1                   steps-rbm92-2023062412
 └-·-✔ hello2a                  steps-rbm92-685171357
   └-✔ hello2b                  steps-rbm92-634838500
```

#### DAG
作为步骤序列指定的一种可选方式, 你可以通过指定每个任务的依赖将工作流定义为一个直接非循环图 (DAG); 这可以简化复杂工作流的维护, 并可在运行任务时运行最大化的并行  
在一下的工作流中, 步骤 `A` 最先运行, 因为它没有任何依赖, 一旦 `A` 完成, 步骤 `B` 和 `C` 会并行运行, 一旦 `B` 和 `C` 完成, 步骤 `D` 便可运行
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
  - name: diamond
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```
依赖图可以有多个 [根节点](https://argoproj.github.io/docs/argo/examples/dag-multiroot.yaml); 在 DAG 中的模板或者在步骤中的模板, 它们可以是 DAG 或者步骤模板; 这将允许将复杂的工作流拆分为可管理的分片

#### 构件
>注意: 你需要配置一个构件库来运行这个示例; 配置构件库见 [这里](https://github.com/argoproj/argo/blob/master/ARTIFACT_REPO.md)

当运行工作流时, 生成或消费构件是非常常见的步骤; 通常, 一个步骤的输出构件可能会用于后续步骤的输入构件  
以下的工作流规范由两个串行运行的步骤组成; 名为 `generate-artifact` 的第一步将会使用 `whalesay` 模板生成一个构件, 这个生成的构件将会被名为 `pring-message` 第二步消费
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-passing-
spec:
  entrypoint: artifact-example
  templates:
  - name: artifact-example
    steps:
    - - name: generate-artifact
        template: whalesay
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          # bind message to the hello-art artifact
          # generated by the generate-artifact step
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      # generate hello-art artifact from /tmp/hello_world.txt
      # artifacts can be directories as well as files
      - name: hello-art
        path: /tmp/hello_world.txt

  - name: print-message
    inputs:
      artifacts:
      # unpack the message input artifact
      # and put it at /tmp/message
      - name: message
        path: /tmp/message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
```
`whalesay` 模板会使用 `cowsay` 命令生成一个名为 `/tmp/hello-world.txt` 的文件; 这个 `outputs` 的文件将会被作为一个名为 `hello-art` 的构件; 通常, 构件的 `path` 是一个文件夹而不是一个文件; `print-message` 模板会获取一个名为 `message` 的输入构件, 将其解压到名为 `/tmp/message` 的路径, 然后使用 `cat` 命令打印 `/tmp/message` 中的内容; 在 `generate-artifact` 步骤中 `artifact-example` 模板将传递生成的 `hello-art` 构件, 在 `print-message` 步骤中将作为 `message` 输入构件; DAG 模板使用任务前缀来引用另一个任务, 例如 `{{tasks.generate-artifact.outputs.artifacts.hello-art}}`

#### 工作流规范的结构
我们现在已经充分了解了关于工作流规范的基础组件, 可以来回顾下它的基础结构了
- 包含 metadata 的 Kubernetes 头
- Spec 体
  - 带可选参数的 entrypoint 调用
  - template 定义列表
- 每个 template 定义
  - template 的 name
  - 可选的 inputs 列表
  - 可选的 outputs 列表
  - container 调用 (叶子 template) 或者 steps 列表
  - 每个 step 的 template 调用

总的来说, 工作流规范由一系列的 Argo 模板组成, 每个模板由一些可选的输入段, 输出段以及一个容器调用或者步骤列表, 每个步骤中调用另一个模板  
注意, 工作流规范中的控制器段将会接受相同的选项作为 pod 规范的控制段, 包括了但不限于环境变量, 密钥, 挂载卷, 类似的还有卷声明和多个卷  

#### 密钥
Argo 支持与 Kubernetes Pod 规范相同的 secrets 语法和机制, 这允许以环境变量或者卷挂的方式访问 secret, 更多信息见 [Kubernetes 文档](https://kubernetes.io/docs/concepts/configuration/secret)
```
# To run this example, first create the secret by running:
# kubectl create secret generic my-secret --from-literal=mypassword=S00perS3cretPa55word
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: secret-example-
spec:
  entrypoint: whalesay
  # To access secrets as files, add a volume entry in spec.volumes[] and
  # then in the container template spec, add a mount using volumeMounts.
  volumes:
  - name: my-secret-vol
    secret:
      secretName: my-secret     # name of an existing k8s secret
  templates:
  - name: whalesay
    container:
      image: alpine:3.7
      command: [sh, -c]
      args: ['
        echo "secret from env: $MYSECRETPASSWORD";
        echo "secret from file: `cat /secret/mountpath/mypassword`"
      ']
      # To access secrets as environment variables, use the k8s valueFrom and
      # secretKeyRef constructs.
      env:
      - name: MYSECRETPASSWORD  # name of env var
        valueFrom:
          secretKeyRef:
            name: my-secret     # name of an existing k8s secret
            key: mypassword     # 'key' subcomponent of the secret
      volumeMounts:
      - name: my-secret-vol     # mount file containing secret at /secret/mountpath
        mountPath: "/secret/mountpath"
```

#### 脚本和结果
通常, 在工作流规范中我们仅仅想用一个模板来执行一个 here-script 的指定脚本 (也被成为 `here document`); 这个示例展示如何实现
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scripts-bash-
spec:
  entrypoint: bash-script-example
  templates:
  - name: bash-script-example
    steps:
    - - name: generate
        template: gen-random-int-bash
    - - name: print
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "{{steps.generate.outputs.result}}"  # The result of the here-script

  - name: gen-random-int-bash
    script:
      image: debian:9.4
      command: [bash]
      source: |                                         # Contents of the here-script
        cat /dev/urandom | od -N2 -An -i | awk -v f=1 -v r=100 '{printf "%i\n", f + r * $1 / 65536}'

  - name: gen-random-int-python
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        i = random.randint(1, 100)
        print(i)

  - name: gen-random-int-javascript
    script:
      image: node:9.1-alpine
      command: [node]
      source: |
        var rand = Math.floor(Math.random() * 100);
        console.log(rand);

  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo result was: {{inputs.parameters.message}}"]
```
`script` 关键字允许使用 `source` 标签指定脚本内容; 这将会创建一个包含脚本内容的临时文件, 然后将临时文件名作为最终参数传递给 `command`, 这应该是一个可以执行脚本内容的解释器  
`script` 特性的使用也会分配脚本运行的标准输出到一个名为 `result` 的特殊输出参数; 这允许你在剩余的工作流规范中使用运行脚本的结果; 在这个示例中, 结果被 print-message 模板简单的打印

#### 输出参数
输出参数提供了一种通用的机制, 将步骤的结果作为一个参数而不是作为一个构件; 这允许你使用任何类型步骤的结果, 不仅仅是 `script`, 对于条件测试, 循环, 参数等等也是如此; 输出参数的工作机制类似于 `script result`, 只是将输出参数的值设置为一个生成文件的内容而不是 `stout` 的内容
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: output-parameter-
spec:
  entrypoint: output-parameter
  templates:
  - name: output-parameter
    steps:
    - - name: generate-parameter
        template: whalesay
    - - name: consume-parameter
        template: print-message
        arguments:
          parameters:
          # Pass the hello-param output from the generate-parameter step as the message input to print-message
          - name: message
            value: "{{steps.generate-parameter.outputs.parameters.hello-param}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo -n hello world > /tmp/hello_world.txt"]  # generate the content of hello_world.txt
    outputs:
      parameters:
      - name: hello-param       # name of output parameter
        valueFrom:
          path: /tmp/hello_world.txt    # set the value of hello-param to the contents of this hello-world.txt

  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```
DAG 模板使用任务前缀来引用另一个任务, 例如 `{{tasks.generate-parameter.outputs.parameters.hello-param}}`

#### 循环
在编写工作流时, 将一系列的输入循环迭代通常是非常有用的, 如以下示例
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-
spec:
  entrypoint: loop-example
  templates:
  - name: loop-example
    steps:
    - - name: print-message
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "{{item}}"
        withItems:              # invoke whalesay once for each item in parallel
        - hello world           # item 1
        - goodbye world         # item 2

  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```
我们也可以迭代条目集合
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-maps-
spec:
  entrypoint: loop-map-example
  templates:
  - name: loop-map-example
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        withItems:
        - { image: 'debian', tag: '9.1' }       #item set 1
        - { image: 'debian', tag: '8.9' }       #item set 2
        - { image: 'alpine', tag: '3.6' }       #item set 3
        - { image: 'ubuntu', tag: '17.10' }     #item set 4

  - name: cat-os-release
    inputs:
      parameters:
      - name: image
      - name: tag
    container:
      image: "{{inputs.parameters.image}}:{{inputs.parameters.tag}}"
      command: [cat]
      args: [/etc/os-release]
```
我们也可以传递条目列表作为参数
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-arg-
spec:
  entrypoint: loop-param-arg-example
  arguments:
    parameters:
    - name: os-list                                     # a list of items
      value: |
        [
          { "image": "debian", "tag": "9.1" },
          { "image": "debian", "tag": "8.9" },
          { "image": "alpine", "tag": "3.6" },
          { "image": "ubuntu", "tag": "17.10" }
        ]

  templates:
  - name: loop-param-arg-example
    inputs:
      parameters:
      - name: os-list
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        withParam: "{{inputs.parameters.os-list}}"      # parameter specifies the list to iterate over

  # This template is the same as in the previous example
  - name: cat-os-release
    inputs:
      parameters:
      - name: image
      - name: tag
    container:
      image: "{{inputs.parameters.image}}:{{inputs.parameters.tag}}"
      command: [cat]
      args: [/etc/os-release]
```
我们甚至可以动态的生成条目列表来迭代
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-result-
spec:
  entrypoint: loop-param-result-example
  templates:
  - name: loop-param-result-example
    steps:
    - - name: generate
        template: gen-number-list
    # Iterate over the list of numbers generated by the generate step above
    - - name: sleep
        template: sleep-n-sec
        arguments:
          parameters:
          - name: seconds
            value: "{{item}}"
        withParam: "{{steps.generate.outputs.result}}"

  # Generate a list of numbers in JSON format
  - name: gen-number-list
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import json
        import sys
        json.dump([i for i in range(20, 31)], sys.stdout)

  - name: sleep-n-sec
    inputs:
      parameters:
      - name: seconds
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for {{inputs.parameters.seconds}} seconds; sleep {{inputs.parameters.seconds}}; echo done"]
```

#### 条件语句
我们也支持条件语句执行, 如以下示例
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # flip a coin
    - - name: flip-coin
        template: flip-coin
    # evaluate the result in parallel
    - - name: heads
        template: heads                 # call heads template if "heads"
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails
        template: tails                 # call tails template if "tails"
        when: "{{steps.flip-coin.outputs.result}} == tails"

  # Return heads or tails based on a random number
  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        result = "heads" if random.randint(0,1) == 0 else "tails"
        print(result)

  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]

  - name: tails
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was tails\""]
```

#### 递归
每个模板都可以递归调用! 作为以上 coin-flip 模板的变种, 我们可以持续执行 flip coins 直到它出现 heads
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-recursive-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # flip a coin
    - - name: flip-coin
        template: flip-coin
    # evaluate the result in parallel
    - - name: heads
        template: heads                 # call heads template if "heads"
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails                     # keep flipping coins if "tails"
        template: coinflip
        when: "{{steps.flip-coin.outputs.result}} == tails"

  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        result = "heads" if random.randint(0,1) == 0 else "tails"
        print(result)

  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]
```
为了比较, 以下是运行一系列的 coinflip 的结果
```
$ argo get coinflip-recursive-tzcb5
...
STEP                         PODNAME                              MESSAGE
 ✔ coinflip-recursive-vhph5
 ├---✔ flip-coin             coinflip-recursive-vhph5-2123890397
 └-·-✔ heads                 coinflip-recursive-vhph5-128690560
   └-○ tails

STEP                          PODNAME                              MESSAGE
 ✔ coinflip-recursive-tzcb5
 ├---✔ flip-coin              coinflip-recursive-tzcb5-322836820
 └-·-○ heads
   └-✔ tails
     ├---✔ flip-coin          coinflip-recursive-tzcb5-1863890320
     └-·-○ heads
       └-✔ tails
         ├---✔ flip-coin      coinflip-recursive-tzcb5-1768147140
         └-·-○ heads
           └-✔ tails
             ├---✔ flip-coin  coinflip-recursive-tzcb5-4080411136
             └-·-✔ heads      coinflip-recursive-tzcb5-4080323273
               └-○ tails
```
在第一个执行中, coin 立马出现了 heads 然后我们停了下来; 在第二个运行中, coin 在最终出现 heads 前出现了三次 tail, 我们才停了下来

#### 退出处理器
退出处理器是一个在工作流结尾总是执行的模板, 无论成功还是失败;　以下是一些通用的退出处理器
- 在工作流运行后的清理
- 发送工作流的状态通知 (例如: email/slack)
- 将 pass/fail 的状态发送到一个 webhook 结果 (例如: github 构建结果)
- 重新提交或者提交另一个工作流

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exit-handlers-
spec:
  entrypoint: intentional-fail
  onExit: exit-handler                  # invoke exit-hander template at end of the workflow
  templates:
  # primary workflow template
  - name: intentional-fail
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo intentional failure; exit 1"]

  # Exit handler templates
  # After the completion of the entrypoint template, the status of the
  # workflow is made available in the global variable {{workflow.status}}.
  # {{workflow.status}} will be one of: Succeeded, Failed, Error
  - name: exit-handler
    steps:
    - - name: notify
        template: send-email
      - name: celebrate
        template: celebrate
        when: "{{workflow.status}} == Succeeded"
      - name: cry
        template: cry
        when: "{{workflow.status}} != Succeeded"
  - name: send-email
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo send e-mail: {{workflow.name}} {{workflow.status}}"]
  - name: celebrate
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo hooray!"]
  - name: cry
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo boohoo!"]
```

#### 超时
限制工作流的的耗时, 你可以使用 `activeDeadlineSeconds` 设置变量
```
# To enforce a timeout for a container template, specify a value for activeDeadlineSeconds.
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: timeouts-
spec:
  entrypoint: sleep
  templates:
  - name: sleep
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for 1m; sleep 60; echo done"]
    activeDeadlineSeconds: 10           # terminate container template after 10 seconds
```

#### 卷
以下示例动态的创建了一个卷, 然后在工作流的步骤中使用这个卷
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: volumes-pvc-
spec:
  entrypoint: volumes-pvc-example
  volumeClaimTemplates:                 # define volume, same syntax as k8s Pod spec
  - metadata:
      name: workdir                     # name of volume claim
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi                  # Gi => 1024 * 1024 * 1024

  templates:
  - name: volumes-pvc-example
    steps:
    - - name: generate
        template: whalesay
    - - name: print
        template: print-message

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo generating message in volume; cowsay hello world | tee /mnt/vol/hello_world.txt"]
      # Mount workdir volume at /mnt/vol before invoking docker/whalesay
      volumeMounts:                     # same syntax as k8s Pod spec
      - name: workdir
        mountPath: /mnt/vol

  - name: print-message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo getting message from volume; find /mnt/vol; cat /mnt/vol/hello_world.txt"]
      # Mount workdir volume at /mnt/vol before invoking docker/whalesay
      volumeMounts:                     # same syntax as k8s Pod spec
      - name: workdir
        mountPath: /mnt/vol
```
卷对于作流中从一个步骤到另一个步骤需要移动大量数据的情况是非常有用的; 依赖于系统, 一些卷可以被多个步骤并行访问; 在一些示例中, 你可能想访问一个存在的卷而不是动态的创建/销毁
```
# Define Kubernetes PVC
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-existing-volume
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi

---
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: volumes-existing-
spec:
  entrypoint: volumes-existing-example
  volumes:
  # Pass my-existing-volume as an argument to the volumes-existing-example template
  # Same syntax as k8s Pod spec
  - name: workdir
    persistentVolumeClaim:
      claimName: my-existing-volume

  templates:
  - name: volumes-existing-example
    steps:
    - - name: generate
        template: whalesay
    - - name: print
        template: print-message

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo generating message in volume; cowsay hello world | tee /mnt/vol/hello_world.txt"]
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol

  - name: print-message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo getting message from volume; find /mnt/vol; cat /mnt/vol/hello_world.txt"]
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol
```

#### 后台容器
当工作流持续执行是, Argo 工作流可以启动在后台运行的容器 (也被称为 `daemon containers`); 需要注意的是, 当工作流退出了守护进程被调用的模板范围, 守护进程将会自动被销毁; 守护容器对于启动用于测试的服务是非常有用的 (例如: fixtures); 在运行大量模拟以将数据库作为用于收集和组织结果的守护进程时, 我们发现它非常有用, 守护进程与 sidecars 相比的最大优势在于它们的存在可以持续跨越多个步骤甚至整个工作流程
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: daemon-step-
spec:
  entrypoint: daemon-example
  templates:
  - name: daemon-example
    steps:
    - - name: influx
        template: influxdb              # start an influxdb as a daemon (see the influxdb template spec below)

    - - name: init-database             # initialize influxdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl -XPOST 'http://{{steps.influx.ip}}:8086/query' --data-urlencode "q=CREATE DATABASE mydb"

    - - name: producer-1                # add entries to influxdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: for i in $(seq 1 20); do curl -XPOST 'http://{{steps.influx.ip}}:8086/write?db=mydb' -d "cpu,host=server01,region=uswest load=$i" ; sleep .5 ; done
      - name: producer-2                # add entries to influxdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: for i in $(seq 1 20); do curl -XPOST 'http://{{steps.influx.ip}}:8086/write?db=mydb' -d "cpu,host=server02,region=uswest load=$((RANDOM % 100))" ; sleep .5 ; done
      - name: producer-3                # add entries to influxdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl -XPOST 'http://{{steps.influx.ip}}:8086/write?db=mydb' -d 'cpu,host=server03,region=useast load=15.4'

    - - name: consumer                  # consume intries from influxdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl --silent -G http://{{steps.influx.ip}}:8086/query?pretty=true --data-urlencode "db=mydb" --data-urlencode "q=SELECT * FROM cpu"

  - name: influxdb
    daemon: true                        # start influxdb as a daemon
    container:
      image: influxdb:1.2
      restartPolicy: Always             # restart container if it fails
      readinessProbe:                   # wait for readinessProbe to succeed
        httpGet:
          path: /ping
          port: 8086

  - name: influxdb-client
    inputs:
      parameters:
      - name: cmd
    container:
      image: appropriate/curl:latest
      command: ["/bin/sh", "-c"]
      args: ["{{inputs.parameters.cmd}}"]
      resources:
        requests:
          memory: 32Mi
          cpu: 100m
```
DAG 模板使用任务前缀引用另一个任务, 例如 `{{tasks.influx.ip}}`

#### Sidecars
sidecar 是另一个容器, 可以和主容器在同一个 pod 中并行执行, 在创建多容器的 pod 非常有用
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: sidecar-nginx-
spec:
  entrypoint: sidecar-nginx-example
  templates:
  - name: sidecar-nginx-example
    container:
      image: appropriate/curl
      command: [sh, -c]
      # Try to read from nginx web server until it comes up
      args: ["until `curl -G 'http://127.0.0.1/' >& /tmp/out`; do echo sleep && sleep 1; done && cat /tmp/out"]
    # Create a simple nginx web server
    sidecars:
    - name: nginx
      image: nginx:1.13
```
在以上示例中, 我们创建了一个 sidecar 容器运行 nginx 作为简单的 web 服务器; 容器的启动顺序是随机的, 所以这个示例中的主容器将会轮询 nginx 容器直到其准备好服务请求; 在设计多容器系统时这是一个好的设计模式: 在运行你的主程序前总是等待你需要的服务启动

#### 硬编码构件
在 Argo 中, 你可以使用你喜欢的任何容器镜像来生成任何类型的构件; 然而在实际操作中, 我们发现一些构件是非常通用的, 所以这有了对 git, http, s3 构件的内建支持
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hardwired-artifact-
spec:
  entrypoint: hardwired-artifact
  templates:
  - name: hardwired-artifact
    inputs:
      artifacts:
      # Check out the master branch of the argo repo and place it at /src
      # revision can be anything that git checkout accepts: branch, commit, tag, etc.
      - name: argo-source
        path: /src
        git:
          repo: https://github.com/argoproj/argo.git
          revision: "master"
      # Download kubectl 1.8.0 and place it at /bin/kubectl
      - name: kubectl
        path: /bin/kubectl
        mode: 0755
        http:
          url: https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl
      # Copy an s3 bucket and place it at /s3
      - name: objects
        path: /s3
        s3:
          endpoint: storage.googleapis.com
          bucket: my-bucket-name
          key: path/in/bucket
          accessKeySecret:
            name: my-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-s3-credentials
            key: secretKey
    container:
      image: debian
      command: [sh, -c]
      args: ["ls -l /src /bin/kubectl /s3"]
```

#### Kubernetes 资源
在许多示例中, 你可能在 Argo 工作流中想管理 Kubernetes 资源; 资源模板允许你创建, 删除, 更新任何类型的 Kubernetes 资源
```
# in a workflow. The resource template type accepts any k8s manifest
# (including CRDs) and can perform any kubectl action against it (e.g. create,
# apply, delete, patch).
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: k8s-jobs-
spec:
  entrypoint: pi-tmpl
  templates:
  - name: pi-tmpl
    resource:                   # indicates that this is a resource template
      action: create            # can be any kubectl action (e.g. create, delete, apply, patch)
      # The successCondition and failureCondition are optional expressions.
      # If failureCondition is true, the step is considered failed.
      # If successCondition is true, the step is considered successful.
      # They use kubernetes label selection syntax and can be applied against any field
      # of the resource (not just labels). Multiple AND conditions can be represented by comma
      # delimited expressions.
      # For more details: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
      successCondition: status.succeeded > 0
      failureCondition: status.failed > 3
      manifest: |               #put your kubernetes spec here
        apiVersion: batch/v1
        kind: Job
        metadata:
          generateName: pi-job-
        spec:
          template:
            metadata:
              name: pi
            spec:
              containers:
              - name: pi
                image: perl
                command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
              restartPolicy: Never
          backoffLimit: 4
```
使用这种方式创建的资源独立于工作流; 如果你想当工作流删除时, 资源也被删除, 你可以在工作流资源中使用 [ Kubernetes garbage collection](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/) 作为一个所属者引用 ([示例](https://argoproj.github.io/docs/argo/examples/k8s-owner-reference.yaml))  
注意: 当修补时, 资源可以接受另一个属性: `megerStrategy`, 它的值可以是 `strategic`, `meger`, `json`; 如果没有使用这个属性, 则默认是 `strategic`; 要记住的是自定义资源不能使用 `strategic` 修补, 所以必须选择不同的策略; 例如, 假定你使用了 [CronTab CustomResourceDefinition](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#create-a-customresourcedefinition) 定义, 以下是一个 CronTab 的示例
```
apiVersion: "stable.example.com/v1"
kind: CronTab
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```
这个 CronTab 可以使用以下 Argo 工作流修改
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: k8s-patch-
spec:
  entrypoint: cront-tmpl
  templates:
  - name: cront-tmpl
    resource:
      action: patch
      mergeStrategy: merge                 # Must be one of [strategic merge json]
      manifest: |
        apiVersion: "stable.example.com/v1"
        kind: CronTab
        spec:
          cronSpec: "* * * * */10"
          image: my-awesome-cron-image
```

#### Docker-in-Docker 使用 Sidecars
TODO

#### 自定义模板变量引用
TODO
