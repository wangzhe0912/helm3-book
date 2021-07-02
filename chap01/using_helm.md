
# helm命令行

本文主要讲解使用helm命令行进行K8s包管理的一些基本操作。我们假定你已经完成了helm客户端的安装。

如果你只是想运行几个简单的命令，那么 [快速入门](./quickstart.md) 文档应该就已经能够满足你的需求了。
而本文讲解的命令相对来说更多且详解的更加清晰。

## 三个重要概念

在开始详解helm命令行操作之前，我们首先需要了解Helm的三个重要的概念：Chart，Repository，Release。

*Chart* 是Helm生态中的部署包。
它包含了为了在K8s中运行一个应用的所有资源信息。
Chart在K8s中的作用类似于 Homebrew包、APT DPKG包或者是YUM RPM文件一样。

*Repository* 是用于存储和共享Charts的一个平台，专门针对与K8s包的存储与共享。

*Release* 是一个Chart在一个K8s集群中运行的实体。在同一个K8s集群中，可以多次安装多一个 *chart* 。
每次安装一个 *chart*，都会创建一个新的 *Release* 。
例如对于一个MySQL Charts而言，如果你想要在你的K8s集群中创建两个数据库，那么你仅仅需要安装两次这个 *chart* 即可。
每次安装都会创建一个对应的 *Release* ，并且每次得到的  *Release* 名称都不相同。

当我们了解了这些概念后，我们可以这么理解Helm的作用：

Helm可以把Charts安装到K8s集群中，并在对于每次安装都会创建一个新的*release*。
同时，如果你想要找到一些有用的charts时，你可以去chart Repository中进行查询。

## 'helm search': 查询Charts

Helm提供了强大的搜索功能。
它可以用于搜索两种不同的源：

