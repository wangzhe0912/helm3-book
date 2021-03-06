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

```
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

当你完成了依赖关系的定义后，你可以使用 `helm dependency update` 命令将它们都下载到你的 `charts/` 目录中。

```
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

为依赖的charts添加alias后，在拉取依赖后，将会以别名的方式进行保存。
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

```
helm install --set tags.front-end=true --set subchart2.enabled=false
```

##### Tags 和 Condition 解析

- **Conditions的优先级高于Tags**。存在多个Conditions时，以匹配的第一个Conditions为准，后续的Conditions会被忽略。
- Tag计算的方式是 如果Charts中的任意Tag为true，则启用该Charts。
- Tag和Condition的设置必须在父Chart的Value文件中。
- `tags:` 必须位于顶层key。目前不支持全局和嵌套 `tags`。

#### 通过依赖项导入子值

在某些情况下，希望允许子charts的值传播到父charts并作为通用默认值共享。
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


### 通过 `charts/` 目录手工管理Chart依赖项

如果需要对依赖项进行更加复杂的管理，那么也可以手动把对应的依赖Charts拷贝至 `charts/` 目录中来明确指明相关依赖。

依赖既可以是一个压缩包文件（`foo-1.2.3.tgz`），也可以是一个解压后的目录。
但是目录/文件名称不能以 `_` 或者 `.` 开头，否则这些文件将会被忽略。

例如，对于一个 WordPress Chart 而言，它会依赖一个 Apache 的Chart。
因此，我们可以将 Apache Chart放入到 WordPress的Chart中的 `charts/` 目录下即可。

