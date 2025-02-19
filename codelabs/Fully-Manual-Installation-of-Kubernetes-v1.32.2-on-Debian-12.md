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

- `k8s-101`: etcd服务器、控制节点、Proxy的L4、L7代理。

同时作为运维主机，一些额外的服务由该主机提供，如：签发证书、dns服务、Docker的私有仓库服务、k8s资源配置清单仓库服务、共享存储（NFS）服务等。不过这些额外服务在需要的时候再安装，现在只是这么规划

- `k8s-102`: k8s-103
- `k8s-102`: k8s-103
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

## 二、安装containerd

containerd的下载网址为<https://containerd.io/downloads/>，在撰写文章时（2025.02.15）最新版本是`v2.0.2`，安装到三台机器作为容器运行时环境，分别执行以下操作

### 2.1 安装 containerd

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

### 2.2 安装 runc

runc 是一个轻量级的容器运行时工具，负责根据 OCI（Open Container Initiative）规范创建和运行容器。containerd 依赖 runc 来实际启动和管理容器。

从 <https://github.com/opencontainers/runc/releases> 下载 `runc.<架构>` 二进制文件，验证其 `sha256sum`，并将其安装为 `/usr/local/sbin/runc`。

```shell
install -m 755 runc.amd64 /usr/local/sbin/runc
```

该二进制文件是静态构建的，应该适用于任何 Linux 发行版。

### 2.3 安装 CNI 插件

CNI（Container Network Interface）插件用于配置容器的网络，包括分配 IP 地址、设置网络接口、配置路由等。通常需要安装，除非明确不需要网络功能。

从 <https://github.com/containernetworking/plugins/releases> 下载 `cni-plugins-<操作系统>-<架构>-<版本>.tgz` 存档，验证其 `sha256sum`，并将其解压到 `/opt/cni/bin` 目录下：

```shell
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

这些二进制文件是静态构建的，应该适用于任何 Linux 发行版。

### 2.4 使用命令行工具

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

### 2.5 配置 containerd

containerd默认配置文件在 `/etc/containerd/config.toml`，通过运行以下命令生成一个默认配置文件：

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

重启 containerd

```shell
systemctl restart containerd
```

## 三、签发SSL证书

### 3.1 安装证书工具

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

### 3.2 k8s所需证书概述

在Kubernetes集群中，我们需要为集群中的各个组件生成证书，以实现安全通信和身份验证。下图展示了Kubernetes所需的主要证书

![k8s证书](assets/202502/k8s_pki.png)

我们将在 `k8s-101` 生成的各个证书存放到 `/etc/kubernetes/pki` 里，并同步到其他主机上。

### 3.3 搭建CA

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

### 3.4 签发证书

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

### 3.5 同步证书

生成证书之后，将证书目录`/etc/kubernetes/pki`同步到其他主机。

## 四、安装etcd

我们将使用`k8s-101`、`k8s-102`、`k8s-103`这三台主机搭建ectd集群。在撰写此文档时（2425.02.18），etcd最新稳定版本是 `3.5.18`，可以从 <https://github.com/etcd-io/etcd/releases/> 这个链接下载对应的安装包。

### 4.1 etcd启动参数

常见参数说明说下

|             参数              |           对应环境变量           |                                           说明                                            |
| ----------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------- |
| --name                        | ETCD_NAME                        | 当前etcd的唯一名称，要保证和其他节点不冲突                                                |
| --data-dir                    | ETCD_DATA_DIR                    | 指定etcd存储数据的存储位置                                                                |
| --listen-peer-urls            | ETCD_LISTEN_PEER_URLS            | 端对端的通信url，包含主机地址和端口号，指定当前etcd和其他节点etcd通信时的服务地址和端口。 |
| --listen-client-urls          | ETCD_LISTEN_CLIENT_URLS          | 指定当前etcd接收客户端指令的地址和端口，在这里的客户端我们指的是k8s集群的master节点       |
| --initial-advertise-peer-urls | ETCD_INITIAL_ADVERTISE_PEER_URLS | 指定etcd广播端口，当前etcd会将数据同步到其他节点，通过2380端口发送                        |
| --advertise-client-urls       | ETCD_ADVERTISE_CLIENT_URLS       | 给客户端通告的端口                                                                        |
| --initial-cluster             | ETCD_INITIAL_CLUSTER             | 定义etcd集群中所有节点的名称和IP，以及通信端口                                            |
| --initial-cluster-token       | ETCD_INITIAL_CLUSTER_TOKEN       | 定义etcd中的token，所有节点的token必须保持一致                                            |
| --initial-cluster-state       | ETCD_INITIAL_CLUSTER_STATE       | 定义etcd集群的状态，new代表新建集群，existing代表加入现有集群                             |

### 4.2 创建数据目录

先创建etcd默认的配置文件目录和数据目录

```bash
mkdir -p /var/lib/etcd/
```

安装到`/opt`目录，后续的k8s集群组件我们将都安装在此

```shell
# 解压
tar -zxvf etcd-v3.5.18-linux-amd64.tar.gz
# 将etc移到/opt目录，并修改etcd目录名
mv etcd-v3.5.18-linux-amd64/ /opt/etcd-v3.5.18
# 创建etcd软链接
ln -s /opt/etcd-v3.5.18 /opt/etcd
```

### 4.3 创建etcd启动脚本

我们先在etcd目录编写启动脚本`/opt/etcd/startup.sh`，如下

```shell
#!/bin/bash
./etcd \
  --name="etcd-server-101" \
  --data-dir="/var/lib/etcd/" \
  --listen-peer-urls="https://192.168.122.101:2380" \
  --listen-client-urls="https://192.168.122.101:2379,http://127.0.0.1:2379" \
  --initial-advertise-peer-urls="https://192.168.122.101:2380" \
  --advertise-client-urls="https://192.168.122.101:2379" \
  --initial-cluster="etcd-server-101=https://192.168.122.101:2380,etcd-server-102=https://192.168.122.102:2380,etcd-server-103=https://192.168.122.103:2380" \
  --initial-cluster-token="etcd-cluster" \
  --initial-cluster-state="new" \
  --cert-file="/etc/kubernetes/pki/etcd.pem" \
  --key-file="/etc/kubernetes/pki/etcd-key.pem" \
  --trusted-ca-file="/etc/kubernetes/pki/ca.pem" \
  --peer-cert-file="/etc/kubernetes/pki/etcd.pem" \
  --peer-key-file="/etc/kubernetes/pki/etcd-key.pem" \
  --peer-trusted-ca-file="/etc/kubernetes/pki/ca.pem" \
  --peer-client-cert-auth \
  --client-cert-auth