- `helm search hub` 用于搜索 [the Helm Hub](https://hub.helm.sh)，它可以从各个不同的Helm源进行聚合搜索。
- `helm search repo` 用于搜索你本地添加的helm源 (with `helm repo add`). 这种搜素方式是基于本地数据的，不需要公网连接即可完成。 

你可以通过 `helm search hub` 命令搜索公开提供的charts:

```console
$ helm search hub wordpress
URL                                                 CHART VERSION APP VERSION DESCRIPTION
https://hub.helm.sh/charts/bitnami/wordpress        7.6.7         5.2.4       Web publishing platform for building blogs and ...
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.6.3        v0.6.3      Presslabs WordPress Operator Helm Chart
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.7.1        v0.7.1      A Helm chart for deploying a WordPress site on ...
```

上述的搜索结果就是在Helm Hub中所有到的所有workpress charts的记录。

如果不加任何搜索过滤词，那么 `helm search hub` 命令将会搜索出所有可用的charts。

使用 `helm search repo` 命令, 你可以用于查询你已经本地添加repo源的charts。

```console
$ helm repo add brigade https://brigadecore.github.io/charts
"brigade" has been added to your repositories
$ helm search repo brigade
NAME                          CHART VERSION APP VERSION DESCRIPTION
brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```

Helm本身支持模糊搜索，所以你也可以仅仅输入关键词的一部分进行搜索：

```console
$ helm search repo kash
NAME            CHART VERSION APP VERSION DESCRIPTION
brigade/kashti  0.4.0         v0.4.0      A Helm chart for Kubernetes
```

通过搜索，我们可以找到我们需要的charts，一旦我们找到了我们需要的charts，接下来，就可以使用 `helm install` 来进行安装了。

## 'helm install': 安装一个包

想要安装一个部署包，我们可以使用 `helm install` 命令。
对于一个最简单的命令而言，我们需要传入两个参数：

1. release name
2. chart name

```console
$ helm install happy-panda bitnami/wordpress
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)

```

现在，`wordpress` chart已经正常安装了，安装charts后，会对应的创建一个新的release对象。
这个release对象的名称是 `happy-panda`。
Ps：如果你懒得起名，也可以添加 `--generate-name` 让Helm自动为你的release对象分配名称。

在安装过程中，helm客户端将会打印一些帮助信息比如创建了哪些资源、当前release的状态以及是否有
一些额外的配置工作需要手动操作等。

Helm按照以下顺序安装资源：

1. Namespace
2. NetworkPolicy
3. ResourceQuota
4. LimitRange
5. PodSecurityPolicy
6. PodDisruptionBudget
7. ServiceAccount
8. Secret
9. SecretList
10. ConfigMap
11. StorageClass
12. PersistentVolume
13. PersistentVolumeClaim
14. CustomResourceDefinition
15. ClusterRole
16. ClusterRoleList
17. ClusterRoleBinding
18. ClusterRoleBindingList
19. Role
20. RoleList
21. RoleBinding
22. RoleBindingList
23. Service
24. DaemonSet
25. Pod
26. ReplicationController
27. ReplicaSet
28. Deployment
29. HorizontalPodAutoscaler
30. StatefulSet
31. Job
32. CronJob
33. Ingress
34. APIService


helm本身并不会等待所有的资源对象成功创建后再退出。很多charts依赖的Docker镜像都很大，比如超过600M等。
拉取镜像到集群中本身也是一件相对耗时的事情。

为了查询release的状态或者查询配置相关信息，你可以使用 `helm status` 命令进行查询：

```console
$ helm status happy-panda                
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

上述内容都是该release的相关信息。

### 安装charts之前进行定制化改造

在前面的步骤中，我们仅仅使用了charts的默认配置进行安装。
然而在更多的时候，我们常常是需要对chart的配置进行一些定制化的修改的。

为了查询有哪些配置可以定制化修改，我们可以使用 `helm show values` 命令进行查询。

```console
$ helm show values bitnami/wordpress
## Global Docker image parameters
## Please, note that this will override the image parameters, including dependencies, configured to use the global value
## Current available global Docker image parameters: imageRegistry and imagePullSecrets
##
# global:
#   imageRegistry: myRegistryName
#   imagePullSecrets:
#     - myRegistryKeySecretName
#   storageClass: myStorageClass

## Bitnami WordPress image version
## ref: https://hub.docker.com/r/bitnami/wordpress/tags/
##
image:
  registry: docker.io
  repository: bitnami/wordpress
  tag: 5.6.0-debian-10-r35
  [..]
```

根据上述说明，我们可以将要重写的配置写入一个yaml格式的配置文件中，并且在安装charts的时候传递这些文件。

```console
$ echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
$ helm install -f values.yaml bitnami/wordpress --generate-name
```

上述配置说明我们将会创建一个默认的MariaDB用户，名称是user0，同时创建一个新的Database为user0db，并授予这个用户访问该DB的权限。
而其余的信息，则统一使用默认配置。

其实，我们有如下两种方式在安装的过程中自定义配置数据：

- `--values` (or `-f`): 指定一个yaml配置文件进行配置重写，也可以同时指定多个配置文件，最右侧的配置优先级最高。
- `--set`: 可以在命令行中传递重写的参数。

如果同时使用两个参数，--set的优先级会更高，重写到与--values中相同的配置项并与其他配置项合并。

使用--set重写的配置将会在ConfigMap中进行持久化。
同时，使用--set重写的配置还可以通过 `helm get values <release-name>` 进行查询。
通过--set重写的配置在 `helm upgrade` 操作时，如果附带了 `--reset-values` 参数，则将会被清除。

#### `--set` 的使用方式与局限性

`--set` 参数可以接收0个或多个键值对参数。一个最简单的例子是： `--set name=value` 。
它等价于如下的yaml格式：

```yaml
name: value
```

多个键值对时，它们之间使用 `,` 进行分隔。例如： `--set a=b,c=d` 等价于：

```yaml
a: b
c: d
```

此外，还有很多复杂的表达式，例如， `--set outer.inner=value` 等价于：

```yaml
outer:
  inner: value
```

列表可以用 `{` 和 `}` 进行包围，例如， `--set name={a, b, c}` 等价于：

```yaml
name:
  - a
  - b
  - c
```

自 Helm 2.5.0 版本后，还允许通过索引的方式访问列表，例如，`--set servers[0].port=80`等价于：

```yaml
servers:
  - port: 80
```

列表中的元素有多个Key时，设置方式  `--set servers[0].port=80,servers[0].host=example` 对应于：

```yaml
servers:
  - port: 80
    host: example
```

有时，你需要在 --set 中传递一些特殊字符，此时则需要通过 \ 进行转义，并用双引号包围，例如 `--set name=value1\,value2` 对应于：

```yaml
name: "value1,value2"
```

类似的是，对于key中包含一些特殊符号时，可以使用相同的策略， `--set nodeSelector."kubernetes\.io/role"=master` 对应于：

```yaml
nodeSelector:
  kubernetes.io/role: master
```

对于多层嵌套的结构而言，--set表达式使用相对复杂。因此在设计 `values.yaml` 文件时，要考虑 --set 的使用方式，尽量避免多层嵌套。

相关内容可以了解：[Values Files](../5chart_template_guide/values_files/)

### 更多的安装方式

`helm install` 命令可以用于安装各种各样的helm源，包括：

- A chart repository (之前的示例)
- 本地chart包 (`helm install foo foo-0.1.1.tgz`)
- 解压后的chart目录 (`helm install foo path/to/foo`)
- 完成的Chart包Url地址 (`helm install foo https://example.com/charts/foo-1.2.3.tgz`)

## 'helm upgrade' and 'helm rollback': Release的升级与回滚

当一个新版本的chart发布后，我们可以希望对我们的release进行升级。
此时，可以使用 `helm upgrade` 命令。

升级操作将基于你提供的信息和当前的release版本进行操作。
由于Kubernetes的charts可能会非常的复杂和庞大，因此，Helm会尝试进行增量升级。
也就说仅仅更新最近一次release变动的内容。

```console
$ helm upgrade -f panda.yaml happy-panda stable/mariadb
Fetched stable/mariadb-0.3.0.tgz to /Users/mattbutcher/Code/Go/src/helm.sh/helm/mariadb-0.3.0.tgz
happy-panda has been upgraded. Happy Helming!
Last Deployed: Wed Sep 28 12:47:54 2016
Namespace: default
Status: DEPLOYED
...
```

在上面的例子中， `happy-panda` release 的升级过程中使用了相同的charts，但是传递了一个新的yaml配置文件：

```yaml
mariadb.auth.username: user1
```

你可以用 `helm get values` 去查询新的配置是否正常生效了。

```console
$ helm get values happy-panda
mariadb:
  auth:
    username: user1
```

`helm get` 命令在查询集群中的发布信息时非常有用。
如上所示，我们可以看到从 `panda.yaml` 配置文件中获取的相关配置。

现在，如果某个版本在发行过程中存在问题，则很容易进行回滚。
使用`helm rollback [RELEASE] [REVISION]`就可以回滚至之前的版本。

```console
$ helm rollback happy-panda 1
```

上述的回滚操作将happy-panda回滚回了第一个版本。
release的版本号是一个增量增加的版本号。
每当进行一次安装、升级或者回滚等时，版本号都会+1。
一个新创建的release的版本号为1。
同时，我们还可以使用 `helm history [RELEASE]` 命令查询指定release的版本信息。

## 安装、升级、回滚中的一些有用的参数

在安装、升级、回滚等过程中，有一些有用的选项可以帮助我们定制操作的行为。
需要注意的是，下面列出的选项并不是全部的选项，更多的选项可以通过 `helm <command> --help` 进行查询。

- `--wait`: 等待所有的Pod状态变为Ready，PVC都绑定成功，Service完成IP分配。他将会阻塞等待直到超时或任务完成。
- `--timeout`: 超时时间，默认为 `5m0s`
- `--no-hooks`: 跳过hook

## 'helm uninstall': 删除一个Release对象

当我们想要删除一个release对象时，可以执行如下命令：

```console
$ helm uninstall happy-panda
```

上述命令将会从集群中移除对应的release。
你可以使用 `helm list` 命令查询当前已经部署的所有release对象。

```console
$ helm list
NAME            VERSION UPDATED                         STATUS          CHART
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
```

从上面的输出可以看到，`happy-panda` release已经被正常删除了。

在Helm之前的版本中，当一个release对象被删除后，它的删除记录还会被保留。
但是在Helm3中，删除release对象时也会同步删除对应的发布记录。
如果你想要包含release的删除记录，可以在删除时使用： `helm uninstall --keep-history`
此时，可以使用 `helm list --uninstalled` 命令查询当前已经被删除的release对象。

`helm list --all` 将会查询所有release对象，包括使用--keep-history参数删除的release。

```console
$  helm list --all
NAME            VERSION UPDATED                         STATUS          CHART
happy-panda     2       Wed Sep 28 12:47:54 2016        UNINSTALLED     mariadb-0.3.0
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
kindred-angelf  2       Tue Sep 27 16:16:10 2016        UNINSTALLED     alpine-0.1.0
```

由于现在release记录默认会被删除，因此想要回滚一个已经被删除的release对象是无法完成的。

## 'helm repo': Repositories操作

Helm3中，已经不再默认添加相关的repo源了。
`helm repo`命令组提供了一组命令用于添加、查询和删除repo源。

`helm repo list` 可以查询当前的repo源:

```console
$ helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
mumoshu         https://mumoshu.github.io/charts
```

`helm repo add` 命令可以添加新的repo源:

```console
$ helm repo add dev https://example.com/dev-charts
```

由于chart repo会频繁的变更，为了保证你的源信息与最新新的同步，你可以经常运行 `helm repo update` 进行源信息同步。

`helm repo remove` 可以用于删除repo源。

## 创建你自己的charts

[Chart Development Guide](../topics/charts.md) 详细介绍了如何制作自己的charts。
此时，你可以使用 `helm create` 命令来快速体验一下:

```console
$ helm create deis-workflow
Creating deis-workflow
```

此时就会有一个 `./deis-workflow` 目录. 你可以编辑它并且创建自己的模板。

当你编辑你的charts过程中，可以使用 `helm lint` 验证格式是否合法。

当你的包内容制作完成后，可以运行 `helm package` 命令进行打包。

```console
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

此时，制作完成的包可以通过 `helm install` 的方式进行安装:

```console
$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
...
```

打包后的charts可以被上传到repo仓库中。更多细节可以参考 [Helm Chart仓库](https://helm.sh/zh/docs/topics/chart_repository) 。

## 总结

本文主要讲解了 `helm` 客户端的基本使用方式，包含查询、安装、升级、卸载等。
同时，还讲解了其中一些有用的命令，例如 `helm status`, `helm get` 和 `helm repo`.

更多关于helm命令行相关的介绍可以使用内置的help命令进行查询：

`helm help`