```yaml
wordpress:
  Chart.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

上面的示例表示了对于一个WordPress的Chart，它依赖了一个Apache和一个MySQL的Chart。

**Ps**： 要将依赖项拉取至`charts/`目录，可以使用`helm pull`命令。

### 依赖项工作方式说明

上面的示例中说明了如何指定Chart之间的依赖关系，但是具体在 `helm install` 或 `helm upgrade` 操作时，这些依赖项是如何工作的呢？

假设一个名为A的Chart创建了以下Kubernetes对象：

1. namespace "A-Namespace"
2. statefulset "A-StatefulSet"
3. service "A-Service"

另外，A的一个依赖项B创建了如下Kubernetes对象：

1. namespace "B-Namespace"
2. replicaset "B-ReplicaSet"
3. service "B-Service"

在执行Chart A的安装和升级操作后会得到一个release对象，该release对象将会以如下顺序更新、创建上述所有的kubernetes对象：

1. A-Namespace
2. B-Namespace
3. A-Service
4. B-Service
5. B-ReplicaSet
6. A-StatefulSet

这是由于在安装、升级Chart的时候，Chart中的所有的Kubernetes对象及其所有的依赖项处理方式如下：

1. 聚合成一个集合。
2. 按照对象类型进行排序。
3. 按照排序后的顺序依次进行创建。

因此，最终Chart及其依赖Charts的所有的资源对象都会由一个release进行创建。

Kubernetes类型的安装顺序由kind_sorter.go中的枚举InstallOrder给出（请参阅[Helm源文件](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go)）。


## 模板和值

Helm的Chart模板是由 [Go template language](https://golang.org/pkg/text/template/) 为基础编写的。
另外还在[Sprig library](https://github.com/Masterminds/sprig)添加了50多个其他的模板函数。
同时还有一小部分[专用函数](../chap02/charts_tips_and_tricks.md)

所有的模板文件都存储在Chart包的 `templates/` 目录下。
在Helm渲染整个chart的时候，它将通过模板引擎处理该目录中的每个文件。

模板中的Values可以通过两种方式来提供：

1. Chart开发人员在Chart包内部提供一个 `values.yaml`的文件，这一文件可以用于包含相关Charts的默认值。 
2. Chart用户可以通过一个YAML文件来指定相关的值，或者也可以通过在 `helm install` 的命令行中传递该值。

当用户提供了自定义的值后，这些用户传递的自定义的值将会覆盖默认的 `values.yaml` 文件。

### 模板文件 

模板文件需要遵循用于编写Go模板的标准约定（参见[the text/template Go package documentation](https://golang.org/pkg/text/template/) ）。
示例如下：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

上面的示例是基于 [https://github.com/deis/charts](https://github.com/deis/charts) 项目。
它是用于描述一个K8s的ReplicationController对象。
它可以使用如下四个模板值进行渲染（通常会在`values.yaml`文件中定义）：

- `imageRegistry`: Docker镜像仓库.
- `dockerTag`: 镜像Tag.
- `pullPolicy`: 镜像拉取策略.
- `storage`: 数据存储后端, 默认值为 `"minio"`

所有的这些值都是由模板开发人员指定的，Helm并不明确进行相关参数名称等的规定。

如果想要查询更多的Charts的示例，可以查询：[Kubernetes Charts project](https://github.com/helm/charts)。

### 预定义文件

在模板文件中，可以通过 `.Values` 的方式来访问 `values.yaml` 或 `--set` 设置的 value的值。
除此之外，还有一些预定义变量也可以使得数据可以在模板中使用。

以下是所有的预定义变量，可用于每个模板，并且不能被覆盖。同时，与其他所有值一样，名称区分大小写。

- `Release.Name`: Release名称。
- `Release.Namespace`: Release所属的Namespace。
- `Release.Service`: 发布Release的服务。
- `Release.IsUpgrade`: 对于升级、回滚操作而言，该变量值为true。
- `Release.IsInstall`: 对于安装操作而言，该变量值为true。
- `Chart`: `Chart.yaml`的文件内容，因此，Chart Version可以表示为 `Chart.Version`，Chart 维护者可以表示为 `Chart.Maintainers`。
- `Files`: Chart中一个字典类型的对象，包含所有非特殊的文件。它无法让你访问模板文件，但是可以其他访问其他文件（除非使用`.helmignore`进行过滤）。
  你可以使用  \{\{ index .Files "file.name" \}\} 或者 \{\{ .Files.Get name \}\} 函数的方式来访问文件。
  此外，你还可以使用 \{\{  .Files.GetBytes \}\} 的方式读取文件中的内容作为 \[\]byte 格式。
- `Capabilities`: 一个字典类型的对象，包含K8s相关的版本信息 (\{\{ .Capabilities.KubeVersion \}\} 
  以及支持的K8s API 版本 (\{\{ .Capabilities.APIVersions.Has "batch/v1" \}\})

**PS**：任何未知的 `Chart.yaml` 字段信息将会被丢弃，他们无法在 Chart 对象中进行访问。因此，`Chart.yaml` 不能用于传递一些结构化数据到模板中。
如果存在相关的需求，可以使用Value文件。

### Value文件

以上述模板为例，一个包含必要字段的示例的 `values.yaml` 文件如下：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

上面的Yaml文件是YAML格式的。
一个Chart中可能会包含一个名为 `values.yaml` 的文件用于提供相关变量的默认值。
而在Helm安装Chart的过程中，还允许用户通过提供一个新的YAML文件来覆盖相关的默认值。

```
$ helm install --generate-name --values=myvals.yaml wordpress
```

当install过程中传递了相关变量时，这些变量值将会与 `value.yaml` 文件中的默认值进行合并。
例如，假设命令行中传入的 `myvals.yaml` 文件的内容如下：

```yaml
storage: "gcs"
```

此时，与 `values.yaml` 的内容进行合并后，得到的结果将会如下：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

如上所示，只有 `storage` 字段被进行了覆盖。

**注意**： Chart内的默认值文件名称必须是 `value.yaml`。但是在命令行中指定的文件名称可以自定义。

**注意**： 如果在`helm install` 和 `helm upgrade` 中使用 `--set` 时，这些参数实际是会在客户端转化为YAML格式进行传递的。

**注意**： 如果在value文件中有一些项是必填的，那么可以在模板中使用 `required` 进行声明，详见 ['required' function](../chap02/charts_tips_and_tricks.md)

values中传入的所有数据都可以在模板中通过 `.Values` 的方式进行读取：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

### Scope、Dependencies、Values

values文件可以在Chart的顶层目录中声明相关的value，同样也可以在 `charts/` 目录中包含的Charts中声明对应的value。
换句话说，value文件可以为当前的chart以及它所有依赖的charts提供对应的配置值。
例如，上面的演示WordPress charts同时具有mysql和apache作为依赖项。values文件可以为所有这些组件提供值：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

父目录的charts可以访问其定义的所有变量，因此，WordPress charts可以访问MySQL密码`.Values.mysql.password`。
但是子目录的charts无法访问父charts中的内容，因此MySQL将无法访问该`title`属性，同时，它也无法访问 `apache.port`。

Values文件中是包含命令空间的，同时在引用value的时候还会剪切到命名空间。
以WordPress chart为例，它可以通过`.Values.mysql.password`访问MySQL的password的字段。
但是对于MySQL chart而言，引用value中的配置时，则不需要带有`mysql`前缀了，可以直接简化为 `.Values.password`。

#### 全局Values

从2.0.0-Alpha.2开始，Helm支持了全局变量。考虑上一个示例的修改后的版本：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

在上面的实例中，包含了一个 `global` 块，包含 `app: MyWordPress`。
这个值可以被所有的charts通过 `.Values.global.app` 获取对应的值。

例如，`mysql`的模板中，可以使用如下方式进行访问 `{{ .Values.global.app}}`，`apache` chart 也是一样的。
实际上，上面的value文件相当于重新生成如下内容：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

这提供了一种与所有子charts共享一个顶级变量的方法，这对于诸如设置`metadata`标签之类的属性很有用。

如果子图声明了全局变量，则该全局变量将向下传递到子图的子图，而不向上传递到父图。即子charts无法影响父charts的值。

此外，父charts的全局变量优先于子图中的全局变量，即同时存在父charts和子charts拥有相同的全局变量值，会以父charts的全局变量为准。

### Schema文件

有时，Chart开发人员希望想要定义他们的value文件的结构格式。
此时，就会用到一个`values.schema.json`的schedule文件。
该schema称之为[JSON Schema](https://json-schema.org/)。
类似如下格式：

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

上述schema文件将会在执行如下命令时进行value值的合法性验证：

- `helm install`
- `helm upgrade`
- `helm lint`
- `helm template`

以一个 `values.yaml` 文件为例，该文件就是满足上述schema校验的。 

```yaml
name: frontend
protocol: https
port: 443
```

需要注意的是，该schema的生效范围是 `.Values` 对象，而不仅仅是 `values.yaml` 文件。
也就是说，即使 `yaml` 文件是满足 schema 校验的，但是如果 `--set` 参数传入的值不满足该schema校验，该请求也会被拦截。

```yaml
name: frontend
protocol: https
```

```
helm install --set port=443
```

另外，最终的 `.Values` 对象是需要经过所有的子Charts的schema检查的。
这也就意味着父Charts也不能规避对子Charts的约束。
换句话说，如果子Chart的 `value.yaml` 文件没有满足本身的约束，那么在父Chart中必须满足相关的限制后才能正常工作。

### 更多参考

在你编写模板文件、Value文件以及Schema文件时，如下一些官方文档可能会对你有一些帮助。

- [Go templates](https://godoc.org/text/template)
- [Extra template functions](https://godoc.org/github.com/Masterminds/sprig)
- [The YAML format](https://yaml.org/spec/)
- [JSON Schema](https://json-schema.org/)


## 自定义资源对象 (CRDs)

K8s提供了一种机制用于声明一些新的kubernetes对象类型。
使用自定义资源对象(CRDs)，Kubernetes开发成员可以自定义各种各样的资源类型。

在Helm3中，CRDs是被当做一种特殊的资源对象来处理的。
它们会在Chart中其他资源对象安装之前首先进行安装，同时收到一些相关的限制。

CRD YAML 文件应该位于Chart包中的 `crds/` 目录下。
多个CRDs的定义可能会存在于同一个文件中。
Helm会尝试加载 `crds/` 目录下的所有文件并在K8s中进行创建。

CRD 文件不能是模板文件，必须是直接可以使用的YAML文档。

当Helm安装一个新的Chart时，它第一步会首先创建所有的CRD，当CRD全部创建完成且API Server侧可用后，然后才会通过模板引擎渲染其他的Chart模板等，并发送给Kubernetes。
由于遵循着这一严格的顺序，因此在CRD相关的信息可以在Helm模板中的 `.Capabilities` 对象中获取相关信息，而在Helm模板中也可以创建一个自定义CRD类型的实例。

例如，如果你的Chart中 `crds/` 目录下存在一个 `CronTab` CRD，从而，你可以在 `templates/` 目录下创建 `CronTab` 类型的实例：

```text
crontabs/
  Chart.yaml
  crds/
    crontab.yaml
  templates/
    mycrontab.yaml