```

给启动脚本添加权限

```shell
chmod +x /opt/etcd/startup.sh
```

### 4.4 使用supervisor来启动etcd

现在我们要安装supervisor，用于管理etcd服务，后续的k8s相关组件，我们都用supervisor来管理

```shell
# 安装supervisor
apt install supervisor -y
# 启动supervisor
systemctl start supervisor
# 让superivisor开机自启
systemctl enable supervisor
```

添加etcd的supervisor进程维护脚本`/etc/supervisor/conf.d/etcd-server.conf`，添加以下内容

```ini
[program:etcd-server-101]
directory=/opt/etcd
command=/opt/etcd/startup.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/supervisor/etcd.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

> 注意：在不同的主机上使用不同的服务名称，这样好辨别，如k8s-101使用`etcd-server-101`，如k8s-102使用`etcd-server-102`

supervisor相关参数：

- `program`: 程序名称
- `directory`: 脚本目录
- `command`: 启动的命令
- `numprocs`: 启动的进程数
- `autostart`: 是否开启自动启动
- `autorestart`: 是否自动重启
- `startsecs`: 启动之后多少时间后判定为已起来
- `startretries`: 重启次数
- `exitcodes`: 退出的code
- `stopsignal`: 停止信号
- `stopwaitsecs`: 停止等待的时间
- `redirect_stderr`: 是否重定向标准输出
- `stdout_logfile`: 进程标准输出内容写入文件
- `stdout_logfile_maxbytes`: stdout_logfile文件做log滚动时，单个stdout_logfile文件的最大字节数，默认50M，设置为0则认为不做log滚动方式
- `stdout_logfile_backups`: stdout_logfile备份文件个数，默认为10
- `stdout_capture_maxbytes`: 当进程处于stdout capture mode模式的时候，写入capture FIFO的最大字节数限制，默认为0，此时认为stdout capture mode模式关闭
- `stdout_event_enabled`: 如果设置为true，在进程写入标准文件是会发起PROCESS_LOG_STDOUT

