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