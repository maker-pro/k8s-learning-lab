# Kubernetes 从零搭建到 Nginx 部署实战手册

## 一、环境说明

### 宿主机

```text
Windows 11
Oracle VirtualBox
```

### 虚拟机

```text
master
192.168.56.11

worker
192.168.56.12
```

系统：

```text
Debian 12
```

网络：

```text
网卡1：NAT
网卡2：Host-Only
```

作用：

```text
NAT
↓
访问互联网

Host-Only
↓
节点间通信
```

---

# 二、系统初始化

## 设置主机名

master：

```bash
hostnamectl set-hostname master
```

worker：

```bash
hostnamectl set-hostname worker
```

---

## 配置 hosts

编辑：

```bash
vim /etc/hosts
```

添加：

```text
192.168.56.11 master
192.168.56.12 worker
```

验证：

```bash
ping master
ping worker
```

---

# 三、关闭 Swap

查看：

```bash
free -h
```

关闭：

```bash
swapoff -a
```

永久关闭：

```bash
vim /etc/fstab
```

注释：

```text
UUID=xxxxxxxx none swap sw 0 0
```

改为：

```text
#UUID=xxxxxxxx none swap sw 0 0
```

验证：

```bash
swapon --show
```

无输出即可。

---

# 四、K8S内核参数

创建：

```bash
cat <<EOF >/etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

加载：

```bash
modprobe overlay
modprobe br_netfilter
```

---

创建：

```bash
cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```

应用：

```bash
sysctl --system
```

验证：

```bash
sysctl net.ipv4.ip_forward
```

结果：

```text
net.ipv4.ip_forward = 1
```

---

# 五、安装 Containerd

安装：

```bash
apt update
apt install -y containerd
```

生成配置：

```bash
mkdir -p /etc/containerd

containerd config default \
>/etc/containerd/config.toml
```

修改：

```toml
SystemdCgroup = true
```

重启：

```bash
systemctl restart containerd
systemctl enable containerd
```

验证：

```bash
systemctl status containerd
```

---

# 六、安装 Kubernetes

安装依赖：

```bash
apt install -y apt-transport-https ca-certificates curl gpg
```

导入密钥：

```bash
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
| gpg --dearmor \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

添加源：

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
>/etc/apt/sources.list.d/kubernetes.list
```

安装：

```bash
apt update

apt install -y \
kubelet \
kubeadm \
kubectl
```

锁定版本：

```bash
apt-mark hold kubelet kubeadm kubectl
```

---

# 七、初始化 Master

master执行：

```bash
kubeadm init \
--apiserver-advertise-address=192.168.56.11 \
--pod-network-cidr=10.244.0.0/16
```

说明：

```text
192.168.56.11
=
Master节点IP

10.244.0.0/16
=
Flannel Pod网络
```

---

# 八、配置 Kubectl

root用户：

```bash
mkdir -p ~/.kube

cp /etc/kubernetes/admin.conf \
~/.kube/config
```

验证：

```bash
kubectl get nodes
```

---

# 九、安装 Flannel

```bash
kubectl apply -f \
https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

查看：

```bash
kubectl get pods -A
```

等待：

```text
coredns Running
flannel Running
```

---

# 十、Worker 加入集群

Master打印：

```bash
kubeadm join ...
```

复制到 worker 执行。

例如：

```bash
kubeadm join 192.168.56.11:6443 \
--token xxx \
--discovery-token-ca-cert-hash sha256:xxx
```

---

# 十一、验证集群

```bash
kubectl get nodes -o wide
```

正常：

```text
master Ready
worker Ready
```

---

# 十二、部署 Nginx

创建：

```bash
kubectl create deployment test-nginx \
--image=nginx \
--replicas=3
```

查看：

```bash
kubectl get pod -o wide
```

---

# 十三、暴露 Service

```bash
kubectl expose deployment test-nginx \
--port=80 \
--type=NodePort
```

查看：

```bash
kubectl get svc
```

结果：

```text
80:32055/TCP
```

访问：

```text
http://192.168.56.11:32055
```

或：

```text
http://192.168.56.12:32055
```

---

# 十四、常见问题

## 问题1：CoreDNS Pending

现象：

```text
coredns Pending
```

原因：

```text
Flannel未安装
```

解决：

```bash
kubectl apply -f kube-flannel.yml
```

---

## 问题2：Worker Join失败

错误：

```text
[ERROR FileExisting-losetup]
```

