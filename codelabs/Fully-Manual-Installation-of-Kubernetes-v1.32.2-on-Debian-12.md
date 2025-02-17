summary: 在 Debian 12 上完全手动安装 kubernetes v1.32.2
id: Fully-Manual-Installation-of-Kubernetes-v1.32.2-on-Debian-12
categories: cloud
tags: cloud
status: Published
authors: panhy
Feedback Link: https://github.com/webtechwiki/codelabs/issues

# 在 Debian 12 上完全手动安装 kubernetes v1.32.2

## 一、基础环境准备

### 1.1 集群主机规划

所有机器的CPU都是x86的64位架构，并且安装了`Debian GNU/Linux 11 (bullseye)`。各个主机配置如下

|  主机   |       IP        |            操作系统            |              配置               |
| ------- | --------------- | ------------------------------ | ------------------------------- |
| k8s-101 | 192.168.122.101 | Debian GNU/Linux 12 (bookworm) | 内存:4G + SSD硬盘:30G + CPU:2核 |
| k8s-102 | 192.168.122.101 | Debian GNU/Linux 12 (bookworm) | 内存:4G + SSD硬盘:30G + CPU:2核 |
| k8s-103 | 192.168.122.101 | Debian GNU/Linux 12 (bookworm) | 内存:4G + SSD硬盘:30G + CPU:2核 |

- `199-debian`: etcd服务器、控制节点、Proxy的L4、L7代理。

同时作为运维主机，一些额外的服务由该主机提供，如：签发证书、dns服务、Docker的私有仓库服务、k8s资源配置清单仓库服务、共享存储（NFS）服务等。不过这些额外服务在需要的时候再安装，现在只是这么规划

- `192-debian`: etcd服务器、控制节点、Proxy的L4、L7代理、工作节点

- `192-debian`: etcd服务器、控制节点、工作节点

以上是在资源有限的情况下做的高可用资源分配，如果你的服务器资源充足，应当将各个服务分别部署到各个主机上，这样更加合理。

### 1.2 设置hostsname

在 `192.168.122.101` 执行以下命令

```bash
hostnamectl set-hostname k8s-101
cat >> /etc/hosts <<EOF
192.168.122.101 k8s-101
EOF
```

在 `192.168.122.102` 执行以下命令

```bash
hostnamectl set-hostname k8s-102
cat >> /etc/hosts <<EOF
192.168.122.102 k8s-102
EOF
```

在 `192.168.122.103` 执行以下命令

```bash
hostnamectl set-hostname k8s-103
cat >> /etc/hosts <<EOF
192.168.122.103 k8s-103
EOF
```

## 二、安装kubernetes

### 2.1 安装containerd

containerd的下载网址为<https://containerd.io/downloads/>，在撰写文章时（2025.02.15）最新版本是`v2.0.2`，安装到三台机器作为容器运行时环境，分别执行以下操作

#### 步骤 1：安装 containerd

从 <https://github.com/containerd/containerd/releases> 下载 `containerd-<版本>-<操作系统>-<架构>.tar.gz` 存档，验证其 sha256sum，并将其解压到 `/usr/local` 目录下

```shell
tar Cxzvf /usr/local containerd-2.0.2-linux-amd64.tar.gz
mkdir -p /usr/local/lib/systemd/system/
cat > /usr/local/lib/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now containerd
```

#### 步骤 2：安装 runc

runc 是一个轻量级的容器运行时工具，负责根据 OCI（Open Container Initiative）规范创建和运行容器。containerd 依赖 runc 来实际启动和管理容器。

从 <https://github.com/opencontainers/runc/releases> 下载 `runc.<架构>` 二进制文件，验证其 `sha256sum`，并将其安装为 `/usr/local/sbin/runc`。

```shell
install -m 755 runc.amd64 /usr/local/sbin/runc
```

该二进制文件是静态构建的，应该适用于任何 Linux 发行版。

#### 步骤 3：安装 CNI 插件

CNI（Container Network Interface）插件用于配置容器的网络，包括分配 IP 地址、设置网络接口、配置路由等。通常需要安装，除非明确不需要网络功能。

从 <https://github.com/containernetworking/plugins/releases> 下载 `cni-plugins-<操作系统>-<架构>-<版本>.tgz` 存档，验证其 `sha256sum`，并将其解压到 `/opt/cni/bin` 目录下：

```shell
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

这些二进制文件是静态构建的，应该适用于任何 Linux 发行版。

#### 步骤 4：使用命令行工具

`containerd` 是一个强大的容器运行时，但它本身是一个守护进程，需要通过命令行工具（CLI）来交互。不同的 CLI 工具（如 `ctr`、`nerdctl`、`crictl`）是为了满足不同用户和场景的需求而设计的。以下是它们的区别和适用场景：

|   工具    |           目标用户            |                                功能特点                                 |             适用场景             |
| --------- | ----------------------------- | ----------------------------------------------------------------------- | -------------------------------- |
| `ctr`     | `containerd` 开发者或高级用户 | `containerd` 自带的官方命令行工具，底层、简单、直接与 `containerd` 交互 | 开发和调试 `containerd`          |
| `nerdctl` | 普通用户和运维人员            | 类似 Docker 的体验，功能丰富                                            | 日常容器管理、生产环境           |
| `crictl`  | Kubernetes 管理员和开发者     | 针对 CRI 设计，适合 Kubernetes 环境                                     | 调试 Kubernetes 节点和容器运行时 |

在这里，我们额外安装 `nerdctl` 工具，以方便后续操作。在 <https://github.com/containerd/nerdctl/releases> 下载对应的操作系统版本，在撰写这边文章时 `nerdctl` 的版本是 `v2.0.3`，安装命令如下

```shell
wget https://github.com/containerd/nerdctl/releases/download/v2.0.3/nerdctl-2.0.3-linux-amd64.tar.gz
tar -zxvf nerdctl-2.0.3-linux-amd64.tar.gz -C /usr/bin/ nerdctl
```

最后，加载 `nerdctl` 的 `Bash` 自动补全功能，并设置 `containerd` 默认的名称空间为 `k8s.io`，如下

```bash
# 追加配置
cat >> /etc/profile <<EOF
source <(nerdctl completion bash)
export CONTAINERD_NAMESPACE=k8s.io
EOF

# 让配置立即生效
source /etc/profile
```

#### 步骤 5：配置 containerd

containerd默认配置文件在 `/etc/containerd/config.toml`，通过运行以下命令生成一个默认配置文件：

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

重启 containerd

```shell
systemctl restart containerd
```

### 2.2 签发SSL证书

`cfssl` 系列工具是 Cloudflare 提供的 PKI/TLS 工具，用于证书管理。可以在 <https://github.com/cloudflare/cfssl> 找到对应的信息，在撰写文章时版本是 `1.6.6`，我们下载对应操作系统的版本，安装到 `k8s-101` 这台主机，以 linux amd64 为例安装命令如下

```shell
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64 -o /usr/local/bin/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64 -o /usr/local/bin/cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64 -o /usr/local/bin/cfssl-certinfo
chmod a+x /usr/local/bin/cfssl*
```

以下是它们的简要功能：

- **cfssl**：核心工具，用于证书生成和管理。
- **cfssl-json**：辅助工具，用于解析 JSON 输出。
- **cfssl-certinfo**：用于查看证书详细信息。
