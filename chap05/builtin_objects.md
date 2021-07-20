# Chart 中的内置变量

在 helm 中，变量会从模板引擎传递到模板中，
我们也可以编写代码来传递变量甚至创建新的变量。

一个变量可以仅仅是一个简单的值，也可以是一个嵌套的对象。
例如，对于 `Release` 变量而言，它还包含了 `Name` 属性等。

在上一节中，我们用 `{{ .Release.Name }}` 在模板中插入版本名称。
Release是你可以在模板中访问的高级对象之一。

## Release 变量

下面，我们就来具体看一下 helm 中 Release 变量具体包含了哪些属性吧。

 - Release： 该变量描述了版本发布本身。包含了以下属性：
   - Release.Name： release名称
   - Release.Namespace： 版本中包含的命名空间(如果manifest没有覆盖的话)
   - Release.IsUpgrade： 如果当前操作是升级或回滚的话，该值为true
   - Release.IsInstall： 如果当前操作是安装的话，该值为true
   - Release.Revision： 此次修订的版本号。安装时是1，每次升级或回滚都会自增
   - Release.Service： 该service用来渲染当前模板。一般是Helm。


## Values 变量

Values是从values.yaml文件和用户提供的文件传进模板的。Values默认为空。

## Chart 变量

`Chart.yaml`文件内容。
Chart.yaml里的任意数据在这里都可以可访问的。比如 `{{ .Chart.Name }}-{{ .Chart.Version }}` 会打印出 mychart-0.1.0。

## Files 变量

在chart中允许访问所有的非特殊文件。
当你不能使用它访问模板时，你可以访问其他文件。 

 - Files.Get 通过文件名获取文件的方法。 （.Files.Getconfig.ini）
 - Files.GetBytes 用字节数组代替字符串获取文件内容的方法。 对图片之类的文件很有用。
 - Files.Glob 用给定的shell glob模式匹配文件名返回文件列表的方法。
 - Files.Lines 逐行读取文件内容的方法。迭代文件中每一行时很有用。
 - Files.AsSecrets 使用Base 64编码字符串返回文件体的方法。
 - Files.AsConfig 使用YAML格式返回文件体的方法。

## Capabilities

Capabilities 提供关于Kubernetes集群支持功能的信息。

 - Capabilities.APIVersions 是一个版本集合。
 - Capabilities.APIVersions.Has $version 说明集群中的版本 (e.g., batch/v1) 或是资源 (e.g., apps/v1/Deployment) 是否可用。
 - Capabilities.KubeVersion 和 Capabilities.KubeVersion.Version 是Kubernetes的版本号。
 - Capabilities.KubeVersion.Major Kubernetes的主版本。
 - Capabilities.APIVersions.KubeVersion.Minor Kubernetes的次版本。

## Template

Template 包含了已经被执行的当前模板信息。

 - Template.Name: 当前模板的命名空间文件路径 (e.g. mychart/templates/mytemplate.yaml)。
 - Template.BasePath: 当前chart模板目录的路径 (e.g. mychart/templates)


## Summary

helm 中所有内置的变量都是以大写字母开始，这符合 Go 语言的命令惯例。

当你创建自己的变量时，可以按照团队约定自由设置。例如，可以将自定义变量全部设置为小写字母开头，从而可以和内置变量进行区分。
