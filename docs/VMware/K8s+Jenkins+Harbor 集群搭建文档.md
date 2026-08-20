

> 本机物理内存：32G，VMware NAT 网关：`192.168.244.2`
> 
> 部署架构：单 Master + 双 Node + Jenkins‑Harbor 一体机；代码源使用 GitHub 在线仓库，不部署本地 GitLab

## 一、集群规划表

| 角色             | 主机名            | IP 地址           | CPU | 内存  | 磁盘  | 运行服务            |
| :------------- | :------------- | :-------------- | :-- | :-- | :-- | :-------------- |
| k8s‑master01   | k8s‑master01   | 192.168.244.120 | 2 核 | 4G  | 40G | k8s 控制平面、docker |
| k8s‑node01     | k8s‑node01     | 192.168.244.121 | 2 核 | 3G  | 40G | k8s 工作节点、docker |
| k8s‑node02     | k8s‑node02     | 192.168.244.122 | 2 核 | 3G  | 40G | k8s 工作节点、docker |
| jenkins‑harbor | jenkins‑harbor | 192.168.244.123 | 2 核 | 4G  | 80G | Jenkins         |

> 总虚拟机内存：14G，宿主机留有充足内存运行 Windows、浏览器、IDE 等软件。

### 访问地址汇总

1. Jenkins：`http://192.168.244.123:8080`
2. Harbor：`http://192.168.244.123:80`

### 流水线整体流程

`GitHub在线仓库 → Jenkins拉取源码 → docker build构建镜像 → push到Harbor私有仓库 → kubectl部署Pod到K8s集群`

## 二、VMware 克隆操作说明

> 步骤：先创建**基础模板虚拟机（CentOS7）**，做完全部系统初始化，关机，使用 VMware【完整克隆】，生成 4 台机器。
> 
> ⚠️必须选择**完整克隆**，不要链接克隆；链接克隆容易出现主机名、IP、UUID 冲突。

1. 新建第一台 CentOS7 虚拟机，作为模板机，CPU2 核，内存 4G，磁盘 40G，网卡 NAT 模式。
2. 模板机内部执行【第三章：模板机完整初始化脚本】全部操作。
3. 执行完成后执行 `poweroff` 关机。
4. VMware 右键虚拟机 → 克隆 → 完整克隆，克隆 4 份：
   
    - 克隆 1：k8s‑master01，内存保持 4G，磁盘 40G，网卡 NAT
    - 克隆 2：k8s‑node01，修改内存为 3G，磁盘 40G，网卡 NAT
    - 克隆 3：k8s‑node02，修改内存为 3G，磁盘 40G，网卡 NAT
    - 克隆 4：jenkins‑harbor，内存 4G，磁盘扩容到 80G，网卡 NAT
    
5. 分别开机 4 台虚拟机，每台修改静态 IP、修改本机主机名。

> 克隆后注意：CentOS7 网卡 ens33 的 UUID 不要复制，保留本机原有 UUID，只修改 IPADDR。

## 三、模板机完整初始化（模板机全部执行，克隆前完成）

> 模板机先不要设置固定 IP，克隆后每台单独配置 IP；其余全部初始化模板机做完。

### 1. 修复 CentOS7 yum 源（官方源已下线）


```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all
yum makecache
```

### 2. 关闭防火墙、SELinux

```
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
getenforce
```

### 3. 关闭 swap（K8s 强制要求）

```
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
free -h
```

### 4. 时间同步 chrony

```
yum install -y chrony
systemctl start chronyd
systemctl enable chronyd
chronyc sources
```

### 5. 预写入集群 hosts 解析

```
> /etc/hosts
cat >> /etc/hosts << EOF
192.168.244.120 k8s‑master01
192.168.244.121 k8s‑node01
192.168.244.122 k8s‑node02
192.168.244.123 jenkins‑harbor
EOF
cat /etc/hosts
```

### 6. 安装 docker

```
yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
```

### 7. docker 配置加速 + 信任 harbor 仓库
```
cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ],
  "insecure-registries": ["192.168.244.123"],
  "log-driver":"json-file",
  "log-opts": {"max-size":"100m"}
}
EOF

systemctl daemon-reload
systemctl restart docker
docker --version
```

### 8. 关闭 NetworkManager 服务

```
systemctl stop NetworkManager
systemctl disable NetworkManager
```
### 9. 模板机关机，准备克隆

```
poweroff
```
## 四、克隆完成后，每台机器单独配置

> 4 台机器开机，分别执行下面操作，只修改 IP 与本机主机名，其余配置克隆已经继承。
### 网卡配置文件 `/etc/sysconfig/network-scripts/ifcfg‑ens33`

> 保留文件内原有 UUID，只修改 IPADDR，其余参数复制。


```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID=【保留本机原有UUID，不要复制】
DEVICE="ens33"
ONBOOT="yes"
IPADDR=【填入对应机器IP】
PREFIX="24"
GATEWAY="192.168.244.2"
DNS1="223.5.5.5"
DNS2="114.114.114.114"
IPV6_PRIVACY="no"
```

|主机|IPADDR 值|设置主机名命令|
|---|---|---|
|k8s‑master01|`192.168.244.120`|`hostnamectl set-hostname k8s-master01`|
|k8s‑node01|`192.168.244.121`|`hostnamectl set-hostname k8s-node01`|
|k8s‑node02|`192.168.244.122`|`hostnamectl set-hostname k8s-node02`|
|jenkins‑harbor|`192.168.244.123`|`hostnamectl set-hostname jenkins-harbor`|

每台修改网卡后执行、测试网络状态：

```bash
systemctl restart network
ip addr
ping www.baidu.com
```

## 五、后续阶段任务清单

### 阶段 1：k8s‑master01、node01、node02 部署 Kubernetes 集群

1. 配置 k8s 阿里云 yum 源
2. 安装 kubeadm、kubelet、kubectl
3. 加载内核模块，ipvs 配置
4. kubeadm init 初始化 master 节点
5. node 节点 join 加入集群
6. 安装 CNI 网络插件 calico
7. 验证集群状态 `kubectl get nodes`

### 阶段 2：jenkins‑harbor 机器部署 docker‑compose + Jenkins+Harbor

1. 安装 docker‑compose
2. docker‑compose.yml 部署 Jenkins、Harbor
3. 初始化 Harbor 项目账号
4. Jenkins 安装插件，配置 GitHub 访问、Harbor 凭证

### 阶段 3：调试流水线

1. 编写 Jenkinsfile
2. GitHub 配置 webhook 触发流水线
3. 完整跑通：代码 push → 自动构建镜像 → push harbor → k8s 部署应用

> 需要的时候，我可以直接输出对应阶段完整复制粘贴命令。

## 重要注意事项

1. 全部使用**完整克隆**，禁止链接克隆。
2. 所有机器 swap 必须关闭，否则 k8s 启动失败。
3. docker 的`insecure‑registries`写`192.168.244.123`，k8s 节点拉 harbor 镜像才不会报错。
4. jenkins‑harbor 机器磁盘调整为 80G，存放镜像与构建产物。
5. 本集群为学习环境，单 Master 无高可用，禁止生产使用。

![](docs/VMware/assets/Pasted%20image%2020260820104836.png)