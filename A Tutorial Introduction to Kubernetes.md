# 一个 Kubernetes 介绍教程

> 看原文请点击[这里](http://okigiveup.net/a-tutorial-introduction-to-kubernetes/)

Kubernetes 是目前最流行的一种容器编排工具。我是在一年前开始写这篇文章的，当时我们决定在 Twyla 使用 Kubernetes ，也是从那时起， Kubernetes 的开发生态系统也已经开始变得势不可挡了。在我看来， Kubernetes 理应受到如此的重视，理由有如下几点：

* 它是建立在一套基本思想上的完整的解决方案，[这篇文章](https://research.google.com/pubs/archive/44843.pdf)中阐述了这些基本思想，并与 Google 开发的连续编排方案进行了比较，得到了一些经验与教训。
* 虽然 Kubernetes 是容器原生的（ container-native ），但它并不局限于一个单一的容器平台，并且这个容器平台也可以被扩展，获得如网络以及存储的特性。
* 除了适合不同工作流的各种模式之外，它还提供了一个开放的、设计良好的 API 。它还有一个非常规范的社区，来持续开发 API 。你需要努力去跟进，但你总会获得回报。

在这篇教程中，我想要写下我学习 Kubernetes 的过程，点明一些我还是一个初学者时遇到的一些坑，并试着来解释它运作过程中的最重要的概念。我不能保证我能够写的面面俱到，因为 Kubernetes 涉及的东西太多以致于无法全部放入这样一个博客中的教程里。

## 开始

使用 Minikube 是开始使用 Kubernetes 最简单的一种方式。如果你在一个云服务提供商有一个账号，而且想要弄清楚他们的平台上的集群运作的细节，这篇教程同样适合你，因为本教程中涉及到的命令适用于任何最近版本的 Kubernetes 。[这里](https://kubernetes.io/docs/getting-started-guides/minikube/)介绍了在你的电脑上使用 Minikube 的细节。你需要使用官方的 CLI 客户端—— kubectl 来操作 Kubernetes 的最小集群 Minikube ，你可以通过[这里](https://kubernetes.io/docs/tasks/tools/install-kubectl/)安装它。你同样会用到 Docker 来创建和推送容器镜像，你可以根据[这里](https://docs.docker.com/engine/installation/#supported-platforms)的指示来在你的电脑上安装 Docker 。

当你安装好了所有的东西，可以通过如下指令来确认他们是否可以访问：

    kubectl version
    docker version
    minikube version

你可以通过如下指令检查 Minikube 是否在运行，这个指令还能告诉你 Minikube 是否有更新：

    minikube status

如果 Minikube 没有在运行，你可以通过 `minikube start` 来运行它。一般当你安装了 Minikube ，他就会自动配置 kubectl 来接入它，你可以使用 `kubectl cluster-info` 来检查。它的输出就像下面这样：

    Kubernetes master is running at https://192.168.99.100:8443

如果 IP 地址不在 `192.168.*.*` 的范围，或者 kubectl 报出配置无效或者无法连接到集群，你需要运行 `minikube update-context` 来让 Minikube 来修复你的配置。

## kubectl 如何配置？

我认为简单的提一下如何配置 kubectl 比较好。 API 的端点和集群连接点默认是定义在 `\~/.kube/config` 文件中的。这个配置文件的位置可以使用环境变量 `KUBECONFIG` 修改，它的值应该是一串路径，如果你怀疑 kubectl 的行为异常与配置有关的话，别忘了检查一下这个环境变量的设置是否正确。像其他很多 Kubernetes 中的配置一样， kubectl 的配置文件也是 YAML 格式的。有两个顶级的键有直接的关系： `contexts` 和 `clusters` 。 clusters 的列表包含用户有权访问的不同集群的端点和认证信息。一个 context 将一个用户和 namespace 值结合起来，来访问集群。这些 context 中的一个在当前是活动的，你可以通过配置文件或者运行 `kubectl config current-context` 来找出哪一个是活动的。你同样可以运行 `kubectl config view` 来查看所有的配置。你可以为这条命令加上 `--minfy` 选项来限制展示的数据的量。

## Node 和 Namespace

有两个基本概念相对来说非常直观，不需要太多上下文就可以说明，它们是 Node 和 Namespace 。 Node 是 Kubernetes 集群中的最小单元，它可以是一个虚拟机，也可以是一台物理机。一个 Node 上会运行一个 `kubelet` 进程，这个进程负责与 Kubernetes 集群的 master 进行通信，并且也要负责使用正确的方式运行正确的容器。你可以通过 `kubectl get nodes` 来获得 Node 列表。如果你使用的是 Minikube 并且没有做任何其他配置，那么就只会有 1 个 Node 。你作为一个 Kubernetes 的用户是不会操作它们的，但是云服务提供商都会自动或手动的扩展 Kubernetes 集群中的 Node 的。

Namespace 提供了在概念上隔离子集群的方法。比如说，如果你在同一个集群上运行不同的应用栈，那么你就可以组织每一个应用的资源，并把它们放在同一个 namespace 下就行了。如果一个资源在创建的时候没有分配 namespace ，那么它就会在默认 namespace `default` 下被创建。虽然 namespace 不是必须的，但是它的确让避免命名冲突、[闲置资源位置](https://kubernetes.io/docs/concepts/policy/resource-quotas/)或者管理访问变得更加容易。如果你使用了 namespace ，但是不想在每条命令后都加上 `--namespace` 的话，你可以使用如下指令来设置你在当前 context 下默认的 namespace ：

    kubectl config set-context $(kubectl config current-context) --namespace=my-namespace

## Kubernetes 仪表盘

Kubernetes 自带了一个内置的仪表盘。你可以通过以下指令通过列出系统中的 pod 来查看它是否在运行：

    kubectl get pods -n kube-system

如果其中有 'kubernetes-dahsboard' 打头的列表项的话就表明它正在运行。为了看到这个仪表盘，我们首先运行 `kubectl proxy` 来代理到 Kubernetes 的 API 上。这个时候我们就可以通过 http://localhost:8001 来访问 Kubernetes 的 API 了。仪表盘的 URL [很复杂](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)，我原来可以使用 http://localhost:8001/ui ，但可能由于安全的问题这个 URL 就被改成现在这样了。

## 通过 Minikube 来使用一个本地构建的镜像

在接下来的教程中，我们会通过部署多个容器镜像来展示 Kubernetes 的一些特性。Kubernetes 使用 Docker 来获得和运行容器镜像，这代表着它使用的是常规的 Docker 容器的 pull 逻辑。这篇教程的目的是让你能够尽快的使用 Kubernetes 来运行你的服务，因此，我推荐你使用 Minikube 内的 Docker 而不是你本地的 Docker 。你可以通过 `eval $(minikube docker-env)` 来配置你的 Docker 来达到这一目的。现在，你可以从 Minikube 中得到很多的镜像，你可以通过 `docker ps` 来确认，如果有很多镜像来自 `gcr.io/google_containers` 就证明你做对了。这个到 Minikube 内部的 Docker Service 的代理只在当前 Shell 中有效，如果你起了另一个 Shell ，你还将使用你本地的 Docker 。

如果你对自己构建相同的 service 不感兴趣，你也可以从我的 [Docker.io](https://hub.docker.com/r/afroisalreadyin/) 中获得相同的镜像。

## 运行一个服务

我们从运行一个能够告诉我们集群中是否有 pod 正在运行的命令开始。我们要使用前面提到的 kubectl 客户端来运行 `kubectl get pods` 这条命令。我稍后会解释什么是 pod 。如果你按照我之前说的正确配置了客户端，那么你就能看到 `No resources found` 的信息。这个过程中 kubectl 做的事情是连接到配置中当前活跃的上下文的 minikube 上运行的 Kubernetes 集群，并展示出结果信息。kubectl 只是众多 API 客户端中的一个，比如 Python 客户端也是官方提供支持的一个。你可以通过增加一个 `--v=7` 参数来查看 kubectl 发送的 API 请求，但是要注意，这可能导致大量的文本输出。

Kubernetes 不会自己去找我们要运行什么，所以我们还要继续，并让它运行一个简单的应用，比如说 Kubernetes 的示例库中的简单 Python 应用。你首先需要克隆这个仓库，并寻找 `simple-python-app` 子文件夹，并使用如下指令来创建一个容器镜像：

    docker build -t kubetutorial/simple-python-app:v0.0.1

只要构建开始，你就可以在 `docker images` 的运行结果中看到它。确认完成之后我们就可以准备运行我们的第一个 Kubernetes 命令：

    kubectl run simple-python-app \
     --image=kubetutorial/simple-python-app:v0.0.1 \
     --image-pull-policy=IfNotPresent \
     --port=8080

很明显这条指令通过某种方式运行了我们刚刚构建的那个镜像，因为这个镜像的标签被传递给了 `--image` 这个参数。 `imagePullPolicy=IfNotPresent` 这个参数告诉 Docker 如果一个本地镜像存在的话就不用再尝试去 pull 它了。我们同时给出了这个服务所在的端口。这个端口一定要与应用绑定的端口一致，如果我们不提供这个信息， Kubernetes 就无法得知要是用哪个端口来连接这个应用。一个小提示：这个示例服务需要在通用接口 `0.0.0.0` 上绑定，而不是 `localhost` 或者 `127.0.0.1` 。

我们如何通过 Kubernetes 连接我们的服务呢？这个时候就是介绍 [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 这个 Kubernetes 中最重要的抽象概念的最佳时机了。同其他抽象概念一样， pod 是 Kubernetes API 资源，我们可以使用 kubectl 列出和查询它们。我们使用 `kubectl get pods` 来看看哪些 pod 正在运行。输出的结果大概是这个样子的：

|NAME|READY|STATUS|RESRARTS|AGE|
|----|-----|------|--------|---|
|simple-python-app-68543294-vhj7g|1/1|Running|0|21s|

很好！已经有一个 pod 在运行了，但是到底 pod 是什么？ pod 是 Kubernetes 中的基础应用单元。它是一个合成整体的容器们的集合，并且这些容器的生命周期也是统一管理的。这些容器部署在同一个节点，共享同一个操作系统 namespace ，存储卷以及 IP 地址。它们可以通过 localhost 相互联系，并且使用可以使用系统级的 IPC 机制比如共享内存。一个 pod 中包含些什么取决于在部署、水平扩展以及复制等方面上，作为一个单元的是什么。比如说，把一个数据存储的容器和一个应用容器放在一个 pod 中就是没有意义的，因为它们在扩展和复制方面都是彼此独立的。而真正应该与应用容器放在一起的应该是像进行日志聚合操作这样的容器。

现在我们知道了什么是 pod ，能够得到正在运行的 pod 的名字，也能通过我们之前使用的 kubectl proxy 来向 pod 发出请求。只要 proxy 在运行，你就可以通过前面的命令中声明的端口来访问 `simple-python-app` 容器。

    curl http://localhost:8001/api/v1/proxy/namespaces/default/pods/simple-python-app-68543294-vhj7g

我们也可以使用 `kubectl logs simple-python-add-68543294-vhj7g` 来查看 pod 的日志，它会展示我们的应用的标准输出。同样，我们使用下面的命令也可以像使用 `docker exec` 一样，在容器中执行命令。

    kubectl exec -ti simple-python-app-68543294-vhj7g CMD

想在 Docker 中一样， `-it` 表示需要分配一个 tty 并在交互模式下运行命令。 `kubectl exec` 允许你使用 `-c` 来选择容器。缺省情况下，如果 pod 中只有一个容器，那么默认使用的就是是 pod 中的这个唯一容器。

## pod 由谁来创建？

Kubernetes 在一个 pod 中运行我们的容器，我们还想知道 pod 到底是从哪来的，我们并没有让 Kubernetes 去创建一个 pod 。实际上，在 Kubernetes 中很少会让人手动去创建 pod 。如果用户想要自己去编排他的应用单元，并保证其可用性， Kubernetes 就不会新建 pod 。上面的 `kubectl run` 命令会开展一个新的部署（ Deployment ），可以通过如下指令列出这些部署：

    $ kubectl get deployments
    NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    simple-python-app   1         1         1            1           1s

Deployment 是 Kubernetes 中的一种特殊的资源，他们负责管理应用容器的生命周期。这种资源可以被称作为 controller ，他们可以算是 Kubernetes 的核心。你可以通过 `kubectl describe deployments simple-python-app` 来获得 Deployment 的更多详细信息。`describe` 这个子命令十分有用，你可以通过它获得所有资源的详细信息。它同样会列出与被描述的资源相关的资源以及时间的信息。对于这个 Deployment 来说，你可以通过 `kubectl describe` 看到不少信息。首先，有一个叫做 pod template 的东西，这个东西是用来在 Deployment 扩展的时候用来创建 pod 的。

那么当我们删除一个 pod 的时候会发生什么呢？为了实时的观察发生了什么，我建议你打开第二个终端，并在其中运行 `kubectl get pods -w` 。`-w` 选项会定时更新输出的信息。现在，使用 `kubectl delete pod simple-python-app-68543294-vhj7g` 删除已经存在的 pod 。在你观察 pod 列表的终端会暂时出现以下这种状态：

    NAME                                 READY     STATUS        RESTARTS   AGE
    simple-python-app-5c9ccf7f5d-8lbb2   1/1       Running       0          4s
    simple-python-app-5c9ccf7f5d-kl77s   1/1       Terminating   0          43s

在一个 pod 被删除的时候，另一个已经被创建了（ STATUS 也有可能是 ContainerCreating 而不是 RUNNING ）。Replica Sets 负责进行这种再创建。你可以使用 `kubectl describe` 看到 replica sets 属于一个 Deployment ，它列表的底部，在 events 之前。你可以看到两种列表：OldReplicaSets 和 NewReplicaSets 。两者之间的区别我会在后面的章节进行阐释。你也可以使用 `kubectl get replicasets` 命令列出 replica sets 。

使用 `kubectl describe replicaset $REPLICA_SET_NAME` 看看我们的 Deployment 创建的 replica set ，我们可以稍微看看下面几行：

    # ... snip
    Replicas:       1 current / 1 desired
    Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
    Pod Template:
    Labels:       pod-template-hash=4035281104
                    run=simple-python-app-2
    Containers:
        simple-python-app:
            Image:              kubetutorial/simple-python-app:v0.0.1
            Port:               8080/TCP
            Environment:        <none>
            Mounts:             <none>
            Volumes:              <none>
    Events:                 <none>

这个 replica set 负责保持我们的一个 simple-python-app 容器的 pod 处于运行状态，并且可以通过 1 current / 1 desired 可以判断它做到了。但是和 pod 一样，一般 replica set 也是由 Deployment 创建的，所以不必手工创建或修改它们。

## 网络知识简介

虽然 Replica Set 很棒很有通，但是他们对高可用并没有太大的帮助。在一个 pod 宕机的时候，会新起一个 pod ，但是它有一个不同的名字、一个不同的 IP 地址，并且有可能运行在一个完全不同的节点之上。还有，如果我们想在这些 Replica 之间做负载均衡的时候应该怎么办？如果 Kubernetes 仅仅提供了基于 pod 名的服务发现功能的话，那么这些服务的客户端就需要在客户端一侧来做负载均衡，并且保存一份 pod 的列表，这个列表需要在所有 pod 的生命周期时间发生的时候进行更新。还有如何将流入的流量路由到相应的服务呢？这些都是一些需要被简化的讨厌的问题。 Kubernetes 提供了非常简单的机制来实现高可用、负载均衡以及服务路由（ Ingress ）。所有的这些都基于 [Kubernetes node 和 pod 的网络要求](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)：

* 所有的 **容器** 可以在不使用 NAT 的情况下与其他的所有 **容器** 进行通信；
* 所有的 **节点（ Node ）** 可以在不使用 NAT 的情况下与所有 **容器** （反之亦然）进行通信；
* **容器** 看到的自己的 IP 地址与其他容器看到的它的 IP 地址应该相同。

可以使用任何一种[符合这种模型的网络选项](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)，其中 kubenet 是默认的一种配置。上面这些的要求听起来相对来说比较直观。我们可能认为每一个应用容器都会有一个它的 IP ，但实际上并不是这样，只有 pod 才有它自己的 IP 地址。如官方文档所示：

> 目前为止，本文档讨论了容器。事实上， Kubernetes 在 pod 这一级上分配 IP 地址。 同一个 pod 下的容器共享同一个网络命名空间，包括 IP 地址。

你同样可以通过 `kubectl get pods -o wide` 获得容器的私有 IP 地址来验证 pod 可以通过 IP 上的暴露端口来访问。然后，通过 `minikube ssh` 来登录到 Minikube 的 Node 上，你可以通过这个节点，使用 `curl $IP_ADDRESS:8080` 来请求这个服务。

那么属于同一个 Replica Set 的 pod 为了高可用、负载均衡以及服务发现的功能是如何组织的呢？说明这个问题的答案之前，首先要介绍另一个 Kubernetes 的概念。

## Services

我之前一直称呼我们那个用来演示的小型 Web App 为一个 Service ，但是在 Kubernetes 中， Service 有着完全不同的含义。一个 Kubernetes Service 就是一个允许一组松耦合的 pod 进行负载均衡、服务发现以及路由的抽象概念。通过 Service ， pod 可以在不影响应用可用性的情况下被替换和转移。我们可以看一个简单的例子——通过下面的指令，将我们的简单 Python 应用转化为一个 Service ：

  kubectl expose deploy simple-python-app --port 8080

如果现在你运行 `kubectl expose deploy simple-python-app --port 8080` ，你应该能看到一个有两个项的列表： `kubernetes` 和 `simple-python-app` 。 `kubernetes` 服务是基础设施的一部分，你不应该随意改动它。另一个服务就是我们要找的，尤其是列在 `CLUSTER-IP` 这一列下的 IP 地址。我们队这个 IP 地址感兴趣，这是因为它是一种特殊的东西，它是 Kubernetes 为这个新服务保留的一个虚拟 IP 地址。在同一段输出中，你可以看到 8080 端口被暴露出来。你现在可以使用 `minikube ssh` 登陆上 Minikube VM（其上运行着 Kubernetes Node），并使用 `kubectl expose deploy simple-python-app --port 8080` 来请求我们的新服务，它同样会正常返回。我们之前提到的网络要求确保了从这个 Node 到 Service IP 的可到达性。

当我们在一个 Replica Set 中使用多个 Pod 的时候，事情会变得更加有趣。为了看看影响，我们使用另一个会在其应答中返回更多信息的服务。这个服务在 `kubernetes-repository` 中的 `env-printer-app` 。当我们调用它的基本路径的时候，它会返回给我们它的环境变量。像前面的服务一样，我们可以使用如下指令创建一个容器：

  docker build -t kube-tutorial/env-printer-app:v0.0.1 .

我们将 Replica 的数量设置为 3 来启动这个 Deployment ，他将让 Kubernetes 用过正确的方式启动3个 pod 。我们使用下面的指令：

  kubectl run env-printer-app \
   --image=kube-tutorial/env-printer-app:v0.0.1 \
   --image-pull-policy=Never \
   --replicas=3 \
   --port=8080

现在，我们通过暴露这个 Deployment 来创建一个 Service ，但我们要在之前用的命令上做些修改：

  kubectl expose deploy env-printer-app --port 8080

一个名为 `env-printer-app` 的新服务会出现在 `kubectl get services` 的输出中。将 `CLUSTER-IP` 下的 IP 地址记为 `$IP_ADDRESS` 。并通过 ssh 登录到 minikube 上。然后运行几次下面的指令：

  curl -s $IP_ADDRESS:8080 | grep HOSTNAME

你应该能观察到 HOSTNAME 的值会在多个 pod 的名字之间变化。 Kubernetes 帮助我们将请求将请求分发到了不同的 pod 上，给了我们开箱即用的负载均衡功能。

这个短小的 demo 给我们映出了更多的问题。比如说 Service 在一个请求进来的时候怎么知道将它分配到哪个 pod 上？我们只能在集群内部访问我们的服务吗？我们如何从外部访问它？在我们能回答这些问题之前，我们首先需要通过一个更好的方式来看 Deployment 、 Service 以及其他资源。

## 使用命令行 VS. 使用清单文件

到目前为止，我们已经通过 `kubectl` 来使用了命令行接口。我们一直使用它，因为它的功能很完整，但是它不容易读、分享以及在一个仓库中进行组织。一种更好的组织 Kubernetes 资源的方法是使用清单文件。这些文件可能是 YAML 或是 JSON（YAML 更受欢迎），它有着更加结构化的格式来组织待创建资源和要执行的操作。一个清单文件采用的格式是使用一个带有元数据不同种类资源的列表以及一个 `spec` 。在实践中常见且推荐的一种做法是声明 API 的版本。每个不同的表项要使用三个破折号来分割，它在 YAML 中表示开启一个新的文档。这个分隔符是强制的，如果你不加它的话只有列表中的第一个表项会被处理。

在 [Kubernetes API 文档](https://kubernetes.io/docs/api-reference/v1.7/)中详细记录了资源规范。而更好的是， `kubectl` 命令是自描述的。如果想要得到有关 pod 的文档，执行 `kubectl explain pods` 命令就可以了。这条指令会打印出一个 pod 清单可以包含的各个字段，同时在前面会附加一小段描述。为了更深入这个清单树，你可以运行比如 `kubectl explain pod.metadata.labels` 的指令，它将会给出更多有关单个字段的详细信息。

如果你在在线文档或命令行文档中看了 Deployment 的入口，你会看到所有资源的元数据字段都是相同的，并且名字字段是必须的。这个字段让我们能够在命令中来引用它们，获得它们的详细信息或者删除它们，或者在其他清单文件中进行交叉引用。 spec 字段需要遵循 `DeploymentSpec` 配置，该配置应该有一个模板字段来描述将要部署的 pod 。反过来，这个模板必须要有它自己的元数据字段，以及一个包含了一个容器列表的 spec 字段。下面是一建立上述 Deployment 的 YAML 格式的例子：

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
        name: env-printer-app
    spec:
        replicas: 3
        template:
            metadata:
                labels:
                    app: env-printer-app
            spec:
                containers:
                - image: twyla.io/env-printer-app:v.0.0.1
                  imagePullPolicy: IfNotPresent
                  name: env-printer-app

从中我们有可能看到嵌套资源的共同模式，它们都有元数据，用于相互引用，模板用来告诉 Kubernetes 创建何种资源。还有很多其他的辅助信息，比如说 replica 字段。现在，你可以继续使用这个 YAML 文件，将它保存为 `kubernetes-repository/env-printer-app` 文件夹下的 `deploy.yaml` ，然后通过 `kubectl apply -f deploy.yaml` 来创建一个 Deployment 。

你可以使用 `kubectl get KIND NAME -o yaml` 来获得一个资源的 YAML 格式的详细描述。这个 YAML 文档相比你创建一个资源时提供的信息，可能包含了更多的信息，比如说被你缺省的默认配置以及那些由 Kubernetes 计算和设置的值。另一项非常棒依赖于 YAML 表示方法的特性是可以通过 `kubectl edit KIND NAME` 对资源进行编辑。这条指令将会取回资源的描述信息并将其加载到 `EDITOR`（或者 `KUBE_EDITOR` ）环境变量指定的编辑器中。只要你保存并退出，新的资源描述就会被应用到资源上。这是一种快速尝试解决方案的好方法，而不需要保留多个版本的资源定义。

## Service ，续

好了，我们说到哪了？我们现在有了一些运行在通过 Development 预分配和保持的 pod 里的 container 了，并把它们塞进了有着同一个 IP 地址的 Service 中。我们可以把这些都放进一个或者多个 YAML 文件中。现在这个地方是一个介绍 Kubernetes 中 Selector 这个很有趣并且很通用的特性的好时机。如果你继续想要获得我们刚才通过 `kubectl describe service env-printer-app` 创建的 `env-printer-app` 服务的详细信息，你将会看到一个开头为 ~Selector:~ 。 这个 Selector 配置会告诉你 Kubernetes 如何从 Service 的虚拟 IP 地址中找到相应的 pod 。如果你在查询的同时不做什么骚操作的话，这个 Selector 行的内容应该是 `run=env-printer-app` 。