# Chart 模板开发引言

Chart 模板开发章节中，我们会讲解如何来制作 Chart。

首先，我们需要了解如何创建一个 Chart 并添加一个模板。

## Charts

如之前 [Charts详解](../chap03/charts.md) 中介绍，Helm chart的结构如下：

```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
  ...
```

templates/ 目录包括了模板文件。

当 helm 应用一个 charts 时，会通过模板渲染引擎将所有文件发送到 `templates/` 目录中。 然后收集模板的结果并发送给Kubernetes。

`values.yaml` 文件也会被导入到模板中。
这个文件包含了chart的默认值。
这些值会在用户执行`helm install`或`helm upgrade`时被覆盖。

`Chart.yaml` 文件包含了该chart的描述。

`charts/`目录可以包含其他的chart(称之为子chart)。

## 制作第一个 Charts

下面，我们会创建一个名为mychart的chart，然后会在chart中创建一些模板。

首先通过 `helm` 命令来创建一个 chart 。

```shell
helm create mychart
# Creating mychart
```

如果你看看 `mychart/templates/` 目录，会注意到一些文件已经存在了：

 - NOTES.txt: chart的"帮助文本"。这会在你的用户执行helm install时展示给他们。
 - deployment.yaml: 创建Kubernetes 工作负载的基本清单
 - service.yaml: 为你的工作负载创建一个 service终端基本清单。
 - _helpers.tpl: 放置可以通过chart复用的模板辅助对象。


然后我们要做的是**把它们全部删掉**！ 这样我们就可以从头开始学习我们的教程。

```shell
rm -rf mychart/templates/*
```

Ps: 编写生产环境级别的chart时，有这些chart的基础版本会很有用。因此在日常编写中，我们通常不会删除它们。

## 第一个模板

第一个创建的模板是ConfigMap。

Kubernetes中，配置映射只是用于存储配置数据的对象。其他组件，比如pod，可以访问配置映射中的数据。

因为配置映射是基础资源，对我们来说是很好的起点。

让我们以创建一个名为 mychart/templates/configmap.yaml的文件开始：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

Ps: 模板名称不需要遵循严格的命名模式。但是建议以.yaml作为YAML文件的后缀，以.tpl作为helper文件的后缀。

上述YAML文件是一个简单的配置映射，构成了最小的必需字段。因为文件在 mychart/templates/ 目录中，它会通过模板引擎传递。

像这样将一个普通YAML文件放在mychart/templates/目录中是没问题的。
当Helm读取这个模板时会按照原样传递给Kubernetes。

有了这个简单的模板，现在有一个可安装的chart了。现在安装如下：

```shell
helm install full-coral ./mychart
# NAME: full-coral
# LAST DEPLOYED: Tue Jul 20 09:04:09 2021
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
```

接下来，我们可以使用helm来查询版本信息并查看实际渲染后的模板。

```sh
helm get manifest full-coral
# ---
# # Source: mychart/templates/configmap.yaml
# apiVersion: v1
# kind: ConfigMap
# metadata:
#   name: mychart-configmap
# data:
#   myvalue: "Hello World"
```

其中，`helm get manifest` 命令后跟一个发布名称(full-coral)然后打印出了所有已经上传到server的Kubernetes资源。
每个文件以---开头表示YAML文件的开头，然后是自动生成的注释行，表示哪个模板文件生成了这个YAML文档。

从这个地方开始，我们看到的YAML数据确实是configmap.yaml文件中的内容。

现在卸载发布： `helm uninstall full-coral`。

### 添加一个简单的模板调用

将`name`硬编码到一个资源中不是很好的方式。
在一个namespace下，资源名称应该是唯一的。
因此，我们可能希望通过插入发布名称来生成name字段。

Ps: 由于DNS系统的限制，name字段长度限制为63个字符。因此发布名称限制为53个字符。
Kubernetes 1.3及更早版本限制为24个字符 (名称长度是14个字符)。

对应改变一下configmap.yaml：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

其中，我们唯一的修改就是 name 字段的取值，我们引入了模板命令 `{{ .Release.Name }}`。 

模板命令 `{{ .Release.Name }}` 将发布名称注入了模板。
值作为一个**命名空间对象**传给了模板，用点(.)分隔每个命名空间的元素。

其中：

 - Release前面的点表示从作用域最顶层的命名空间开始（稍后会谈作用域）。
 - 这样.Release.Name就可解读为“通顶层命名空间开始查找 Release对象，然后在其中找Name对象”。


Release是一个Helm的内置对象，关于 Release ，后面会进行更加详细的说明，现在只需要知道它可以显式获取发布对象的名称即可。

现在再次安装资源，并查询对应的完整 yaml 配置，可以看到的是：
**configmap的名称从之前的mychart-configmap 变为了 full-coral-configmap**。

由此我们已经看到了最基本的模板替换功能：**YAML文件有嵌入在{{ 和 }}之间的模板命令进行替换**。

下一部分，会深入了解模板， 但在这之前，有个快捷的技巧可以加快模板的构建过程中的调试速度。

当你想测试模板渲染但又不想安装任何内容时，
可以使用 `helm install --debug --dry-run goodly-guppy ./mychart` 命令。

在执行`helm install`的时候带上这两个参数就可以把对应的 values 值和生成的最终的资源清单文件打印出来，而不会真正的去部署一个release实例。

Ps：这种方式虽然会使得调试过程相对简单，但是不能完全保证一定可以在 K8s 中正常工作，因此，调试完成后，还是需要真正 install 进行测试。
