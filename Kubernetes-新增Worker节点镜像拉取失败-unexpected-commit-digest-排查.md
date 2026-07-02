# Kubernetes 实战：新增 Worker 节点后镜像无法拉取（unexpected commit digest）

## 问题现象

新增一个 Worker 节点后，Pod 一直无法启动。

查看 Pod：

``` bash
kubectl describe pod <pod-name>
```

Events 提示：

``` text
Failed to pull image "...:v3"

failed commit on ref "layer-sha256:3bdd..."

unexpected commit digest

expected:
sha256:3bdd47a1bb29b6d395c159fc82579a0123bb2c1bd515270ae6928f1ad98e765c

actual:
sha256:0a67fb82a6d981cffe356d507e1b00f9378f1c8f9e31b7a6ff34825003c14d01
```

------------------------------------------------------------------------

## 排查过程

-   Harbor 镜像损坏
-   Dockerfile 问题
-   基础镜像（base-php）
-   Harbor Tag 问题
-   网络问题
-   overlayfs
-   磁盘空间不足

最终均排除。

------------------------------------------------------------------------

## 最关键的一步

Master 与 Worker 分别执行：

``` bash
ctr -n k8s.io images pull --plain-http 111.229.217.116:30887/library-management-system-php/library-management-system-php:v3
```

### Master

能够正常拉取。

### Worker2

固定报错：

``` text
failed commit on ref "layer-sha256:3bdd..."
unexpected commit digest
```

最终定位：

> Worker2 本地 containerd Content Store / Snapshot 缓存异常。

------------------------------------------------------------------------

# 解决方案

## 1. 禁止调度新节点 （Master上运行）

``` bash
kubectl cordon worker2
```

## 2. 删除失败 Pod（Master上运行）

``` bash
kubectl delete pod -l app=library-management-system-php
```

## 3. 停止服务（新节点 worker2 上运行）

``` bash
systemctl stop kubelet
systemctl stop containerd
```

## 4. 清理 containerd 缓存（新节点 worker2 上运行）

``` bash
rm -rf /var/lib/containerd/io.containerd.content.v1.content
rm -rf /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs
rm -rf /var/lib/containerd/io.containerd.metadata.v1.bolt
```

## 5. 启动服务（新节点 worker2 上运行）

``` bash
systemctl start containerd
systemctl start kubelet
```

## 6. 恢复调度（Master上运行）

``` bash
kubectl uncordon worker2
```

成功后重新部署：

``` bash
kubectl rollout restart deployment library-management-system-php-dp
```

------------------------------------------------------------------------

# 原因分析

新增 Worker 节点后，containerd 本地 Content Store / Snapshot
缓存异常，导致 layer 在 commit 时 digest 校验失败。

该问题不是：

-   Harbor 问题
-   Dockerfile 问题
-   Kubernetes YAML 问题

而是 **Worker 节点本地 containerd 缓存异常**。

------------------------------------------------------------------------

# 排查经验

出现 `unexpected commit digest` 时建议：

1.  使用 `ctr --plain-http images pull` 验证镜像。
2.  对比 Master 与 Worker 是否都存在问题。
3.  若仅新增 Worker 有问题，优先检查 containerd 本地缓存。
4.  清理 `/var/lib/containerd` 后重新拉取镜像。