更新supervisod配置文件

```shell
# 创建supervisor日志目录
mkdir -p /data/logs/supervisor/
# 更新supervisod配置
supervisorctl update
```

通过`supervisorctl status`查询supervisord状态，看到如下内容，代表supervisor正常运行

```shell
root@debian:/opt/etcd# supervisorctl status
etcd-server-101                  RUNNING   pid 85297, uptime 0:04:38
```

此时，我们再使用`netstat -luntp | grep etcd`查看网络服务端口，看到如下信息代表etcd已经正常启动

```shell
root@debian:/opt/etcd# netstat -luntp | grep etcd
tcp        0      0 192.168.122.101:2379      0.0.0.0:*               LISTEN      85298/./etcd        
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      85298/./etcd        
tcp        0      0 192.168.122.101:2380      0.0.0.0:*               LISTEN      85298/./etcd
```

### 4.5 集群验证

为了方便直接调用`etcdctl`命令，我们还可以创建其软连接

```bash
ln -s /opt/etcd/etcdctl /usr/local/bin/etcdctl
```

我们在任意节点使用etcdctl命令检查集群状态，需要注意的是，要确切指定证书的位置

```shell
etcdctl --cacert="/etc/kubernetes/pki/ca.pem" --cert="/etc/kubernetes/pki/etcd.pem" --key="/etc/kubernetes/pki/etcd-key.pem" --endpoints="https://192.168.122.101:2379,https://192.168.122.102:2379,https://192.168.122.103:2379" endpoint status --write-out=table
```

如果看到如下输出，代表 ectd 集群搭建成功

