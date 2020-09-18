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
