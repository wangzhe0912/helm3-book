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

## LICENSE, README and NOTES

在一个Charts中，同样会包含一些文件用于描述Chart的安装、配置、使用和许可信息。

LICENSE是一个文本文件，包含这个Chart的[license](https://en.wikipedia.org/wiki/Software_license)。
charts可以包含一个许可证，它可能会在模块中存在替换逻辑，因此可能不能仅仅是一个配置。
如果有必要的话，可以对Chart中的不同应用创建不同的LICENSE。

Chart中的README文档通常要求为 Markdown 格式（README.md），并且包含如下内容：

- 对Chart提供的应用和服务的功能描述。
- 该图片使用前的准备工作和相关依赖。
- `values.yaml` 文件中的参数可选项与默认值
- 其他与Chart安装与配置相关的任何信息

当一些用户界面显示该Chart详情时，首先看到的将会是这个chart中`README.md`文件的内容。

在chart中，同样还可以包含一个 `templates/NOTES.txt` 文件用于在安装完成或查询release状态时能够打印到标准输出中。
该文件可以用于打印相关的使用方式、安装后的下一步、以及其他Chart相关的任何信息。
例如，可以提供用于连接到数据库或访问Web UI的指令。
由于在运行“ helm install”或“ helm status”时此文件被打印到STDOUT，因此建议保持内容简短并指向README以获取更多详细信息。

## Chart 依赖

在Helm中，一个Chart可能会依赖一系列其他的Chart。
这些依赖关系将会在 `Chart.yaml` 文件中的 `dependencies` 字段中进行动态解析 或者是 通过 `charts/` 目录进行手工管理。

### 使用 `dependencies` 字段管理依赖

当前Chart依赖的其他Charts可以定义为 `dependencies` 字段中对应的一组元素。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

- `name` 字段表示依赖chart的名称
- `version` 字段表示依赖chart的版本
- `repository` 字段是Chart Repo完整url。注意，你必须手动使用 `helm repo add` 命令将其添加到本地repo源中。
- 此外，你有可能会使用repo的名称来代替它的完整url地址。

```console
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

当你完成了依赖关系的定义后，你可以使用 `helm dependency update` 命令将它们都下载到你的 `charts/` 目录中。

```console
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo https://example.com/charts
Downloading mysql from repo https://another.example.com/charts
```

通过执行上面的命令，我们应该可以看到在charts目录中存在如下文件：

```text
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

#### dependencies中的alias字段

除了上面描述的字段外，在每个依赖的chart中，可能都会包含一个可选字段 `alias`。

为依赖的图表添加alias后，在拉取依赖后，将会以别名的方式进行保存。
例如，我们可以通过alias的方式来访问对应的Charts：

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

在上面的示例中，我们可以看到在 `parentchart` 中存在3个依赖。

```text
subchart
new-subchart-1
new-subchart-2
```

手动获取依赖的方式是可以拉取原始Charts包，然后拷贝复制为不同的名称后保存到 `charts/` 目录下。

#### dependencies 中的 Tags 和 Condition 字段

除了上面提及的字段外，每个依赖chart可能还会包含可选字段 `tags` 和 `condition`。

所有的依赖charts默认情况下都会被加载。
但是如果增加了 `tags` 或 `condition` 字段后，它们将进行判断对应依赖charts是否需要被加载。

Condition：

Condition字段可以由一个或多个YAML PATY的字符串（用逗号分隔）组成。如果该YAML PATH对应的内容在父Values文件中存在且可以被解析成为一个布尔值，将根据布尔值的结果来判断是否启用该依赖。
如果对应的YAML PATH解析不存在，则该Contidion等价于没有设置，不会生效。

Tags:

Tag字段是Chart YAML中一组label组成的列表。
在顶层的父Value文件中，可以通过指定Tag对应的布尔值来批量启用/禁用相关的依赖charts。

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled, global.subchart1.enabled
    tags:
      - front-end
      - subchart1
  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

在上面的例子中，subchart1有一个tag `front-end`。
由于在 `value.yaml`文件中，front-end是被禁用的，但是对于 `subchart1.enabled` 而言，又被设置为了启用。
因此对于 `subchart1` 而言，最终还是被启用了。

由于 `subchart2` 打了 `back-end` Tag，同时该Tag被设置为了 `true`，因此，`subchart2` 将会被启用。
另外，需要注意的是尽量 `subchart2` 设置了一个Condition，但是由于没有传入对应的Values，相当于该condition不会生效。

##### 在cli中使用Tags 和 Condition

使用 `--set` 参数可以用于设置或改变 Tag 和 Condition 的值。

```console
helm install --set tags.front-end=true --set subchart2.enabled=false
```

##### Tags 和 Condition 解析

- **Conditions的优先级高于Tags**。存在多个Conditions时，以匹配的第一个Conditions为准，后续的Conditions会被忽略。
- Tag计算的方式是 如果Charts中的任意Tag为true，则启用该Charts。
- Tag和Condition的设置必须在父Chart的Value文件中。
- `tags:` 必须位于顶层key。目前不支持全局和嵌套 `tags`。

#### 通过依赖项导入子值

在某些情况下，希望允许子图表的值传播到父图表并作为通用默认值共享。
使用 `exports` 格式的另一个好处是，它将使将来的工具能够自动补全用户设置的值。

在charts文件中的 `dependencies` 列表中，`import-values`字段中包含的数据可以被导入到父charts中。
`import-values`字段中的每个数据项都是从 子charts的yaml文件 中的 `exports` 字段导出的。

如果想要导入 `exports` 中没有包含的key，那么需要使用 [child-parent](#using-the-child-parent-format) 的方式。
下面，我们将依次对上述两种场景给以示例进行说明。

##### 使用export的方式导入数据

如果一个子chart的 `values.yaml` 文件在根目录中包含一个 `exports` 字段，那么它就可以直接通过在父chart中进行import，从而导入到父Charts中。
如下所示：

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    import-values:
      - data
```

```yaml
# child's values.yaml file

exports:
  data:
    myint: 99
```

由于我们在import-values中指定了具体的key为 `data`，因此，Helm将会从子chart中找出对应的 `exports` 字段，并将其中的 `data` key的内容导入到父chart中。

最终，父Chart的Value中将会包含我们子charts中导出的指定字段。

```yaml
# parent's values

myint: 99
```

Ps：需要注意的是，导入数据后，之前的数据的key `data` 并没有包含在父chart的value中。
如果你想要把数据的key同时保存到父chart的value中，那么就需要了解下面的 'child-parent' 方式了。


##### 使用child-parent方式导入数据

如果想要获取在子chart中没有显式进行 `exports` 的字段，那么，你就需要指定导入的值的原始键（子chart中）以及导入到父Chart中的目标位置。

在下面的例子中，`import-values`告诉Helm需要找出 `child` 中指定的所有的value值并他们拷贝到 `parent` 中指定的父value的指定key下。

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

在上述例子中，在subchart1子Chart中找到的 `default.data` 的数据将会到导出到父Chart指定的 `myimports`的key中，如下所示：

```yaml
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```yaml
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

The parent chart's resulting values would be:

```yaml
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

最终，父chart中包含的 `myint` 和 `mybool` 字段将会被从 subchart1 子chart中导入的数据进行覆盖。








