# Kubernetes：为什么 ConfigMap 能挂载成功，而 hostPath 是空目录

## 现象

Master 节点存在项目目录：

```text
/app/web_demo/web
```

Worker 节点不存在该目录。

### 使用 ConfigMap

```bash
kubectl create configmap web-demo-config-map-demo \
  --from-file=/app/web_demo/web
```

Pod 中可以看到项目文件。

### 使用 hostPath

```yaml
volumes:
- name: web-demo-volume
  hostPath:
    path: /app/web_demo/web
```

Pod 中的：

```text
/usr/share/nginx/html
```

是空目录。

---

# 根本原因

## ConfigMap 工作原理

ConfigMap 会先读取 Master 上的文件内容，然后把内容保存到 Kubernetes 集群中。

流程：

```text
Master文件
/app/web_demo/web
        ↓
kubectl读取文件
        ↓
创建ConfigMap
        ↓
保存到Kubernetes(etcd)
        ↓
Pod挂载ConfigMap
```

因此：

```text
Pod运行在Master
      或
Pod运行在Worker
```

都可以获取 ConfigMap 数据。

因为数据已经进入 Kubernetes 集群。

---

## hostPath 工作原理

hostPath 不会把文件上传到 Kubernetes。

它只是告诉 Kubernetes：

```text
请把当前节点上的某个目录
挂载到 Pod 里面
```

流程：

```text
Pod被调度到某个Node
          ↓
读取该Node上的目录
          ↓
挂载到Pod
```

例如：

```text
worker
└── /app/web_demo/web
```

挂载到：

```text
Pod
└── /usr/share/nginx/html
```

---

# 为什么目录是空的

假设：

```text
master
└── /app/web_demo/web （有文件）

worker
└── /app/web_demo/web （不存在）
```

而 Pod 被调度到了：

```text
worker
```

那么 Kubernetes 实际挂载的是：

```text
worker:/app/web_demo/web
```

而不是：

```text
master:/app/web_demo/web
```

因此 Pod 中看到的目录为空。

---

# 如何确认

查看 Pod 在哪个节点：

```bash
kubectl get pod -o wide
```

示例：

```text
NAME                      NODE
web-demo-xxxxx            worker
```

说明实际读取的是：

```text
worker:/app/web_demo/web
```

---

# hostPath 与 ConfigMap 的核心区别

| 项目 | ConfigMap | hostPath |
|--------|--------|--------|
| 数据来源 | Kubernetes 集群 | Node 本地目录 |
| 是否上传到集群 | 是 | 否 |
| Pod 在任意节点运行 | 可以 | 不一定 |
| 多节点环境 | 推荐 | 不推荐 |
| 适合 | 配置文件、小文件 | 本地目录挂载 |

---

# 解决方案

## 方案1：复制到所有节点

在每个节点创建：

```bash
mkdir -p /app/web_demo/web
```

同步项目文件。

例如：

```bash
scp -r /app/web_demo/web root@worker:/app/web_demo/
```

---

## 方案2：固定到 Master

```yaml
spec:
  nodeName: master
```

这样 Pod 一定运行在 Master。

适合学习环境。

---

## 方案3：使用 NFS

```text
NFS服务器
      ↓
Master
Worker
      ↓
Pod
```

多个节点共享同一份文件。

适合生产环境。

---

# 一句话总结

## ConfigMap

```text
先上传到 Kubernetes
再挂载给 Pod
```

因此：

```text
Pod在哪个节点都能用
```

---

## hostPath

```text
直接挂载当前节点本地目录
```

因此：

```text
Pod在哪个节点
就读取哪个节点的目录
```

这也是你看到：

```text
Master有文件
Worker没有文件

ConfigMap正常
hostPath为空
```

的根本原因。