```

`crontab.yaml` 必须是一个CRD的标准YAML文件，而不是模板文件：

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

接下来的模板文件  `mycrontab.yaml` 中，就可以创建 `CronTab` 类型的实例了：

```yaml
apiVersion: stable.example.com
kind: CronTab
metadata:
  name: {{ .Values.name }}
spec:
   # ...
```

Helm会保证先创建 `CronTab` CRD对象，等待API Server中对应的CRD可用后，再去处理 `templates/` 下的相关模板。

### CRDs的相关限制

与K8s中其他大部分的对象不同，CRDs是全局性资源。因此，Helm在创建CRDs资源的时候非常谨慎，CRDs需要满足如下一系列限制：

- CRDs不会重复安装，如果Helm发现 `crds/` 目录下的CRDs已经在K8s中安装（无论版本是否一致）了，那么Helm都不会重新安装或者升级。
- CRDs不会进行升级和回滚操作，仅仅会执行安装操作。
- CRDs不会被删除，由于删除CRDs会导致所有Namespace下对应该CRDs类型的资源都被删除，因此，Helm不会去删除CRDs。

总之，如果想要升级、修改、删除CRDs时，需要手工进行处理。

## Chart Repositories

Chart Repositories本质上就是一个存放打包后的Charts的一个HTTP服务器。
`helm` 可以用于管理本地的Charts目录，如果想要与他人共享Charts时，就会首先想到Chart Repositories了。

任何一个HTTP服务器，只要能够存放YAML文件和Tar包，同时能够通过GET请求下载相关资源时，都可以作为一个repository Server。
Helm团队本身测试了一些Server，例如开启了Web模式的Google Cloud Storage与Amazon S3。

repository的主要特征时存在一个 `index.yaml` 文件，它包含着repository中存在的Package列表以及用于检索和验证Package的元数据。

在客户端，repositories是通过 `helm repo` 命令进行管理的。
然而，Helm本身并没有提供相关的工具用于上传charts到远程的repo中。
主要原因是相关的功能实现其实时需要服务端满足一些相关的功能，而这会对repo的搭建带来一定的障碍。

## Chart 初始化包

`helm create` 命令可以接收一个可选参数 `--starter` 用于指定一个初始化Chart。

它其实就一个普通的Chart，但是存放于 `$XDG_DATA_HOME/helm/starters`。
对于一个Charts开发者而言，你可以以它为基础来开发你自己的Charts。
在开发Charts时，需要注意如下点：

- `Chart.yaml` 文件会被生成器自动重写。
- Chart用户可能会去修改Charts的一些内容，因此，需要有相关的注释能够帮助用户快速理解该Chart。
- 所有 `<CHARTNAME>` 的部分最后都会被指定的Chart名称替换。

目前，唯一添加初始化包的方式就是手动的把Chart包拷贝到 `$XDG_DATA_HOME/helm/starters` 。
