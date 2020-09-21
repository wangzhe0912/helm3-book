# Chart Hooks详解

Helm提供了一个 Hook 的机制可以使得Charts开发者能够在release生命周期的合适时机进行干预。
例如，你可以使用hooks进行：

- 在其他Charts安装之前首先创建ConfigMap和Secret。
- 在安装一个新的Chart之前，首先执行一个Job用于备份数据库，然后在升级完成后执行第二个Job用户恢复数据。
- 在删除release对象之前，运行一个Job用于使得服务能够优雅退出。

Hooks工作方式与一般的模板类似，但是它们可以使用一些特殊的标注从而可以让Helm对它们进行特殊处理。
在本节中，我们将会讲解Hook模式的一些基本使用方式：

## 可用的Hooks

Helm内置定义了如下的Hooks:

| 注释名称          | 功能说明                                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------------------- |
| `pre-install`    | 执行时机: 模板渲染完成，但是在向K8s提交之前                                                                 |
| `post-install`   | 执行时机: 所有资源对象都已经提交给K8s之后                                                                   |
| `pre-delete`     | 执行时机: 对于delete请求，在请求发送给K8s之前                                                               |
| `post-delete`    | 执行时机: 对于delete请求，在请求发送给K8s之后                                                               |
| `pre-upgrade`    | 执行时机: 对于升级请求，模板渲染完成，但是在向K8s提交之前                                                      |
| `post-upgrade`   | 执行时机: 对于升级请求，模板渲染完成，并提交K8s之后                                                           |
| `pre-rollback`   | 执行时机: 对于回滚请求，模板渲染完成，但是在向K8s提交之前                                                      |
| `post-rollback`  | 执行时机: 对于回滚请求，模板渲染完成，并提交K8s之后                                                           |
| `test`           | 执行时机: 当执行helm test子命令时被调用 ([view test docs](/docs/chart_tests/))                             |

Ps: 在helm3中，`crd-install` hook已经被移除。


## Hooks 与 the Release 的生命周期

Hooks可以帮助Chart开发者在release声明周期中定义一些策略，从而可以直接一些相关的操作。
以 `helm install` 的声明周期为例：

1. 用户运行 `helm install foo` 命令
2. Helm库中install API被调用
3. 在完成基本的参数验证后，渲染 `foo` 的模板
4. 将相关的资源对象信息发送给kubernetes
5. lib库返回release对象或其他数据给客户端
6. 客户端退出

Helm定义了两个hooks在 `install` 操作的生命周期中，分别是: `pre-install` 和 `post-install`。
如果开发者同时实现了 `foo` chart的这两个hooks，那么整体生命周期的变为了如下方式：

1. 用户运行 `helm install foo` 命令
2. Helm库中install API被调用
3. `crds/` 目录下的相关CRDs被安装到K8s中
4. 在完成基本的参数验证后，渲染 `foo` 的模板
5. lib库准备执行 `pre-install` hooks（加载hooks资源到k8s中）
6. lib库对hooks按照权重（默认权重为0）、资源类型、名称进行升序排序
7. 根据排序逆序执行相关的hooks
8. 等待Hook的状态变为Ready（CRDs除外）
9. 提交渲染后的资源对象给K8s。Ps，如果设置了 `--wait` 参数，则Lib库将会等待所有的资源对象状态变为Ready再进行下一步。
10. 执行`post-install` hook （加载hooks资源到k8s中）
11. Lib库等待 hook 状态变为Ready。
12. lib库返回release对象或其他数据给客户端。
13. 客户端退出。

等待Hook执行完成的含义是什么呢？它取决于Hook中定义的资源对象的类型:

如果资源类型是`Job` 或者 `Pod`，那么Helm将会等待这个任务成功运行完成。
如果Hook运行失败，那么这个Release对象也会创建失败。
这是一个阻塞性操作，在Job运行过程中，helm客户端会进行阻塞性等待。

对于其他所有类型的资源而言，只要K8s成功加载了对应的资源，Helm就认为该资源的状态已经是 "Ready" 了。
当Hook中定义了很多资源对象时，这些资源对象互相之间是串行执行的。
如果对它们定义了不同的权重，那么它们将会按照权重大小依次执行。
从Helm 3.2.0版本开始，拥有相同权重的hook资源与普通资源执行的顺序是一致的，只有拥有不同权重的hook资源才能保证执行顺序。
所以，如果想要定义执行顺序，那么最好的方法就是主动设置相关的权重，如果不设置权重的话，默认为0，可以认为对顺序无关。

### Hook资源不受release对象控制

Hook场景的资源目前并没有作为release对象的一部分进行跟踪。
一旦Helm验证这个Hook达到了Ready状态后，它将会让Hook资源自行进行运行。
在Helm3的未来的版本中，一旦release被删除后，对应的hook资源将会被进行垃圾回收。
因此，如果有哪些Hook资源你是希望进行永久保留的，那么就需要增加如下注释：`helm.sh/resource-policy: keep`。

实际上，这也就意味着如果你在hook中创建了一些资源，你无法通过 `helm uninstall` 来删除这些资源。
为了删除这些资源，一方面你可以在Hook模板文件中添加 [ `helm.sh/hook-delete-policy` 注释](#hook-deletion-policies)，
另一方面也可以 [设置资源的生效时间](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/).

## 编写一个Hook

Hooks本质上就是一些带有特殊注释的K8s yaml文件。由于它们也属于template文件，因此它们也可以使用Template中一些基本功能，例如 `.Values`, `.Release` 以及 `.Template`。

例如，对于 `templates/post-install-job.yaml` 这个template文件而言，定义了一个Job在 `post-install` 时运行。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep", "{{ default "10" .Values.sleepyTime }}"]
```

决定该template能够作为hooks的关键点是如下annotation:

```yaml
annotations:
  "helm.sh/hook": post-install
```

同一个资源可以同时作为多个hooks:

```yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
```

类似的事，对于某一个Hook场景，可以存在多个不同的资源对象。
例如，我们可以定义一个secret和一个ConfigMap都是 pre-install 的Hook。

当子Chart中定义了Hooks时，这些Hooks也都会被计算和执行。
目前还不支持在父Chart中禁用子Charts中定义的Hooks。

我们可以定义 Hooks 的权重，从而能够保证多个Hooks时按照指定的顺序执行。
权重的定义方式如下：

```yaml
annotations:
  "helm.sh/hook-weight": "5"
```

Hook的权重可以是正数、也可以是负数，需要以字符串的形式表示。
Helm在执行特定场景的Hooks时，会对这些Hooks的权重进行排序，并从大到小依次执行。

### Hook 删除策略

我们可以定义 Hook资源的删除策略。
Hook资源删除策略的annotation的关键词如下：

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

你可以选择一个或多个定义好的annotation值：

| Annotation Value       | Description                             |
| ---------------------- | ----------------------------------------|
| `before-hook-creation` | 在新Hook运行时，删除之前的Hook资源 (default)|
| `hook-succeeded`       | 当Hook资源Ready后，删除对应的Hook资源       |
| `hook-failed`          | 当Hook资源执行失败后，删除对应的Hook资源     |

如果没有显式指定Hook的删除策略，那么默认策略为 `before-hook-creation`。
