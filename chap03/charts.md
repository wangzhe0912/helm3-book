# Charts详解

Helm使用的包的格式我们称之为 `charts`。
一个Charts中描述了一组相互关联的Kubernetes资源。
Charts既能部署复杂的应用，例如Memcached Pod，也能部署一些复杂的应用，例如一套包含了HTTP Server、数据据、缓存等等的完整项目。

Charts由一组特定的目录结构格式的文件组成的包，它们可以按照版本进行管理和部署。

如果你仅仅想要下载和查询Charts包而不想直接安装它们，那么你可以使用这个命令：
`helm pull chartrepo/chartname`。

本文档主要用于讲解Charts的基本格式，并且讲解了构建Charts的基本指南。

## Chart文件的结构

Charts是由一组文件组成了一个目录。其中，目录的名称与Charts的名称一致。
例如，一个WordPress应用的Chart通常会存储在一个 `wordpress/` 目录中。

在这个目录中，Helm包期望的结构如下：

```text
wordpress/
  Chart.yaml          # YAML file 包含chart的基本信息。
  LICENSE             # OPTIONAL: chart的许可信息。
  README.md           # OPTIONAL: 用于介绍chart的说明文档。
  values.yaml         # Chart的默认配置文件。
  values.schema.json  # OPTIONAL: A JSON Schema指名values.yaml的数据格式。
  charts/             # 一个目录，其中包含该Charts依赖的所有子Charts。
  crds/               # 自定义资源定义文件
  templates/          # 模板目录，与Value文件结合后，可以生成对应的合法的K8s资源文件。
  templates/NOTES.txt # OPTIONAL: 简单的使用说明
```

Helm预留了 `charts/`, `crds/`, and `templates/` 目录以及上面所列出的文件作为特殊文件。而其他名称的文件、目录将会被原样保留。

## Chart.yaml 文件说明

`Chart.yaml` 是一个Chart中必备的文件，它包含如下字段：

```yaml
apiVersion: chart API 版本 (required)
name: chart 名称 (required)
version: chart 版本 (required)
kubeVersion: K8s兼容版本 (optional)
description: Chart文本描述 (optional)
type: chart类型 (optional)
keywords:
  - chart的关键词 (optional)
home: 项目主页地址 (optional)
sources:
  - 项目源代码 (optional)
dependencies: # A list of the chart requirements (optional)
  - name: Chart名称 (nginx)
    version: Charts版本 ("1.2.3")
    repository: Chart地址或别名 ("@repo-name")
    condition: (optional) 一个Yaml路径，解析可以得到Bool值，从来判断是否存在依赖 (e.g. subchart1.enabled )
    tags: # (optional)
      - Tag可以用于将charts分组并统一启用/禁用
    enabled: (optional) 是否启动，决定是否加载
    import-values: # (optional)
      - ImportValues保存原始值映射到父键中引用。
      - 每条记录可以是一个字符串或者是一组子列表。
    alias: (optional) Chart别名，当该Chart被多次引用时会有用
maintainers: # (optional)
  - name: 项目成员名称 (required for each maintainer)
    email: 项目成功email (optional for each maintainer)
    url: 项目成功个人地址 (optional for each maintainer)
icon: SVG或PNG图像地址 (optional).
appVersion: 应用版本 (optional). This needn't be SemVer.
deprecated: 是否作废 (optional, boolean)
annotations:
  example: 一组annotation key的名称 (optional).
```

其他的字段全部会被忽略

### Charts的版本控制

