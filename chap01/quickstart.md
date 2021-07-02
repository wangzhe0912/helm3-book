# Helm引言

你是一个Helm新人吗？下面跟着我来一步步的学习Helm吧！

# QuickStart

本节主要帮助你快速了解helm的使用。

## 前置条件

如果想要使用helm，需要首先满足如下条件：

1. 有一个k8s集群
2. 如果要开启安全性验证，需要提前设置相关的安全性策略
3. 正确安装和配置helm

### 部署K8s集群或者能够访问一套K8s集群

- 需要提前部署好K8s集群，对于最新的稳定版helm而言，通常对应于K8s集群的版本是倒数第二个稳定版。
- 本地有一个已经提前配置到的 `kubectl` 工具.

关于Helm与K8s的版本兼容性问题，可以访问 [Helm Version Support Policy](https://helm.sh/docs/topics/version_skew/) 查询完整信息。

## 安装Helm

安装Helm本身非常简单，仅需要下载Helm客户端即可。
下载的方式可以使用一些包管理工具，例如homebrew或者Github官方发布产出地址： [the official releases page](https://github.com/helm/helm/releases)

关于安装的更多细节，可以访问 [install.md](./install.md) 文档进行了解。

## 初始化 Helm Chart 仓库

当Helm安装完成后，接下来就是需要配置charts仓库源了。一个常用的官方Helm稳定源的地址如下：

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

接下来，你就能罗列出来当前你已经安装的charts: 

```shell
helm search repo bitnami
# NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION
# bitnami/bitnami-common                      	0.0.9        	0.0.9        	DEPRECATED Chart with custom templates used in ...
# bitnami/airflow                             	10.2.2       	2.1.0        	Apache Airflow is a platform to programmaticall...
# bitnami/apache                              	8.5.7        	2.4.48       	Chart for Apache HTTP Server
# bitnami/aspnet-core                         	1.3.7        	3.1.16       	ASP.NET Core is an open-source framework create...
# bitnami/cassandra                           	7.6.2        	3.11.10      	Apache Cassandra is a free and open-source dist...
# bitnami/cert-manager                        	0.1.4        	1.4.0        	Cert Manager is a Kubernetes add-on to automate...
# bitnami/common                              	1.6.1        	1.6.1        	A Library Helm Chart for grouping common logic ...
# bitnami/consul                              	9.2.14       	1.10.0       	Highly available and distributed service discov...
# bitnami/contour                             	4.3.9        	1.16.0       	Contour Ingress controller for Kubernetes
# bitnami/dataplatform-bp1                    	5.0.0        	1.0.0        	OCTO Data platform Kafka-Spark-Solr Helm Chart
# ... and many more
```

## 安装一个示例charts

为了安装一个charts，你将会用到的是 `helm install` 命令。helm有多种方式查询和安装charts，其中最简单的方式使用官方的stable charts。 

```shell
helm repo update              # 确保拿到最新的charts
helm install bitnami/mysql --generate-name
# NAME: mysql-1625213307
# LAST DEPLOYED: Fri Jul  2 16:08:32 2021
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:...
```

在上面的例子中，我们使用了 `bitnami/mysql` charts，release的名称叫做 `mysql-1625213307`。

如果想要了解charts相关的概要信息，可以使用 `helm show chart bitnami/mysql` 命令。

想要获得更加详细的信息，则可以运行如下命令： `helm show all bitnami/mysql` 。

只要你install 一个 chart后，会自动创建一个新的release版本。所以同一个chart其实可以在一个集群中install多次。
而每个release版本都可以进行独立的管理和升级。

`helm install` 的命令非常强大，想要更加深入的了解，可以阅读 [using_helm.md](./using_helm.md) 文档。

## release对象

通过如下命令可以查询当前所有的Helm的release对象：

```shell
helm ls
# NAME            	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
# mysql-1625213307	default  	1       	2021-07-02 16:08:32.405932 +0800 CST	deployed	mysql-8.7.0	8.0.25
```

`helm list` 命令将会打印目前已经部署的一组release对象。

## 删除一个release对象

如果想要删除一个release对象，可以使用 `helm uninstall` 命令：

```shell
helm uninstall mysql-1625213307
# Removed mysql-1625213307
```

这一操作将会从K8s中删除 `mysql-1625213307` 这个release对象中所有的关联资源以及release历史。

如果增加 `--keep-history` 参数，release历史将会被保存下来，仍然能够查询这一release对象之前的release历史。

```shell
helm status mysql-1625213307
# Status: UNINSTALLED
...
```

此时，即使你删除了该release对象，但由于Helm本身追踪记录了你的release历史，因此你仍然能够对集群对象历史进行审计，甚至必要的时候还可以通过 `helm rollback` 命令来恢复release对象。 

## 帮助文档

为了了解更多的帮助文档, 可以使用 `helm help` 进行查询。

同时，对于每个子命令而言，也可以使用 `-h` 参数打印相关的帮助文档。

```shell
helm get -h
```
