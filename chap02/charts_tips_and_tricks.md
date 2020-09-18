# Chart编写小技巧

本文涵盖了一些Helm Charts开发人员一些需要了解的提示和技巧，学习本文有助于开发人员制作出高质量的charts。

## 了解你的模板功能

Helm 使用 [Go templates](https://godoc.org/text/template) 来模板化资源文件。
尽管Go本身内置了一些函数，但是我们还要添加一些其他的函数。

我们添加的所有函数位于：[Sprig library](https://masterminds.github.io/sprig/).

我们还添加了两个模板函数: `include` and `required`. 
`include` 函数可以引入其他模板，并传递结果给其他模板函数。

例如，此模板代码段包含一个名为 "mytpl" 的模板，然后将内容小写化，最后用双引号引起来。

```yaml
value: {{ include "mytpl" . | lower | quote }}
```

`required` 函数可以声明某个字段在模板渲染中是必填的字段。
如果该字段为空，那么模板渲染将会失败并且返回用户对应的错误信息。

下面就是一个 `required` 函数的示例，它声明了 .Values.who 是必填的。
如果该字段为空，则会打印错误信息。

```yaml
value: {{ required "A valid .Values.who entry required!" .Values.who }}
```

## 字符串需要引号包围，数字不需要

当你在处理字符串的时候，你最好将所有的结果最后增加 | quote 进行引号包围。

```yaml
name: {{ .Values.MyName | quote }}
```

但是对于数字类型的变量而言，则不需要 | quote 进行引号包围，否则会导致解析错误。

```yaml
port: {{ .Values.Port }}
```

不过如果它们的格式预期就是字符串格式，那么也应该增加 | quote 进行转化。

```yaml
env:
  - name: HOST
    value: "http://host"
  - name: PORT
    value: "1234"
```

## 使用 include 函数

Go提供了一种使用内置模板指令将一个模板包含在另外一个模板中方法。
但是Go内置的函数不能用于Go模板管道。

为了能够支持模板包含和管道操作，Helm提供了一个 `include` 函数

```
{{ include "toYaml" $value | indent 2 }}
```

上述操作中，包含了一个名为 `toYaml` 的模板，传递给它一个 `$value`，并对它的输出执行了 `indent`函数操作。

由于YAML本身对缩进与空格非常的敏感，因此一种推荐的方式就是使用 include 引入代码段，再通过管道的方式处理缩进。 

## 使用 'required' 函数

Go提供了一种设置模板选项来控制当Map中没有找到对应Key行为的方式。
其中，典型的设置方式就是：`template.Options("missingkey=option")`。
其中， `option` 可以是 `default`, `zero`, or `error`。
当该 option 设置为error的时候，将会停止安装操作。
这种配置方式适用于Charts开发者希望用户必须在 `values.yaml` 中添加指定字段时使用。

`required` 函数能够使开发人员声明模板渲染的必填字段。
如果 `values.yaml` 中对应字段为空，则模板不会正常安装，并会返回开发人员提供的错误消息。

例如:

```
{{ required "A valid foo is required!" .Values.foo }}
```

只有当 `.Values.foo` 字段定义后，才能正常工作。否则，它将会渲染失败。

## 使用 'tpl' 函数

`tpl` 函数允许开发人员将模板的执行值用于模板中。
这适用于传递模板字符串的值给chart或者给外部配置文件。
语法格式：`{{ tpl TEMPLATE_STRING VALUES }}` 

例如：

```yaml
# values
template: "{{ .Values.name }}"
name: "Tom"

# template
{{ tpl .Values.template . }}

# output
Tom
```

渲染外部配置文件：

```yaml
# external configuration file conf/app.conf
firstName={{ .Values.firstName }}
lastName={{ .Values.lastName }}

# values
firstName: Peter
lastName: Parker

# template
{{ tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```

## 创建镜像拉取密钥

镜像拉取密钥是由镜像仓库、用户名和密码组成的。
你在部署你的应用的时候，可能需要用到这些信息，但是为了创建这些信息，你可能不得不多次进行base64编码等。

我们可以通过一个模板来接收Docker配置文件用于获得镜像拉取密钥，下面就是一个示例。

首先，假定授权信息是定义在 `values.yaml` 文件中：

```yaml
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
  email: someone@host.com
```

此时，我们的模板格式可以如下：

```
{{- define "imagePullSecret" }}
{{- with .Values.imageCredentials }}
{{- printf "{\"auths\":{\"%s\":{\"username\":\"%s\",\"password\":\"%s\",\"email\":\"%s\",\"auth\":\"%s\"}}}" .registry .username .password .email (printf "%s:%s" .username .password | b64enc) | b64enc }}
{{- end }}
{{- end }}
```

最后，我们可以在最终YAML的模板中引用我们自定义的模板来生成密钥信息。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

## 自动化滚动升级

很多时候，ConfigMap或Secrets会作为配置文件注入到容器中，或者有其他外部依赖项更改需要重启Pod。
而很多应用的重启本身其实需要依赖于 `helm upgrade` 命令来完成。
但是，如果deployment 的spec配置没有发生变化时，则应用程序将继续使用旧配置运行，从而导致部署不生效。

其中，`sha256sum` 函数可以用于保证当其他文件更新时，deployment的annotation部分一定会发生变化。

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

如果你想要每次都触发相关部署，你可以使用一种类似的方式，即在annotation中使用一个随机字符串。

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        rollme: {{ randAlphaNum 5 | quote }}
[...]
```

上述两种方式都可以保证你的应用在更新场景真正触发更新，而不会导致状态不一致的问题。

Ps：在Helm2中，我们使用  `--recreate-pods` 参数来保证每次更新，但是在Helm3中，我们移除了相关的参数，而是使用上述方式。

## Helm删除时保留部分资源

有时，我们在执行 `helm uninstall` 时，可能希望Helm能够保留部分资源对象不要删除。
Charts开发任务可以编写相关的annotation来说明资源需要进行保留。

```yaml
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
[...]
```

Ps: annotation的key需要用双引号包围起来。

annotation `"helm.sh/resource-policy": keep` 表示当进行helm操作，例如 `helm uninstall`, `helm upgrade` 以及 `helm rollback` 时，跳过删除资源的步骤。
但是，一旦设置了保留资源，这些资源将会被残留，不能再被helm进行管理。
这可能会导致在使用 `helm install --replace` 某个发布对象时，这个发布对象已经被删除了，但是资源还残留了下来。

## 使用 "Partials" 和 Template Includes

有时，你希望在你们chart中使用一些可复用的组件，它们可能时你的模板中的一部分。
通过提取可复用的组件，会让你的Charts文件看的更加清晰整洁。

在 `templates/` 目录中，所有以下划线 (`_`) 开头的文件都不希望直接针对K8s创建，而是一些内部模板。
例如，一些帮助模板等常常会放在 `_helpers.tpl` 文件中。

## 包含很多依赖的复杂Charts

在 [official charts repository](https://github.com/helm/charts) 中，有很多charts其实是完整charts的一部分，主要用于被其他一些复杂应用来引用的。
最终，它们会组成一个大规模的应用。在这种场景中，一个Charts可能同时还包含很多的子Charts，每个子Charts都是完整项目的一部分。

当前，针对该场景的最佳实践是创建一个顶层的伞状结构，然后使用 `charts/` 子目录来包含其他相关的charts组件。

## YAML是JSON的超集

根据YAML规范，YAML是JSON的超集。这意味着任何有效的JSON结构都可以用YAML结构进行表示。

尽管有时模板开发人员可能会发现，使用类似于JSON的语法来表示数据结构比处理YAML这种空格敏感的方式更加有效。

但是作为最佳实践，模板最好还是要使用YAML格式语法。

## 生成随机数的使用要小心

Helm中有一些函数可以帮助你生成随机数、加密密钥等等。
这些功能很好用，但是在使用过程中，也需要加以注意。

因为在升级过程中，模板将会被重新执行，当模板产生的数据与上次不一致时，将会触发资源对象的更新部署。

## 使用同一条命令进行 安装/升级 release对象。

Helm提供了一种方式使得 安装/升级 release对象都使用同一条命令。
那就是使用 `helm upgrade` 命令并增加 `--install` 命令。

它的效果是当release已经存在时，执行升级操作；否则执行新增操作。

```console
$ helm upgrade --install <release name> --values <values file> <chart directory>
```
