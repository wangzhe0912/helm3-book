# 测试你的Charts

在一个Chart中，往往会包含一批K8s资源对象。
作为Chart的开发人员，我们可能通常需要编写一些测试Case验证我们的Chart在安装过程中是否可以正常工作。
同时，这些测试Case也可以帮助Chart的用户更好的理解如何使用这些Chart。

在Helm Chart中，测试Case是保存在 `templates/` 目录下，它们是一些Job对象并指定了要运行的命令和容器。
当测试运行成功时，容器的退出码预期应该为0。
这些Job对象必须包含Helm Test相关的注释，即：`helm.sh/hook: test`。

Ps：在Helm3之前，Job中需要包含Helm测试的Hook注释为 `helm.sh/hook: test-success` 或 `helm.sh/hook: test-failure`。
为了保证兼容，`helm.sh/hook: test-success` 在Helm3中保存了下来，等价于 `helm.sh/hook: test`。

测试示例:

- 验证你的values.yaml文件中配置是否可以正常进行替换。
  - 确保你的用户名和密码是正确的
  - 确保错误的用户名和密码不能正常工作
- 验证你的Service可以正常创建并且能够正常进行负载均衡。
- 等等。

你可以使用 `helm test <RELEASE_NAME>` 来针对一个release对应运行相关预定义的测试。
对于一个Chart用户而言，这是一个很好的方案来验证他们创建的release对象是否能够正常工作。

## 示例

下面是一个针对 [bitnami wordpress chart](https://hub.helm.sh/charts/bitnami/wordpress) 应用定义的Helm Test Pod。
如果你下载下来该Chart，你可以看到如下文件：

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm pull bitnami/wordpress --untar
```

```
wordpress/
  Chart.yaml
  README.md
  values.yaml
  charts/
  templates/
  templates/tests/test-mariadb-connection.yaml
```

`wordpress/templates/tests/test-mariadb-connection.yaml` 文件内容如下：

```yaml
{{- if .Values.mariadb.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-credentials-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: {{ .Release.Name }}-credentials-test
      image: {{ template "wordpress.image" . }}
      imagePullPolicy: {{ .Values.image.pullPolicy | quote }}
      {{- if .Values.securityContext.enabled }}
      securityContext:
        runAsUser: {{ .Values.securityContext.runAsUser }}
      {{- end }}
      env:
        - name: MARIADB_HOST
          value: {{ template "mariadb.fullname" . }}
        - name: MARIADB_PORT
          value: "3306"
        - name: WORDPRESS_DATABASE_NAME
          value: {{ default "" .Values.mariadb.db.name | quote }}
        - name: WORDPRESS_DATABASE_USER
          value: {{ default "" .Values.mariadb.db.user | quote }}
        - name: WORDPRESS_DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ template "mariadb.fullname" . }}
              key: mariadb-password
      command:
        - /bin/bash
        - -ec
        - |
          mysql --host=$MARIADB_HOST --port=$MARIADB_PORT --user=$WORDPRESS_DATABASE_USER --password=$WORDPRESS_DATABASE_PASSWORD
  restartPolicy: Never
{{- end }}
```

## 针对一个Release对象运行测试用例的步骤

首先，你需要在你的K8s集群中创建一个对应的release对象。
然后，你需要等待所有的Pod都可以正常运行。
Ps：如果你install命令完成后立马运行测试，此时可能会得到一些错误信息，需要等待所有Pod部署完成后再重新进行测试。

```
$ helm install quirky-walrus wordpress --namespace default
$ helm test quirky-walrus
Pod quirky-walrus-credentials-test pending
Pod quirky-walrus-credentials-test pending
Pod quirky-walrus-credentials-test pending
Pod quirky-walrus-credentials-test succeeded
Pod quirky-walrus-mariadb-test-dqas5 pending
Pod quirky-walrus-mariadb-test-dqas5 pending
Pod quirky-walrus-mariadb-test-dqas5 pending
Pod quirky-walrus-mariadb-test-dqas5 pending
Pod quirky-walrus-mariadb-test-dqas5 succeeded
NAME: quirky-walrus
LAST DEPLOYED: Mon Jun 22 17:24:31 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     quirky-walrus-mariadb-test-dqas5
Last Started:   Mon Jun 22 17:27:19 2020
Last Completed: Mon Jun 22 17:27:21 2020
Phase:          Succeeded
TEST SUITE:     quirky-walrus-credentials-test
Last Started:   Mon Jun 22 17:27:17 2020
Last Completed: Mon Jun 22 17:27:19 2020
Phase:          Succeeded
[...]
```

## 说明

- 在 `templates/` 目录下，你可以定义若干个yaml文件来包含你想要的全部的测试用例。
- 你也可以在 `templates/` 目录下创建一个子目录 `tests/` 目录来存放你的相关测试用例，例如： `<chart-name>/templates/tests/`。
- 每次测试Case其实都是一个[Helm hook](/docs/charts_hooks/)，所以你可以添加一些 `helm.sh/hook-weight` 或 `helm.sh/hook-delete-policy` 之类的注释。
