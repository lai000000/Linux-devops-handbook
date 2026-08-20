# Linux 安装 Docker 笔记（面向 K8s）

> 适用系统：CentOS 7 / 8（Rocky Linux / AlmaLinux 兼容）
> 目的：为后续搭建 K8s 集群做准备，Docker 作为容器引擎 + 镜像构建工具。

---

## 1. 环境准备

### 1.1 检查系统

```bash
# 查看系统版本
cat /etc/redhat-release

# 查看内核版本（K8s 要求内核 ≥ 3.10，建议 4.x+）
uname -r

# 查看 CPU / 内存
nproc
free -h
```

### 1.2 关闭 swap（K8s 硬性要求）

K8s 安装时如果 swap 未关闭，`kubelet` 会启动失败。

```bash
# 临时关闭
swapoff -a

# 永久关闭：注释掉 /etc/fstab 中的 swap 行
sed -i '/swap/s/^/#/' /etc/fstab

# 验证
free -h
```

### 1.3 关闭 SELinux（K8s 推荐）

K8s 官方文档要求 SELinux 为 disabled 或 permissive。

```bash
# 临时关闭
setenforce 0

# 永久关闭：修改配置文件
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# 验证
getenforce
```

### 1.4 关闭 firewalld（内网集群常见做法）

K8s 集群网络由 CNI 插件管理，firewalld 常导致 Pod 网络异常，内网环境建议关闭：

```bash
systemctl stop firewalld
systemctl disable firewalld
```

> 若必须开启防火墙，需放行：6443(kube-apiserver)、2379/2380(etcd)、10250(kubelet)、30000-32767(NodePort) 等端口。

### 1.5 加载内核模块

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 确保模块已加载
lsmod | grep -E "overlay|br_netfilter"
```

### 1.6 配置内核参数（iptables 桥接流量）

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# 验证
sysctl net.bridge.bridge-nf-call-iptables
```

---

## 2. 安装 Docker（CentOS / yum 源）

### 2.1 卸载旧版本

```bash
yum remove docker docker-client docker-common docker-engine -y
```

### 2.2 安装依赖并添加 Docker 仓库（阿里云源）

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2

# CentOS 7
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# CentOS 8 如报错，改用：
# yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# sed -i 's/\$releasever/8/g' /etc/yum.repos.d/docker-ce.repo

# 将仓库地址全部替换为阿里云镜像
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo

# 刷新缓存
yum makecache fast
```

### 2.3 安装 Docker

```bash
yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 2.4 启动并设置开机自启

```bash
systemctl enable docker --now
systemctl status docker --no-pager
```

### 2.5 验证

```bash
docker version
docker run --rm hello-world
```

### 2.6 将当前用户加入 docker 组（免 sudo）

```bash
usermod -aG docker $USER
newgrp docker   # 立即生效，或重新登录
```

---

## 3. 配置镜像加速（国内必做）

编辑 `/etc/docker/daemon.json`：

```bash
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn"
  ]
}
EOF

systemctl daemon-reload
systemctl restart docker

# 验证加速是否生效
docker info | grep -A5 "Registry Mirrors"
```

> 注意：第三方加速源可能失效，如失效可改用 `https://mirror.ccs.tencentyun.com`（仅腾讯云内网）。

---

## 4. 为 K8s 做准备（关键配置）

### 4.1 cgroup driver 改为 systemd（K8s 要求）

K8s 官方要求容器运行时的 cgroup driver 与 kubelet 保持一致，推荐 `systemd`。

```bash
tee /etc/docker/daemon.json <<'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

systemctl daemon-reload
systemctl restart docker

# 验证 cgroup driver
docker info | grep -i cgroup
# 输出应为：Cgroup Driver: systemd
```

### 4.2 ⚠️ 重要提醒：K8s 1.24+ 已移除 dockershim

K8s 从 **v1.24** 起移除了内置的 dockershim，**直接装 Docker 后 kubeadm 无法识别**。

两种方案二选一：

**方案 A：安装 cri-dockerd 适配器（坚持用 Docker 做运行时）**

```bash
# 下载 cri-dockerd（以 v0.3.16 为例，版本按最新发布调整）
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.16/cri-dockerd-0.3.16.amd64.tgz
tar xvf cri-dockerd-0.3.16.amd64.tgz
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd

# 生成 systemd 服务
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
cp cri-docker.service cri-docker.socket /etc/systemd/system/
systemctl daemon-reload
systemctl enable cri-docker --now

# 验证
cri-dockerd --version
```

**方案 B：改用 containerd（官方推荐，K8s 1.24+ 默认）**

```bash
# Docker 安装时已自带 containerd，只需配置好镜像加速
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml > /dev/null

# 修改 sandbox 镜像源为国内可访问（否则 pause 镜像拉不下来）
sed -i 's#registry.k8s.io/pause:3.8#registry.aliyuncs.com/google_containers/pause:3.9#' /etc/containerd/config.toml

# 启用 SystemdCgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd --now
```

> **建议**：如果你是 K8s 新手，直接选**方案 B（containerd）**，官方支持最好、坑最少；Docker 继续留着用来构建镜像即可。

---

## 5. Docker 常用命令速查

```bash
# 镜像
docker images                       # 查看本地镜像
docker pull nginx:latest            # 拉取镜像
docker rmi <image_id>               # 删除镜像
docker build -t myapp:v1 .          # 构建镜像
docker tag myapp:v1 reg.com/myapp:v1
docker push reg.com/myapp:v1        # 推送镜像

# 容器
docker ps -a                        # 查看容器（含已停止）
docker run -d --name nginx -p 80:80 nginx
docker exec -it nginx bash          # 进入容器
docker logs -f nginx                # 查看日志
docker stop/start/restart nginx     # 生命周期
docker rm -f nginx                  # 删除容器

# 清理
docker system prune -a              # 清理所有无用镜像/容器/缓存
docker system df                    # 查看磁盘占用

# compose
docker compose up -d
docker compose down
```

---

## 6. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `yum makecache` 报 404 | CentOS 8 已停止维护，`$releasever` 解析失败 | 在 repo 里 `sed` 替换 `$releasever` 为 `8` |
| `failed to pull image` 超时 | 镜像源失效/网络问题 | 更换 `registry-mirrors` 加速源 |
| 容器内时间不对 | 未挂载时区 | 运行加 `-v /etc/localtime:/etc/localtime:ro` |
| `Cannot connect to the Docker daemon` | docker 服务未启动 | `systemctl start docker` |
| 磁盘被日志占满 | 日志无上限 | 配置 `log-opts` 限制大小（见 4.1） |
| kubeadm 报容器运行时未找到 | K8s 1.24+ 需 cri-dockerd/containerd | 参考第 4.2 节 |
| Pod 网络不通 | firewalld 拦截 / 桥接流量未转发 | 关闭 firewalld（见 1.4）或检查 sysctl（见 1.6） |

---

## 7. 后续 K8s 安装链接

- 安装 containerd 方案：`docs/k8s/1.md`（截图笔记）
- 后续补：kubeadm 初始化、CNI 网络插件（calico/flannel）、集群 join

> 笔记维护说明：本仓库图片统一放在当前笔记同级的 `assets/` 目录，引用格式 `![](assets/xxx.png)`，Obsidian 与 GitHub 均可正常显示。
