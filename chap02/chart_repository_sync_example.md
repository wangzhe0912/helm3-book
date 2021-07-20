# 同步本地和远程Charts存储库

本文以 Google Cloud Storage (GCS) 块存储为例讲解如何进行 Chart Repo同步。

## 准备工作
* 安装 [gsutil](https://cloud.google.com/storage/docs/gsutil) 工具。
* helm二进制工具能够正常使用

## 创建本地Chart Repo目录

[创建一个本地目录](../chap03/chart_repository.md) 然后将你的charts包放到该目录下。

例如：

```
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
```

## 生成并更新index.yaml文件

传递工作目录和远程Repo地址作为参数使用helm来创建并更新 index.yaml 文件，示例如下：

```
$ helm repo index fantastic-charts/ --url https://fantastic-charts.storage.googleapis.com
```

上述命令将会生成并修改一个index.yaml文件并将其放置在 `fantastic-charts/` 目录下。

## 同步你的本地与远程repo

运行  `scripts/sync-repo.sh` 命令并传递本地目录名称和GCS块名称作为参数，来上传目录内容到GCS块中。

例如：

```
$ pwd
/Users/me/code/go/src/helm.sh/helm
$ scripts/sync-repo.sh fantastic-charts/ fantastic-charts
Getting ready to sync your local directory (fantastic-charts/) to a remote repository at gs://fantastic-charts
Verifying Prerequisites....
Thumbs up! Looks like you have gsutil. Let's continue.
Building synchronization state...
Starting synchronization
Would copy file://fantastic-charts/alpine-0.1.0.tgz to gs://fantastic-charts/alpine-0.1.0.tgz
Would copy file://fantastic-charts/index.yaml to gs://fantastic-charts/index.yaml
Are you sure you would like to continue with these changes?? [y/N]} y
Building synchronization state...
Starting synchronization
Copying file://fantastic-charts/alpine-0.1.0.tgz [Content-Type=application/x-tar]...
Uploading   gs://fantastic-charts/alpine-0.1.0.tgz:              740 B/740 B
Copying file://fantastic-charts/index.yaml [Content-Type=application/octet-stream]...
Uploading   gs://fantastic-charts/index.yaml:                    347 B/347 B
Congratulations your remote chart repository now matches the contents of fantastic-charts/
```

## 更新你的Chart Repo

你可能想要保存本地Chart Repo的内容，同时想要使用 `gsutil rsync` 来将远程Chart Repo内容同步到本地目录。

例如:
```
$ gsutil rsync -d -n gs://bucket-name local-dir/    # the -n flag does a dry run
Building synchronization state...
Starting synchronization
Would copy gs://bucket-name/alpine-0.1.0.tgz to file://local-dir/alpine-0.1.0.tgz
Would copy gs://bucket-name/index.yaml to file://local-dir/index.yaml

$ gsutil rsync -d gs://bucket-name local-dir/       # performs the copy actions
Building synchronization state...
Starting synchronization
Copying gs://bucket-name/alpine-0.1.0.tgz...
Downloading file://local-dir/alpine-0.1.0.tgz:                        740 B/740 B
Copying gs://bucket-name/index.yaml...
Downloading file://local-dir/index.yaml:                              346 B/346 B
```

参考链接:

* [gsutil rsync](https://cloud.google.com/storage/docs/gsutil/commands/rsync#description) 文档
* [The Chart Repository Guide](../chap03/chart_repository.md)