```bash
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.122.101:2379 | c8815bb4b21730b3 |   3.5.18 |  311 kB |      true |      false |         3 |      37628 |              37628 |        |
| https://192.168.122.102:2379 | f30299e8a0b43b4d |   3.5.18 |  311 kB |     false |      false |         3 |      37628 |              37628 |        |
| https://192.168.122.103:2379 | 61c90f737ccf2682 |   3.5.18 |  311 kB |     false |      false |         3 |      37628 |              37628 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

为了验证etcd集群是否正常工作，我们还可以现在`k8s-101`设置一个值，如下

```bash
etcdctl put name lixiaoming123
```

再通过`k8s-102`和`k8s-103`去读取值，如果正常取到，代表etcd集群正常工作，如下命令

```bash
etcdctl get name
```

如果需要了解`etcdctl`这个指令的更多用法，使用`--help`参数即可查看。

## 五、将kubernetes二进制安装包解压到系统中

在撰写这个文档时，kubernetes最新稳定版本为`v1.32.2`，所以这里也采用这个版本。通过 <https://kubernetes.io/zh-cn/releases/> 下载最新的对应操作系统的稳定版本。

我们下载好对应的 “Server Binarie” 之后，在所有k8s主机上执行安装，如下步骤

```shell
# 解压安装包
tar -zxvf kubernetes-server-linux-amd64.tar.gz
# 将安装包移到/opt目录下并根据版本重命名
mv kubernetes /opt/kubernetes-v1.32.2
# 创建软连接
ln -s /opt/kubernetes-v1.32.2/ /opt/kubernetes
```

在k8s二进制安装目录里包含了k8s源码包，还包含k8s核心组件的docker镜像，因为我们的核心服务不运行在容器里，所以可以删除掉，操作过程如下

```shell
# 进入k8s目录
cd /opt/kubernetes
# 删除源代码
rm kubernetes-src.tar.gz
# 删除二进制文件目录下以tar作为名称后缀的docker镜像包
rm -rf server/bin/*.tar
```

## 六、安装apiserver

搭建好etcd数据库集群之后，我们就可以安装apiserver组件了，在所有主机上安装apiserver，以下是具体的安装过程。

### 6.1 创建kubelet授权用户

因为后面要配置kubelet的bootstrap认证，即kubelet启动时自动创建CSR请求，这里需要在apiserver上开启token的认证。所以先在master上生成一个随机值作为token。下面在一台主机操作即可

```shell
# 创建证书目录
openssl rand -hex 10
```

假设生成的token为`88c916f382dc619a6bca`，把这个token写入到一个文件里，这里写入到 `/etc/kubernetes/bb.csv`，如下

```bash
cat > /etc/kubernetes/bb.csv <<EOF
88c916f382dc619a6bca,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

这里第二列定义了一个用户名kubelet-bootstrap，后面在配置kubelet时会为此用户授权。创建好该文件后，同步到其他主机。

### 6.2 启动apiser服务

#### 6.2.1 创建启动脚本

在apiserver二进制文件目录创建`/opt/kubernetes/server/bin/kube-apiserver.sh`启动脚本文件，写入以下内容

```bash
#!/bin/bash
./kube-apiserver \
    --v=2 \
    --logtostderr=true \
    --allow-privileged=true \
    --bind-address="192.168.122.101" \
    --secure-port="6443" \
    --token-auth-file="/etc/kubernetes/bb.csv" \
    --advertise-address="192.168.122.101" \
    --service-cluster-ip-range="10.96.0.0/16" \
    --service-node-port-range="30000-60000" \
    --etcd-servers="https://192.168.122.101:2379,https://192.168.122.102:2379,https://192.168.122.103:2379" \
    --etcd-cafile="/etc/kubernetes/pki/ca.pem" \
    --etcd-certfile="/etc/kubernetes/pki/etcd.pem" \
    --etcd-keyfile="/etc/kubernetes/pki/etcd-key.pem" \
    --client-ca-file="/etc/kubernetes/pki/ca.pem" \
    --tls-cert-file="/etc/kubernetes/pki/apiserver.pem" \
    --tls-private-key-file="/etc/kubernetes/pki/apiserver-key.pem" \
    --kubelet-client-certificate="/etc/kubernetes/pki/apiserver.pem" \
    --kubelet-client-key="/etc/kubernetes/pki/apiserver-key.pem" \
    --service-account-key-file="/etc/kubernetes/pki/ca-key.pem" \
    --service-account-signing-key-file="/etc/kubernetes/pki/ca-key.pem" \
    --service-account-issuer="https://kubernetes.default.svc.cluster.local" \
    --kubelet-preferred-address-types="InternalIP,ExternalIP,Hostname" \
    --enable-admission-plugins="NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota" \
    --authorization-mode="Node,RBAC" \
    --enable-bootstrap-token-auth=true
    #--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
    #--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
    #--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
    #--requestheader-allowed-names=aggregator  \
    #--requestheader-group-headers=X-Remote-Group  \
    #--requestheader-extra-headers-prefix=X-Remote-Extra-  \
    #--requestheader-username-headers=X-Remote-User
```

赋予执行权限

```shell
chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh
```

上面注释的部分是配置聚合层的，本环境里没有启用聚合层所以这些选项被注释了，如果配置了聚合层的话，则需要把#取消。相关参数说明

- `--v`：日志输出级别
- `--logtostderr`：将输出记录到标准日志，而不是文件，默认是true
- `--allow-privileged`：是否使用超级管理员权限创建容器，默认为false
- `--bind-address`：绑定的IP地址，如果没有指定地址（0.0.0.0或者::），默认是 0.0.0.0，代表所有的网卡都在监听服务
- `--secure-port`：参数指定的端口号对应监听的IP地址
- `--token-auth-file`：该文件用于指定api-server颁发证书的token授权
- `--advertise-address`：向集群广播的ip地址，这个ip地址必须能被集群的其他节点访问，如果不指定，将使用--bind-address，如果不指定--bind-addres，将使用默认网卡
- `--service-cluster-ip-range`：创建service时，使用的虚拟网段
- `--service-node-port-range`：创建service时，服务端口使用的端口范围（默认 30000-32767）
- `--etcd-cafile`：访问etcd时使用，ectd的ca文件
- `--etcd-certfile`：访问etcd时使用，ectd的证书文件
- `--etcd-servers`：各个etcd节点的IP和端口号
- `--etcd-keyfile`：访问etcd时使用，ectd的证书私钥文件
- `--client-ca-file`：访问apiserver时使用，客户端ca文件
- `--tls-cert-file`：访问apiserver时使用，tls证书文件
- `--tls-private-key-file`：访问apiserver时使用，tls证书私钥文件
- `--kubelet-client-certificate`：访问kubelet时使用，客户端证书路径
- `--kubelet-client-key`：访问kubelet时使用，客户端证书私钥
- `--service-account-key-file`：包含 PEM 编码的 x509 RSA 或 ECDSA 私钥或公钥，用来检查 ServiceAccount 的令牌
- `--service-account-signing-key-file`：指向包含当前服务账号令牌发放者的私钥的文件路径。 此发放者使用此私钥来签署所发放的 ID 令牌
- `--service-account-issuer`：服务账户令牌发放者的身份标识
- `--enable-admission-plugins`：允许使用的插件
- `--authorization-mode`：授权模式
- `--enable-bootstrap-token-auth`：是否使用token的方式来自动颁发证书，如果主机节点比较多的时候，手动颁发证书可能不太现实，可以使用基于token的方式自动颁发证书

以上是我们在启动apiserver的时候常用的参数，apiserver具有很多参数，很多参数也有默认值，可以`./kube-apiserver --hep`命令查看更多的帮助。

#### 6.2.2 使用supervisor运行

创建supervisor配置文件`/etc/supervisor/conf.d/kube-apiserver.conf`

```ini
[program:kube-apiserver-160]
directory=/opt/kubernetes/server/bin
command=/opt/kubernetes/server/bin/kube-apiserver.sh
numprocs=1
autostart=true
autorestart=true
startsecs=30
startretries=3
exitcodes=0,2
stopsignal=QUIT
stopwaitsecs=10
user=root
redirect_stderr=true
stdout_logfile=/data/logs/supervisor/apiserver.stdout.log
stdout_logfile_maxbytes=64MB
stdout_logfile_backups=4
stdout_capture_maxbytes=1MB
stdout_event_enabled=false
```

更新supervisor服务

```shell
supervisorctl update
```

再使用`supervisorctl status`命令查看apiserver启动状态，如果显示如下内容，代表正常服务

此时，还可以使用`netstat -luntp | grep kube-api`命令查看网络服务的端口是否正常，如果正常，将返回如下内容

## 七、搭建L4层负载均衡

负载均衡是网络层的一种机制，它将请求分发到后端服务器，从而实现高可用和高性能。负载均衡器通常包含一个或多个负载均衡器，每个负载均衡器负责将请求分发到后端服务器。负载均衡器通常使用TCP或UDP协议进行通信，并通过网络层（如TCP或UDP）将请求分发到后端服务器。负载均衡器通常使用轮询、权重、会话保持等功能来优化请求分发。

现在，我们需要在`k8s-101`和`k8s-102`上安装nginx作为反向代理服务且两个服务实现负载均衡，再使用keepalived保证高可用性

### 7.1 安装nginx

`k8s-101`在安装harbor时已经安装过，需要继续在`k8s-102`上安装

```bash
# 下载代码
wget https://nginx.org/download/nginx-1.26.3.tar.gz
# 解压文件
tar -zxvf nginx-1.26.3.tar.gz
# 进入源码目录
cd nginx-1.26.3
# 配置编译参数，--prefix参数指定安装目录
./configure \
--prefix=/usr/local/nginx-1.26.3 \
--with-stream \
--with-http_stub_status_module \
--with-http_ssl_module --with-http_v2_module \
--error-log-path=/data/log/nginx/error.log \
--http-log-path=/data/log/nginx/access.log
# 编译并安装
make && make install
# 设置链接
ln -s /usr/local/nginx-1.26.3 /usr/local/nginx
ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/nginx
```

### 7.2 配置nginx

安装完成之后，我们需要两台机的nginx的配置文件`/usr/local/nginx/conf/nginx.conf`的`http`节点旁边添加四层反向代码规则，将7443端口的流量使用负载均衡的方式转发到3台主机的6443端口上

```shell
# 设置代理规则
stream {
    upstream kube-apiserver {
        server 192.168.122.101:6443  max_fails=3  fail_timeout=30s;
        server 192.168.122.102:6443  max_fails=3  fail_timeout=3101
        server 192.168.122.103:6443  max_fails=3  fail_timeout=30s;
    }
    server {
        listen  7443;
        proxy_connect_timeout  2s;
        proxy_timeout  900s;
        proxy_pass kube-apiserver;
    }
}
```

在两台主机上配置好规则之后，通过`nginx -t`命令检查配置结果，如果输出以下内容代表配置正确

```shell
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

配置成功之后，启动nginx，如下指令

```bash
# 启动ginx，k8s-101主机使用 nginx -s reload重新加载配置即可
nginx
```

要让你手动编译安装的 Nginx 实现开机自启，你可以通过以下几种方式来完成（基于常见的 Linux 系统，如 Ubuntu、CentOS 等）。

### 7.3 使用systemd设置nginx开机自启

- 创建 Nginx 的 Systemd 服务文件

在 `/etc/systemd/system/` 目录下创建一个 `nginx.service` 文件：

```bash
sudo vim /etc/systemd/system/nginx.service
```

- 添加以下内容到服务文件中

```ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=network.target

[Service]
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PIDFile=/usr/local/nginx/logs/nginx.pid
Restart=on-failure
Type=forking

[Install]
WantedBy=multi-user.target
```

- 保存并刷新 `systemd` 配置

```bash
sudo systemctl daemon-reload
```

- 启用 Nginx 开机自启

```bash
sudo systemctl enable nginx
```

### 7.4 安装keepalived

Keepalived 的虚拟 IP 通过 VRRP 协议在多个服务器间切换，确保服务高可用性和负载均衡。这个虚拟 IP 是 Keepalived 配置的 IP 地址，不属于任何特定服务器，而是由主服务器持有，主服务器故障时切换到备用服务器。我们将使用keepalived实现代理服务器的高可用，以下是安装过程

```shell
apt install keepalived -y
```

在两台主机的创建`/etc/keepalived/check_port.sh`脚本文件，添加以下内容

```shell
#!/bin/bash
CHK_PORT=$1
if [ -n "$CHK_PORT" ]; then
    PORT_PROCESS=`ss -lnt|grep $CHK_PORT|wc -l`
    if [ $PORT_PROCESS -eq 0 ]; then
        echo "Port $CHK_PORT Is Not Used, End"
        exit 1
    fi
else
    echo "Check Port Cant Be Empty!"
    exit 1 
fi
```

添加执行权限

```shell
chmod +x /etc/keepalived/check_port.sh
```

以上的操作就准备好keepalived的基础环境了，接下来我们使用`k8s-101`这台主机作为主节点，使用`k8s-102`作为重节点，进行以下配置

`k8s-101`作为主节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```shell
! Configuration File for keepalived
# 全局配置
global_defs {
   router_id 192.168.122.101  # 当前服务器的唯一标识，通常用 IP 地址或主机名
}

# 定义健康检查脚本
vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"  # 检查端口 7443 是否在监听的脚本
    interval 2  # 每隔 2 秒执行一次脚本
    weight -20  # 如果脚本检查失败，当前服务器的优先级降低 20
}

# 定义一个 VRRP 实例
vrrp_instance VI_1 {
    state MASTER  # 当前服务器的初始状态是 MASTER（主服务器）
    interface enp1s0  # 绑定虚拟 IP 的网络接口
    virtual_router_id 251  # VRRP 实例的唯一 ID，范围是 1-255
    priority 100  # 当前服务器的优先级，数值越大优先级越高
    advert_int 1  # 每隔 1 秒发送一次 VRRP 通告
    mcast_src_ip 192.168.122.101  # 发送 VRRP 通告的源 IP 地址
    nopreempt  # 如果主服务器挂了又恢复，不会抢占虚拟 IP

    # 认证配置
    authentication {
        auth_type PASS  # 认证类型为密码认证
        auth_pass 1111  # 认证密码
    }

    # 虚拟 IP 地址配置
    virtual_ipaddress {
        192.168.122.100  # 虚拟 IP 地址，主服务器会持有这个 IP
    }
}
```

`k8s-102`作为从节点，修改`/etc/keepalived/keepalived.conf`配置文件如下

```shell
! Configuration File for keepalived

global_defs {
   router_id 192.168.122.102
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp3s0
    virtual_router_id 251
    priority 90
    advert_int 1
    mcast_src_ip 192.168.122.101
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.122.100
    }
}
```

启动服务

```shell
# 重启服务
systemctl restart keepalived
# 设置服务为开机自启
systemctl enable keepalived
```

需要注意的是，`interface`参数对应的是真实的主机网卡名称，`virtual_router_id`参数需要在同一个虚拟IP的前提下，设置需一致。

### 7.5 验证

通过`ping 192.169.9.190`的方式进行验证，如果有正常返回，代表keepalived运行正常。