原因：

```text
losetup不存在
```

解决：

```bash
apt install -y util-linux
```

确认：

```bash
which losetup
```

---

## 问题3：kubectl连接localhost:8080

错误：

```text
connection refused
```

原因：

```text
未配置admin.conf
```

解决：

```bash
cp /etc/kubernetes/admin.conf ~/.kube/config
```

---

## 问题4：ErrImagePull

现象：

```text
ErrImagePull
```

原因：

```text
镜像拉取失败
```

解决：

```text
配置镜像仓库
配置Harbor
配置containerd镜像加速
```

---

## 问题5：hostPath挂载为空

现象：

```text
Pod内部目录为空
```

原因：

```text
worker节点不存在对应目录
```

例如：

```text
master:
/app/web_demo

worker:
无目录
```

Pod调度到worker后：

```text
目录为空
```

解决：

```text
所有节点同步目录

或

使用ConfigMap

或

制作镜像
```

---

## 问题6：ConfigMap更新不生效

更新：

```bash
kubectl create configmap xxx \
--from-file=xxx \
-o yaml \
--dry-run=client \
| kubectl apply -f -
```

重启：

```bash
kubectl rollout restart deployment nginx
```

---

# 十五、常用命令

查看节点：

```bash
kubectl get nodes -o wide
```

查看Pod：

```bash
kubectl get pods -A
```

查看日志：

```bash
kubectl logs POD_NAME
```

进入容器：

```bash
kubectl exec -it POD_NAME -- bash
```

删除Deployment：

```bash
kubectl delete deployment nginx
```

删除Service：

```bash
kubectl delete svc nginx
```

重启Deployment：

```bash
kubectl rollout restart deployment nginx
```

查看事件：

```bash
kubectl get events -A
```

查看Pod详情：

```bash
kubectl describe pod POD_NAME
```


---

## 问题7：Flannel 选择错误网卡（VirtualBox 双网卡环境）

### 现象

集群节点状态正常：

```bash
kubectl get nodes
```

显示：

```text
master Ready
worker Ready
```

Service 也正常：

```bash
kubectl get svc
kubectl get endpoints
```

但是：

```bash
curl http://10.244.1.2
curl http://10.244.1.3
curl http://10.244.1.4
```

无法访问。

或者：

```bash
curl http://192.168.56.12:NodePort
```

访问失败。

### 环境

```text
VirtualBox

网卡1:
NAT
10.0.2.15

网卡2:
Host-Only
192.168.56.x
```

例如：

```text
master
enp0s3 = 10.0.2.15
enp0s8 = 192.168.56.11

worker
enp0s3 = 10.0.2.15
enp0s8 = 192.168.56.12
```

### 根本原因

Flannel 默认选择默认路由网卡。

查看日志：

```bash
kubectl -n kube-flannel logs -l app=flannel
```

如果看到：

```text
Using interface with name enp0s3 and address 10.0.2.15
```

说明 Flannel 选择了 NAT 网卡。

由于 VirtualBox NAT 网络中：

```text
master = 10.0.2.15
worker = 10.0.2.15
```

两个节点对 Flannel 来说变成了同一个地址。

导致：

```text
Pod 网络异常
NodePort 无法访问
跨节点通信失败
```

### 验证

查看 Flannel 路由：

```bash
ip route | grep 10.244
```

查看 Flannel 日志：

```bash
kubectl -n kube-flannel logs -l app=flannel --tail=50
```

### 解决方案

编辑 Flannel DaemonSet：

```bash
kubectl -n kube-flannel edit ds kube-flannel-ds
```

找到：

```yaml
args:
- --ip-masq
- --kube-subnet-mgr
```

修改为：

```yaml
args:
- --ip-masq
- --kube-subnet-mgr
- --iface=enp0s8
```

说明：

```text
enp0s8
=
Host-Only 网卡
=
192.168.56.x
```

重启：

```bash
kubectl -n kube-flannel rollout restart ds kube-flannel-ds
```

验证：

```bash
kubectl -n kube-flannel logs -l app=flannel --tail=50
```

正确结果：

```text
Using interface with name enp0s8 and address 192.168.56.11
Using interface with name enp0s8 and address 192.168.56.12
```

此时再测试：

```bash
curl http://10.244.1.2
curl http://10.244.1.3
curl http://10.244.1.4
```

以及：

```bash
curl http://192.168.56.12:NodePort
```

应该恢复正常。
