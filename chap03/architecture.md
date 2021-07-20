# Helm架构

本文主要会介绍 Helm 整体的体系结构。

## Helm 的定位和目标

Helm 主要用于管理名为 chart 的 Kubernetes 包的工具。

Helm可以做以下的事情：

 - 从头开始创建新的chart。
 - 将chart打包成归档(tgz)文件。
 - 与存储chart的仓库进行交互。
 - 在现有的Kubernetes集群中安装和卸载chart。
 - 管理与Helm一起安装的chart的发布周期。


对于Helm，有三个重要的概念：

 - chart 创建Kubernetes应用程序所必需的一组信息。
 - config 包含了可以合并到打包的chart中的配置信息，用于创建一个可发布的对象。
 - release 是一个与特定配置相结合的chart的运行实例。


## Helm 组件

Helm是一个可执行文件，执行时分成两个不同的部分：

### Helm 客户端

Helm客户端 是终端用户的命令行客户端。负责以下内容：

 - 本地chart开发。
 - 仓库管理。
 - 管理发布。
 - 与 Helm 库建立接口
   - 发送安装的 chart
   - 发送升级或卸载现有发布的请求


### Helm 库

Helm 库提供执行所有 Helm 操作的逻辑，与Kubernetes API服务交互并提供以下功能：

 - 结合chart和配置来构建版本。
 - 将chart安装到Kubernetes中，并提供后续发布对象。
 - 与Kubernetes交互升级和卸载chart。

独立的 Helm 库封装了 Helm 逻辑以便不同的客户端可以使用它。


## 实现方式

Helm 的客户端和 lib 库都是使用 Go 来开发的。

Helm lib 库是使用 K8s 提供的客户端 Lib 库与 K8s 进行通信，通信的方式是 REST + JSON 格式。
Helm 的元数据存储在 K8s 的密钥中，自己本身不需要任何数据库。

Helm 的配置文件通常是以 YAML 格式编写的。