每一个chart都必须包含一个版本号，同时版本号必须符合 [SemVer2](https://semver.org/spec/v2.0.0.html) 标准。
在 repositories 中，通过 chart名称 + 版本号 来唯一标识对应的Chart包。

例如，一个 `nginx` charts的Version字段可以设置为 `version: 1.2.3`。

此时，Chart包的名称将会是：

```text
nginx-1.2.3.tgz
```

SemVer2还支持一些复杂的版本设置，例如： `version: 1.2.3-alpha.1+ef365`。但是一定需要符合SemVer标准。

`Chart.yaml` 文件中的 `version` 字段在很多Helm工具中都会用到，例如CLI工具。
当创建一个包时， `helm package` 命令将会从 `Chart.yaml` 文件中找出对应的版本信息并作为部署包中的Token信息。
Helm要求Chart包名称中的版本号必须与 `Chart.yaml` 中的Version信息一致。

### `apiVersion` 字段

对于必须Helm 3以上版本安装的Chart而言，`apiVersion` 字段应该设置为 `v2`。
而对于同时支持早期Helm版本的Charts而言，`apiVersion` 字段应该设置为 `v1`。

从v1版本迁移到v2版本:

- v2版本中增加了一个 `dependencies` 字段用于定义Charts之间的依赖关系，而在v1版本中，则是使用了一个单独的 `requirements.yaml` 文件来表示依赖。[Chart Dependencies](#chart-dependencies)).
- type字段区分了应用charts与library charts。[Chart Types](#chart-types)).

### `appVersion` 字段

`appVersion` 字段与 `version` 字段无关，它是一个用于指定应用版本的方式。
例如，`drupal` Chart可能有一个 `appVersion: 8.2.1` 用于表明Charts中Drupal的版本默认为 `8.2.1`。
这个字段其实是一个提示字段，对应Chart Version计算等而言不起任何作用。

### `kubeVersion` 字段

`kubeVersion` 可以用于定义当前charts支持的Kubernetes版本。
Helm在安装charts服务时将会验证kubernetes的版本是否兼容，如果kubernetes版本不兼容，则会安装失败。

版本约束表达式空格分隔的多个且条件，例如：
```
>= 1.13.0 < 1.15.0
```

如果想要表达 或 的含义，需要使用 || 分隔符。

```
>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0
```

在上面的示例中，`1.14.0` 版本被排除了，这种方式适用于当某个版本中确定存在某个特定的bug时，可以阻止Chart错误运行。

在版本限制表达式中，除了支持 `=` `!=` `>` `<` `>=` `<=` 这些操作符外，还支持如下这些简单符号：

 * 对于闭区间范围而言， `1.1 - 2.3.4` 等价于 `>= 1.1 <= 2.3.4`.
 * 通配符 `x`, `X` and `*`。 例如 `1.2.x` 等价于 `>= 1.2.0 < 1.3.0`.
 * 小区间范围, 例如 `~1.2.3` 等价于 `>= 1.2.3 < 1.3.0`.
 * 大区间范围, 例如 `^1.2.3` 等价于 `>= 1.2.3 < 2.0.0`.

关于更多的版本约束表达式使用方式，可以参考：[Masterminds/semver](https://github.com/Masterminds/semver).

### 废弃 Charts

在Chart Repository中管理charts时，有时我们需要废弃一个chart。这时，我们可以用到`Chart.yaml`文件中的`deprecated`字段。
如果一个chart的 **latest** 版本的chart被标记为废弃，那么会将这个Chart都标记为废弃。
当然，后续你还能用相关的Chart名发布新的chart并取消废弃标记。
关于废弃chart的工作流可以参考：[kubernetes/charts](https://github.com/helm/charts)。
主要包含如下几步：

1. 更新chart的 `Chart.yaml` 表示该chart作废。
2. 在 Chart Repository 中发布新的chart版本。
3. 从源代码库删除该chart代码。

### Chart类型

`type` 字段定义了chart的类型。
其中，包含两种类型的Type，分别是：`application` 和 `library`。
默认的Type类型为`application`，它是标准类型的Chart，可以用于直接操作。
而 [library chart](./library_charts.md) 则是一种用于提供一些工具和函数来构建其他Chart的Chart。
library chart与application chart最本质的区别是它不能直接安装，通常也不会包含任何的资源对象。

Ps：如果把一个 application chart的type字段改为 `library`，那么它就可以作为一个library chart。
此时，该chart中的所有工具和函数可以被使用，但是所有的资源对象都会失效。

