# Helm安装

本文主要讲解如何安装Helm CLI工具。
Helm可以通过源代码安装和二进制包安装等多种方式进行。


Helm项目官方提供了两种方式来安装Helm。
此外，Helm的社区来提供了通过不同包管理器的方式安装Helm。
接下来，我们将会依次介绍官方提供的方式和包管理器安装的方式。


## Helm原生安装

### 二进制发布包安装

Helm的二进制包发布版本地址 [release](https://github.com/helm/helm/releases)，它提供了各种操作系统下的二进制包。
这些二进制包可以直接进行手动下载和安装。

1. 下载你想要的Helm版本的二进制包
2. 解压二进制包 (`tar -zxvf helm-v3.0.0-linux-amd64.tar.gz`)
3. 在解压目录下找到helm二进制文件，并把它移动到bin目录下即可。 (`mv linux-amd64/helm /usr/local/bin/helm`)

接下来，你就可以 [添加repo源](./quickstart): `helm help`.

### 脚本安装

目前，helm有一个一键安装脚本可以自动化的下载并[安装最新版本的helm](https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3)。
你可以下载该脚本并手动执行。

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

同时，如果你想要一条命令搞定安装的任务，可以执行如下脚本：

`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

## 包管理器安装

Helm社区提供了各种操作系统下的包管理器的安装方式。
这些安装方式都不是Helm项目自身维护的，而是一些第三方伙伴开发和维护的。

### Homebrew(macOS系统安装)

```
brew install helm
```

### Chocolatey(Windows安装)

[Helm package](https://chocolatey.org/packages/kubernetes-helm) 是由 [Chocolatey](https://chocolatey.org/) 维护的。

```
choco install kubernetes-helm
```

### Apt(Debian/Ubuntu安装)

[Helm package](https://helm.baltorepo.com/stable/debian/) for Apt

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Snap

[Snapcrafters](https://github.com/snapcrafters) 社区维护了 [Helm package](https://snapcraft.io/helm):

```
sudo snap install helm --classic
```

### 开发版本安装

除了发布版本外，你还可以下载和安装Helm的开发版本。

### 金丝雀版本安装

金丝雀版本是由Helm的Master分支最新的代码产出的版本，它们没有经过完整的测试，可能会不太稳定，但是可以帮助用户了解Helm的最新功能。

Helm金丝雀版本产出存在 [get.helm.sh](https://get.helm.sh)。下面是一些常用平台的编译产出地址。

- [Linux AMD64](https://get.helm.sh/helm-canary-linux-amd64.tar.gz)
- [macOS AMD64](https://get.helm.sh/helm-canary-darwin-amd64.tar.gz)
- [Experimental Windows AMD64](https://get.helm.sh/helm-canary-windows-amd64.zip)

### 源码安装(Linux, macOS)

从源码构建helm的话需要稍微做一些工作，但是这是你想要测试最新版本helm的一个最好的方式。

首先，你需要搭建Go语言的开发环境。

```
$ git clone https://github.com/helm/helm.git
$ cd helm
$ make
```

此外，你还需要拉取相关的Go依赖库并且验证依赖是否完整。
当编译产出新的helm后，同样需要将他放到bin目录下。

## 总结

通常来说，安装helm二进制包是非常简单的。本文中也涵盖了大部分Helm安装的场景。

当你完成了Helm的安装后，下面你就可以用helm来进行[Charts管理和操作](./quickstart.md)了。
