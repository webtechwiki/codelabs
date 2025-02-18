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

#### 2.1.1 安装 containerd

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

#### 2.1.2 安装 runc

runc 是一个轻量级的容器运行时工具，负责根据 OCI（Open Container Initiative）规范创建和运行容器。containerd 依赖 runc 来实际启动和管理容器。

从 <https://github.com/opencontainers/runc/releases> 下载 `runc.<架构>` 二进制文件，验证其 `sha256sum`，并将其安装为 `/usr/local/sbin/runc`。

```shell
install -m 755 runc.amd64 /usr/local/sbin/runc
```

该二进制文件是静态构建的，应该适用于任何 Linux 发行版。

#### 2.1.3 安装 CNI 插件

CNI（Container Network Interface）插件用于配置容器的网络，包括分配 IP 地址、设置网络接口、配置路由等。通常需要安装，除非明确不需要网络功能。

从 <https://github.com/containernetworking/plugins/releases> 下载 `cni-plugins-<操作系统>-<架构>-<版本>.tgz` 存档，验证其 `sha256sum`，并将其解压到 `/opt/cni/bin` 目录下：

```shell
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

这些二进制文件是静态构建的，应该适用于任何 Linux 发行版。

#### 2.1.4 使用命令行工具

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

#### 2.1.5 配置 containerd

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

#### 2.2.1 安装证书工具

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

#### 2.2.2 k8s所需证书概述

在Kubernetes集群中，我们需要为集群中的各个组件生成证书，以实现安全通信和身份验证。下图展示了Kubernetes所需的主要证书

![k8s证书](assets/202502/k8s_pki.png)

我们将在 `k8s-101` 生成的各个证书存放到 `/etc/kubernetes/pki` 里，并同步到其他主机上。

#### 2.2.3 搭建CA

CA是证书的签发机构，签发证书的前提是有一个签发机构，下文我们搭建自己的签发机构。

使用以下命令生成CA配置

```shell
mkdir -p /etc/kubernetes/pki
cat > /etc/kubernetes/pki/ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "175200h"
        },
        "profiles": {
            "www": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

使用以下命令生成 CA 请求文件

```shell
cat > /etc/kubernetes/pki/ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Guangzhou",
            "ST": "Guangdong",
            "O": "kubernetes",
            "OU": "system"
        }
    ],
    "ca": {
        "expiry": "175200h"
    }
}
EOF
```

证书根字段

- `CN`：证书名称
- `key`：定义证书类型，algo为加密类型，size为加密长度

`names` 定义证书的通用名称，可以有多个条目

- `CN`: Common Name，一般使用域名
- `C`: Country Code，申请单位所属国家，只能是两个字母的国家码。例如，中国只能是CN。
- `ST`: State or Province，省份名称或自治区名称
- `L`: Locality，城市或自治州名
- `O`: Organization name，组织名称、公司名称
- `OU`: Organization Unit Name，组织单位名称、公司部门

`ca.expiry` 代表有效时间，175200h代表20年。

最后使用以下命令生成CA自签名根证书

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

最后一个参数指定了证书文件名，最后生成以下三个文件

- `ca.csr`: 证书签名申请（Certificate Signing Request）文件
- `ca.pem`: ca公钥证书
- `ca-key.pem`: ca私钥证书

生成的三个文件是根证书包含的内容。后续，我们给各个服务颁发证书的时候，都基于CA根证书来颁发。

#### 2.2.4 签发证书

- etcd

定义证书信息如下

```shell
cat > /etc/kubernetes/pki/etcd-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "192.168.122.101",
        "192.168.122.102",
        "192.168.122.103"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "kubernetes",
        "OU": "system"
    }]
}
EOF
```

支持的主机列表对应本机以及所有 etcd 节点。使用以下命令生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www etcd-csr.json | cfssljson -bare etcd
```

- kube-apiserver

k8s的其他组件跟 apiserver 要进行双向TLS（mTLS）认证，所以 apiserver 需要有自己的证书，以下定义证书申请文件

```shell
cat > /etc/kubernetes/pki/apiserver-csr.json <<EOF
{
    "CN": "apiserver",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
        "127.0.0.1",
        "10.96.0.1",
        "192.168.122.101",
        "192.168.122.102",
        "192.168.122.103",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "names": [{
        "C": "CN",
        "ST": "Guangdong",
        "L": "Guangzhou",
        "O": "kubernetes",
        "OU": "system"
    }]
}
EOF
```

该证书后续被 kubernetes master 集群使用，需要将 master 节点的 IP 都填上，同时还需要填写 service 网络的第一个IP（后续计划使用`10.96.0.0 255.255.0.0` 网段作为service网络，因此加上 `10.96.0.1`），后续可能加到集群里的IP也需要都填写上去。最后使用以下命令生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www apiserver-csr.json | cfssljson -bare apiserver
```

-- kube-controller-manager

controller-manager需要跟apiserver进行mTLS认证，定义证书申请文件如下

```shell
cat > /etc/kubernetes/pki/controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
   "hosts": [
     "127.0.0.1",
     "192.168.122.101",
     "192.168.122.102",
     "192.168.122.103"
    ],
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Guangzhou",
      "O": "system:kube-controller-manager",
      "OU": "system"
    }
  ]
}
EOF
```

hosts 列表包含所有 kube-controller-manager 节点 IP；CN 为 system:kube-controller-manager，O 为 system:kube-controller-manager，k8s里内置的ClusterRoleBindings system:kube-controller-manager 授权 kube-controller-manager所需的权限。后面组件证书都做类似操作。生成证书命令如下

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www controller-manager-csr.json | cfssljson -bare controller-manager
```

- kube-scheduler

kube-scheduler需要跟apiserver进行mTLS认证，生成证书申请文件如下

```shell
cat > /etc/kubernetes/pki/scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
   "hosts": [
     "127.0.0.1",
     "192.168.122.101",
     "192.168.122.102",
     "192.168.122.101"
    ],
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Guangzhou",
      "O": "system:kube-scheduler",
      "OU": "system"
    }
  ]
}
EOF
```

kubernetes内置的ClusterRoleBindings system:kube-scheduler将授权kube-scheduler所需的权限。生成证书命令如下

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www scheduler-csr.json | cfssljson -bare scheduler
```

- kube-proxy

kube-proxy需要跟apiserver进行mTLS认证，生成证书申请请求文件如下

```shell
cat > /etc/kubernetes/pki/proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangdong",
      "L": "Guangzhou",
      "O": "kubernetes",
      "OU": "system"
    }
  ]
}
EOF
```
生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www proxy-csr.json | cfssljson -bare proxy
```

- 管理员admin能用的证书

创建证书信息

```shell
cat > /etc/kubernetes/pki/admin-csr.json << EOF 
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Guangzhou",
      "L": "Guangdong",
      "O": "system:masters",
      "OU": "system"
    }
  ]
}
EOF
```

生成证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www admin-csr.json | cfssljson -bare admin
```

#### 2.2.5 同步证书

生成证书之后，将证书目录`/etc/kubernetes/pki`同步到其他主机。
