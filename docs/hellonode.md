---
assignees:
- dchen1107
- pwittrock

---

* TOC
{:toc}

## 介绍

这个教程的目标是，让你吧一个 node.js 的简单的 Hello World 应用，在 Kubernetes 上部署成为一个复制应用。
我们会告诉你，如何将你的代码变成一个 Docker 镜像，并且在[Google Container Engine](https://cloud.google.com/container-engine/)上运行它。

这个是本教程帮助你了解本教程每一个细节的图解。 整个教程，我们都会参考这个图解。

![image](/images/hellonode/image_1.png)

Kubernetes 是可以运行在许多不同环境的开源项目，无论是笔记本还是高可用多点集群，公有云还是本地环境，无论是虚拟机还是物理机。使用例如 Google Container Engine (一个 Google 托管的 Kubernetes 运行环境) 托管环境的话，会让你不用设置底层基础架构而更专注地体验 Kubernetes。

## 设置和要求

如果你还没有 Google 账号(Gmail 或者 Google Apps), 你必须[创建一个](https://accounts.google.com/SignUp)。然后登录 Google 云平台([console.cloud.google.com](http://console.cloud.google.com)) 并创建一个项目:

![image](/images/hellonode/image_2.png)

![image](/images/hellonode/image_3.png)

记住这个项目 ID; 下面将称它为`PROJECT_ID`。把 项目 ID 存储到一个变量会很方便使用。

确保你有一个 Linux 终端，你将使用它的命令行来控制你的群集。你可以使用 [Google Cloud Shell](https://console.cloud.google.com?cloudshell=true)。按照这个教程它会预安装，所以你可以跳过下面大多数的环境配置。

用下面的命令来储存你的项目 ID 到一个变量，会让接下来的操作非常便捷。

 ```shell
 export $PROJECT_ID="your-project-id"
 ```

接下来, 在开发者控制台[启用结算](https://console.cloud.google.com/billing) 以便使用 Google 云资源 and [开启容器引擎 API](https://console.cloud.google.com/project/_/kubernetes/list).

新用户会获得[300美元的试用额度](https://console.cloud.google.com/billing/freetrial?hl=en). 运行本示例只会用掉几美元的额度。Google 容器引擎的定价在[这里](https://cloud.google.com/container-engine/pricing).

下面, 你要[下载 Node.js](https://nodejs.org/en/download/).You can skip this and the steps for installing Docker and Cloud SDK if you're using Cloud Shell.

并且安装 [Docker](https://docs.docker.com/engine/installation/) 和 [Google Cloud SDK](https://cloud.google.com/sdk/).

最后, 在安装完 Google Cloud SDK 以后，运行一下命令来安装[`kubectl`](http://kubernetes.io/docs/user-guide/kubectl-overview/):

```shell
gcloud components install kubectl
```

全部设置完毕以后，你就可以创建容器的镜像了，运行 Node 应用, 在本地运行 Kubernetes 集群, 并且部署 Kubernetes 集群到 Google Container Engine。让我们马上开始行动吧!

## 创建你的 Node.js 应用

首先要写这个程序。保存代码到 "`hellonode/`" 文件夹下的`server.js`:

#### server.js

```javascript
var http = require('http');
var handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World!');
};
var www = http.createServer(handleRequest);
www.listen(8080);
```

现在运行这个简单的命令 :

```shell
node server.js
```

在浏览器打开 http://localhost:8080/ 你应该会看到你的 "Hello World!"。如果你用 Cloud Shell，那么你需要使用[Web Preview](https://cloud.google.com/shell/docs/using-web-preview)来打开这个 URL。

停止运行 node 只需按下 Ctrl-C.

接下来让我们来打包这个程序到 Docker 容器中.

## 创建一个 Docker 容器镜像

下面, 创建一个文件, 也在`hellonode/` 文件夹中 创建一个叫 `Dockerfile` 的文件. 一个 Dockerfile 定制你想要构建的镜像。 Docker 容器镜像可以基于其他镜像进行拓展，所以我们这个镜像将要基于已经存在的 Node 镜像来构建。

#### Dockerfile

```conf
FROM node:4.4
EXPOSE 8080
COPY server.js .
CMD node server.js
```

这个 "秘方" 会让搭建的 Docker 镜像基于 Docker 镜像仓库的 Node.js LTS镜像, 开放 8080 端口, 复制我们的 `server.js` 文件到镜像，并且启动 Node  服务。

现在运行 `docker build` 来构建你的容器，并用 Google Container Registry 得到的 `PROJECT_ID` 来标记这个容器:

```shell
docker build -t gcr.io/$PROJECT_ID/hello-node:v1 .
```
现在你有了一个，具有一段可以正常运行的代码的容器。

让我们用 Docker 来试一下你的镜像:

```shell
docker run -d -p 8080:8080 --name hello_tutorial gcr.io/$PROJECT_ID/hello-node:v1
```

你可以通过浏览器访问你的应用，如果你愿意，你也可以使用`curl` 或者 `wget`:

```shell
curl http://localhost:8080
```

你应该能看到 `Hello World!`

**如果你得到一个来自于 `Docker for Mac`的 `Connection refused` 信息，请确认你使用的是 Docker 最新版本 (1.12 或者更高).如果你在 OSX 上用的是 `Docker Toolbox`,确保你用的是虚拟机的 IP 而不是 localhost**

```shell
curl "http://$(docker-machine ip YOUR-VM-MACHINE-NAME):8080"
```

让我们停止运行这个容器。你可以使用`docker ps`列出容器列表:

```shell
docker ps
```

You should something like see:

```shell
CONTAINER ID        IMAGE                                 COMMAND                  NAMES
c5b6d4b9f36d        gcr.io/$PROJECT_ID/hello-node:v1      "/bin/sh -c 'node ser"   hello_tutorial
```

Now stop the running container with

```
docker stop hello_tutorial
```

现在镜像已经可以如期运行，并标记了你的`PROJECT_ID`，我们推送它到[Google 容器仓库](https://cloud.google.com/tools/container-registry/), 每一个 Google 云项目具有一个私有 Docker 镜像库 (而且也在 Google 云平台之外) :

```shell
gcloud docker push gcr.io/$PROJECT_ID/hello-node:v1
```

如果一切顺利, 你应该会在控制台中看到容器镜像: *Compute > Container Engine > Container Registry*. 我们现在有一个项目内可用的， Kubernetes 可以访问和编排的 Docker 镜像.

![image](/images/hellonode/image_10.png)

## 创建你的群集

集群由一个 master API server 和一组叫做 node 的虚拟机。

首先,选择一个[Google Cloud Project zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones) 来运行你的服务，
在这个教程中我们将使用 **us-central1-a**。
这个配置命令如下：

```
gcloud config set compute/zone us-central1-a
```

现在，通过`gcloud`命令行工具来创建一个群集：

```shell
gcloud container clusters create hello-world
```

另外,通过: [Google Cloud Console](https://console.cloud.google.com): *Compute > Container Engine > Container Clusters > New container cluster* 创建群集。将名称设置为'hello-world'，使用所有默认设置选项。

你会得到一个具有三个可以部署你容器镜像的 node（节点）群集(这个步骤会花上几分钟)。

![image](/images/hellonode/image_11.png)

是时候来部署你的`容器式应用` 到 Kubernetes 集群了!

```shell
gcloud container clusters get-credentials hello-world
```

**文档其余部分都需要 Kubernetes 的客户端和服务端版本为1.3。运行`kubectl version`查看你当前的版本**  1.2版看[这个文档](https://github.com/kubernetes/kubernetes.github.io/blob/release-1.2/docs/hellonode.md).

## 创建你的 pod

一个 Kubernetes **[pod](/docs/user-guide/pods/)** 是一个容器组， 目的是捆绑管理网络，它可以包含一个或者多个容器。

使用 `kubectl run` 命令来创建一个 pod:

```shell
kubectl run hello-node --image=gcr.io/$PROJECT_ID/hello-node:v1 --port=8080
```

如图所示， `kubectl run` 创建一个 **[deployment](/docs/user-guide/deployments/)**。 推荐使用 `deployment` 创建和扩容 pod。在这个例子中，一个新的`deployment`管理一个运行 *hello-node:v1* 镜像的单独的 pod 副本。

要查看我们刚创建`deployment`运行：

```shell
kubectl get deployments

NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   1         1         1            1           3m
```

要查看我们刚通过`deployment`创建`pod`运行：

```shell
kubectl get pods

NAME                         READY     STATUS    RESTARTS   AGE
hello-node-714049816-ztzrb   1/1       Running   0          6m
```

要查看 pod 输入或输出错误 (hello-node 镜像没有输出，所以在这记录会是空的) 运行:

```shell
kubectl logs <POD-NAME>
```

查看群集的 metadata 运行:

```shell
kubectl cluster-info
```

查看群集的 events 运行:

```shell
kubectl get events
```

查看 kubectl 设置运行:

```shell
kubectl config view
```

完整的 kubectl命令 文档 **[在这](/docs/user-guide/kubectl-overview/)**:

A此时你应该已经让我们的容器在 Kubernetes 的控制下运行了，但是我们还没能够让外界访问它。

## 允许外部访问

默认情况下，pod 只可以在 Kubernetes 群集内部访问，为了使`hello-node`容器可以从 Kubernetes 虚拟网络外访问，你必须用 Kubernetes **[service](/docs/user-guide/services/)** 让 pod 对外开放。

在我们的开发机上，我们可以使用`kubectl expose`以及`--type="LoadBalancer"`标识来公开 pod `--type="LoadBalancer"`标志了需要从外部 IP 访问:

```shell
kubectl expose deployment hello-node --type="LoadBalancer"
```

**如果失败了，请查看你的客户端和服务端版本是否都是1.3，详情请参阅 [创建群集](#create-your-cluster) 部分。**

此命令中使用这个标识指定了我们将使用底层架构来提供负载均衡 (是这种情况 [Compute Engine 负载均衡](https://cloud.google.com/compute/docs/load-balancing/))。注意,我们开放的是 deployment, 问不是直接公开 pod.  这将会让所有由 deployment 管理的 pod 负载均衡(这只有一个 pod，我们一会会复制多个).

由 Kubernetes master 创建的负载均衡和相关的运算方法，对象池以及防火墙规则，会让这个`service`在 Google 云平台之外可访问.

查看该服务关联的多个 IP:

```shell
kubectl get services hello-node

NAME         CLUSTER_IP    EXTERNAL_IP     PORT(S)    AGE
hello-node   10.3.246.12                   8080/TCP   23s
```

 `EXTERNAL_IP` 可能需要过几分钟才可见。如果没有`EXTERNAL_IP`，那么需要等上几分钟再重试。

```shell
kubectl get services hello-node

NAME         CLUSTER_IP    EXTERNAL_IP     PORT(S)    AGE
hello-node   10.3.246.12   23.251.159.72   8080/TCP   2m
```

注意 这里罗列出了两个 IP，都监听8080端口.  `CLUSTER_IP` 只在你的内部网络中有效。  `EXTERNAL_IP` 是外部的.  这个例子中，外部 IP 是 23.251.159.72.

现在你应该可以通过这个地址来访问这个`service`: http://EXTERNAL_IP**:8080** 或者运行 `curl http://EXTERNAL_IP:8080`

![image](/images/hellonode/image_12.png)

如果通过浏览器或者 CURL 来访问新的 web 服务，
你应该可以看到一些运行日志：

```shell
kubectl logs <POD-NAME>
```

Kubernetes 的强大功能之一就是他可以很容易的扩容你的应用程序。假设你突然需要增加你的应用;你只需要告诉`deployment`一个新的 pod 副本总数即可:

```shell
kubectl scale deployment hello-node --replicas=4
```

现在你有4个应用副本了， 每个都在群集上独立运行，并能负载均衡他们之间的流量。

```shell
kubectl get deployment

NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   4         4         4            3           40m
```

```shell
kubectl get pods

NAME                         READY     STATUS    RESTARTS   AGE
hello-node-714049816-g4azy   1/1       Running   0          1m
hello-node-714049816-rk0u6   1/1       Running   0          1m
hello-node-714049816-sh812   1/1       Running   0          1m
hello-node-714049816-ztzrb   1/1       Running   0          41m
```

注意 **declarative approach** 在这是要声明你要运行多少实例，而不是启动多少新实例或停止几个实例。Kubernetes reconciliation loops 只要知道你实际要的是什么，然后采取行动。

这张图显示了我们的 Kubernetes 的状态:

![image](/images/hellonode/image_13.png)

## 滚动升级你的网站

与往常一样，修复 bug 或者添加功能的应用，部署到生产环境。 Kubernetes 可以让你不影响用户使用的情况下，部署一个新的版本到生产环境。

首先, 让我们修改程序。在开发机上，编辑 server.js 的输出消息:

```javascript
  response.end('Hello Kubernetes World!');
```

我们构建并上传一个具有新标识的容器到仓库:

```shell
docker build -t gcr.io/$PROJECT_ID/hello-node:v2 .
gcloud docker push gcr.io/$PROJECT_ID/hello-node:v2
```

构建和上传会非常快，因为我们用到了 Docker 的缓存机制。

我们现在已经为 Kubernetes 能够顺利更新部署提供了一个新版本的应用程序。为了区分新镜像，我们需要修改即存的 *hello-node deployment*
`gcr.io/$PROJECT_ID/hello-node:v1` 为 `gcr.io/$PROJECT_ID/hello-node:v2`。为此，我们需要使用 `kubectl set image` 命令.

```shell
kubectl set image deployment/hello-node hello-node=gcr.io/$PROJECT_ID/hello-node:v2
```

这会用新的镜像来更新`deployment`，它会创建新的 pod 并删除旧的 pod。

```
kubectl get deployments

NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   4         5         4            3           1h
```

当发生这种情况的时候，用户应该不会看到服务中断。过一会，他们就将访问新版本的应用程序了。更多细节你可以查看[deployment 文档](/docs/user-guide/deployments/).

希望只要你愿意，这个`deployment`, 扩容以及功能更新就会部署到你的环境中 (你的 GKE/Kubernetes 群集), Kubernetes 让你更专注于你的应用，而不用考虑基础设施。

## Kubernetes Web UI (可选)

随着Kubernetes, 图形化 Web 界面（控制台）也已经发布了。在群集中会默认启用。
通过这个用户界面你会很容易上手，并且更容易，方便的发现和使用 CLI 与系统交互。

尽情享受 Kubernetes 图形管理，用它来部署容器化应用，以及监视和管理你的群集！

![image](/images/docs/ui-dashboard-workloadview.png)

通过[Dashboard 之旅](/docs/user-guide/ui/)更多地了解 Web 界面。

## 删除它

这只是一个演示!所以，要记得关闭它，否则会产生费用的。让我们来学一下如何关闭它。

删除`deployment` (这也会删除正在运行的 pod) 和 `service` (还会删除外部负载均衡):

```shell
kubectl delete service,deployment hello-node
```

删除群集:

```shell
gcloud container clusters delete hello-world
```

You should see:

```
The following clusters will be deleted.
 - [hello-world] in [us-central1-a]

Do you want to continue (Y/n)?

Deleting cluster hello-world...done.
Deleted [https://container.googleapis.com/v1/projects/<$PROJECT_ID>/zones/us-central1-a/clusters/hello-world].
```

这回删除正在运行在 Google Compute Engine 的群集实例。

最后通过 `gsutil` 来删除 你的 Docker 仓库的镜像，gcloud 安装的时候应该已经安装了该命令工具，
有关`gsutil`的更多信息请查看[gsutil 文档](https://cloud.google.com/storage/docs/gsutil)

要列出我们在本教程前面创建的镜像

```shell
gsutil ls
```

你可以看到:

```shell
gs://artifacts.<$PROJECT_ID>.appspot.com/
```

然后 删除这个路径下所有的 `镜像` 运行:

```shell
gsutil rm -r gs://artifacts.<$PROJECT_ID>.appspot.com/
```

当然，你也可以删除整个项目，但请注意，你必须先关闭计费，此外当结算周期结束以后，项目才会被删除。
