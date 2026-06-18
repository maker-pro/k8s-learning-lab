# Kubernetes YAML 中文图解手册：Deployment + Service

> 适合对象：刚开始学习 Kubernetes YAML、正在用 `Deployment + Service(NodePort)` 部署 Nginx / HTML 页面的人。  
> 重点目标：搞清楚 `replicas`、`selector`、`matchLabels`、`template.labels`、`Service.selector`、`port`、`targetPort`、`nodePort` 到底是什么意思，以及它们之间如何关联。

---

## 目录

1. [本手册使用的两个 YAML](#1-本手册使用的两个-yaml)
2. [Deployment 是什么](#2-deployment-是什么)
3. [Deployment YAML 完整解释](#3-deployment-yaml-完整解释)
4. [`replicas: 3` 是什么意思](#4-replicas-3-是什么意思)
5. [`selector.matchLabels` 是什么意思](#5-selectormatchlabels-是什么意思)
6. [`template.metadata.labels` 是什么意思](#6-templatemetadatalabels-是什么意思)
7. [`selector` 和 `labels` 为什么必须对应](#7-selector-和-labels-为什么必须对应)
8. [Service 是什么](#8-service-是什么)
9. [Service YAML 完整解释](#9-service-yaml-完整解释)
10. [`Service.selector.app=html-demo` 是什么意思](#10-serviceselectorapphtml-demo-是什么意思)
11. [`port`、`targetPort`、`nodePort` 的区别](#11-porttargetportnodeport-的区别)
12. [Deployment 和 Service 的完整关系图](#12-deployment-和-service-的完整关系图)
13. [外部访问流程图](#13-外部访问流程图)
14. [常用排查命令](#14-常用排查命令)
15. [常见错误](#15-常见错误)
16. [最终记忆口诀](#16-最终记忆口诀)

---

# 1. 本手册使用的两个 YAML

## 1.1 Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: html-demo

spec:
  replicas: 3

  selector:
    matchLabels:
      app: html-demo

  template:
    metadata:
      labels:
        app: html-demo

    spec:
      containers:
      - name: nginx
        image: nginx
```

---

## 1.2 Service YAML

```yaml
apiVersion: v1
kind: Service

metadata:
  name: html-demo-svc

spec:
  type: NodePort

  selector:
    app: html-demo

  ports:
  - port: 80
    targetPort: 80
    nodePort: 32056
```

---

# 2. Deployment 是什么

Deployment 是 Kubernetes 中最常用的工作负载资源。

它主要负责：

```text
1. 创建 Pod
2. 管理 Pod 副本数量
3. Pod 挂了自动重建
4. 支持滚动更新
5. 支持回滚
```

简单理解：

```text
Deployment 不是容器
Deployment 也不是 Pod

Deployment 是 Pod 的管理器
```

例如你写：

```yaml
replicas: 3
```

意思就是：

```text
Deployment 要保证始终有 3 个 Pod 在运行
```

如果其中一个 Pod 挂了，Deployment 会自动再创建一个新的 Pod。

---

# 3. Deployment YAML 完整解释

原始 YAML：

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: html-demo

spec:
  replicas: 3

  selector:
    matchLabels:
      app: html-demo

  template:
    metadata:
      labels:
        app: html-demo

    spec:
      containers:
      - name: nginx
        image: nginx
```

---

## 3.1 `apiVersion: apps/v1`

```yaml
apiVersion: apps/v1
```

表示当前资源使用的是 Kubernetes 的 `apps/v1` API 版本。

Deployment 属于 `apps` 这个 API 组，所以这里写：

```yaml
apps/v1
```

---

## 3.2 `kind: Deployment`

```yaml
kind: Deployment
```

表示当前 YAML 要创建的资源类型是 Deployment。

也就是说，这份 YAML 不是创建 Pod，也不是创建 Service，而是创建一个 Deployment。

---

## 3.3 `metadata.name`

```yaml
metadata:
  name: html-demo
```

表示 Deployment 的名字叫：

```text
html-demo
```

创建后可以通过下面命令查看：

```bash
kubectl get deployment
```

也可以指定名称查看：

```bash
kubectl get deployment html-demo
```

---

# 4. `replicas: 3` 是什么意思

```yaml
spec:
  replicas: 3
```

意思是：

```text
创建 3 个 Pod 副本
```

Deployment 会尽量保证集群中一直有 3 个符合条件的 Pod。

---

## 4.1 图解

```text
Deployment: html-demo
replicas: 3
        │
        ▼
┌─────────────────────┐
│ Pod 1               │
│ app=html-demo       │
└─────────────────────┘

┌─────────────────────┐
│ Pod 2               │
│ app=html-demo       │
└─────────────────────┘

┌─────────────────────┐
│ Pod 3               │
│ app=html-demo       │
└─────────────────────┘
```

---

## 4.2 验证命令

创建 Deployment 后执行：

```bash
kubectl get pods
```

你可能会看到类似：

```text
NAME                         READY   STATUS    RESTARTS   AGE
html-demo-6d7b8f9c4d-abcde   1/1     Running   0          1m
html-demo-6d7b8f9c4d-fghij   1/1     Running   0          1m
html-demo-6d7b8f9c4d-klmno   1/1     Running   0          1m
```

这里就是 3 个 Pod。

---

## 4.3 如果删除一个 Pod 会怎样

假设你删除其中一个 Pod：

```bash
kubectl delete pod html-demo-6d7b8f9c4d-abcde
```

Deployment 会发现：

```text
我要求 replicas=3
现在只剩 2 个 Pod
```

于是它会自动创建一个新的 Pod。

这就是 Deployment 的自愈能力。

---

# 5. `selector.matchLabels` 是什么意思

这一段非常重要：

```yaml
selector:
  matchLabels:
    app: html-demo
```

它的意思是：

```text
Deployment 要管理所有标签为 app=html-demo 的 Pod
```

可以理解成 Deployment 在问自己：

```text
我要管理哪些 Pod？
```

答案就是：

```text
管理标签 app=html-demo 的 Pod
```

---

## 5.1 什么是 Label

Label 就是 Kubernetes 给资源贴的标签。

例如：

```yaml
labels:
  app: html-demo
```

表示给资源贴上一个标签：

```text
app=html-demo
```

标签本身没有特殊含义，它是你自己定义的。

你也可以写成：

```yaml
labels:
  app: nginx
```

或者：

```yaml
labels:
  project: blog
```

但是 Deployment、Service、Pod 之间要靠这些标签建立关系，所以标签要写对。

---

# 6. `template.metadata.labels` 是什么意思

这一段：

```yaml
template:
  metadata:
    labels:
      app: html-demo
```

意思是：

```text
Deployment 创建出来的 Pod，都要带上 app=html-demo 这个标签
```

注意这里的 `template` 是 Pod 模板。

也就是说：

```text
Deployment 不是直接写 Pod
Deployment 是通过 template 来描述要创建什么样的 Pod
```

---

## 6.1 图解 template

```text
Deployment
   │
   │ 根据 template 创建 Pod
   ▼

template:
  metadata:
    labels:
      app=html-demo
  spec:
    containers:
    - nginx
```

最终创建出来：

```text
Pod
  labels:
    app=html-demo
  container:
    nginx
```

---

# 7. `selector` 和 `labels` 为什么必须对应

这里是 Deployment YAML 最核心的关系：

```yaml
selector:
  matchLabels:
    app: html-demo
```

和：

```yaml
template:
  metadata:
    labels:
      app: html-demo
```

这两个地方必须对应。

---

## 7.1 正确关系

```text
Deployment.selector.matchLabels
        │
        │ 选择 app=html-demo 的 Pod
        ▼
Pod labels
        app=html-demo
```

也就是说：

```text
Deployment 说：我要管理 app=html-demo 的 Pod

template 说：我创建出来的 Pod 就叫 app=html-demo

这样 Deployment 就能管理自己创建出来的 Pod
```

---

## 7.2 如果不一致会怎样

错误示例：

```yaml
selector:
  matchLabels:
    app: html-demo

template:
  metadata:
    labels:
      app: nginx
```

这就变成：

```text
Deployment 想管理 app=html-demo 的 Pod
但是 template 创建出来的是 app=nginx 的 Pod
```

Kubernetes 会直接报错，因为 Deployment 找不到自己创建的 Pod。

常见错误提示类似：

```text
selector does not match template labels
```

---

## 7.3 记忆方法

```text
selector.matchLabels 是“我要找谁”
template.metadata.labels 是“我创建出来的 Pod 叫什么标签”
```

所以这两个地方必须对应。

---

# 8. Service 是什么

Service 是 Kubernetes 中用来访问 Pod 的资源。

为什么需要 Service？

因为 Pod 是不稳定的：

```text
Pod 会重建
Pod IP 会变化
Pod 数量会变化
```

如果你直接访问 Pod IP，一旦 Pod 被重建，IP 可能就变了。

Service 的作用就是提供一个稳定入口。

---

## 8.1 Service 的作用

```text
1. 给一组 Pod 提供固定访问入口
2. 自动发现匹配标签的 Pod
3. 在多个 Pod 之间做负载均衡
4. 对外暴露服务，比如 NodePort
```

---

## 8.2 图解

```text
没有 Service：

客户端
  │
  ├── Pod IP 1
  ├── Pod IP 2
  └── Pod IP 3

问题：
Pod IP 可能变化，访问不稳定
```

```text
有 Service：

客户端
  │
  ▼
Service
  │
  ├── Pod 1
  ├── Pod 2
  └── Pod 3

好处：
客户端只访问 Service，Pod 变化由 Service 自动处理
```

---

# 9. Service YAML 完整解释

原始 YAML：

```yaml
apiVersion: v1
kind: Service

metadata:
  name: html-demo-svc

spec:
  type: NodePort

  selector:
    app: html-demo

  ports:
  - port: 80
    targetPort: 80
    nodePort: 32056
```

---

## 9.1 `apiVersion: v1`

```yaml
apiVersion: v1
```

Service 属于 Kubernetes 核心 API 组，所以使用：

```yaml
v1
```

Deployment 使用的是：

```yaml
apps/v1
```

这是因为它们属于不同的 API 组。

---

## 9.2 `kind: Service`

```yaml
kind: Service
```

表示当前 YAML 要创建 Service。

---

## 9.3 `metadata.name`

```yaml
metadata:
  name: html-demo-svc
```

表示 Service 名称是：

```text
html-demo-svc
```

查看命令：

```bash
kubectl get service
```

或者：

```bash
kubectl get svc
```

---

## 9.4 `type: NodePort`

```yaml
type: NodePort
```

表示这个 Service 会在每个 Kubernetes 节点上开放一个端口。

例如：

```yaml
nodePort: 32056
```

那么你可以通过：

```text
任意 Node IP:32056
```

访问这个服务。

比如：

```bash
curl http://192.168.56.101:32056
curl http://192.168.56.102:32056
```

前提是对应节点网络可达，并且 Service、Pod 正常。

---

# 10. `Service.selector.app=html-demo` 是什么意思

这一段：

```yaml
selector:
  app: html-demo
```

意思是：

```text
这个 Service 要寻找所有标签为 app=html-demo 的 Pod
```

只要 Pod 上有这个标签：

```text
app=html-demo
```

Service 就会把它加入自己的后端服务列表。

---

## 10.1 Service 如何找到 Pod

Deployment 创建 Pod 时，Pod 有标签：

```yaml
labels:
  app: html-demo
```

Service 里写：

```yaml
selector:
  app: html-demo
```

于是 Service 会找到这些 Pod。

---

## 10.2 图解

```text
Service: html-demo-svc
selector:
  app=html-demo
        │
        │ 查找标签
        ▼

┌─────────────────────┐
│ Pod 1               │
│ labels:             │
│   app=html-demo     │
└─────────────────────┘

┌─────────────────────┐
│ Pod 2               │
│ labels:             │
│   app=html-demo     │
└─────────────────────┘

┌─────────────────────┐
│ Pod 3               │
│ labels:             │
│   app=html-demo     │
└─────────────────────┘
```

---

## 10.3 如果 Service selector 写错会怎样

错误示例：

Deployment 创建的 Pod 标签是：

```yaml
labels:
  app: html-demo
```

但是 Service 写成：

```yaml
selector:
  app: nginx
```

结果：

```text
Service 找不到任何 Pod
```

查看 Service：

```bash
kubectl describe svc html-demo-svc
```

可能看到：

```text
Endpoints: <none>
```

这说明 Service 没有找到后端 Pod。

---

# 11. `port`、`targetPort`、`nodePort` 的区别

这一段是 Service 中最容易混淆的：

```yaml
ports:
- port: 80
  targetPort: 80
  nodePort: 32056
```

---

## 11.1 `nodePort`

```yaml
nodePort: 32056
```

这是节点上开放的端口。

外部访问时用它：

```bash
curl http://节点IP:32056
```

例如：

```bash
curl http://192.168.56.101:32056
```

---

## 11.2 `port`

```yaml
port: 80
```

这是 Service 自己的端口。

集群内部可以通过：

```text
html-demo-svc:80
```

访问该 Service。

---

## 11.3 `targetPort`

```yaml
targetPort: 80
```

这是 Pod 容器内部真正监听的端口。

比如 Nginx 默认监听 80，所以这里写：

```yaml
targetPort: 80
```

如果你的容器应用监听 8080，那么这里应该写：

```yaml
targetPort: 8080
```

---

## 11.4 三者关系图

```text
外部用户
   │
   │ 访问 NodeIP:32056
   ▼
nodePort: 32056
   │
   ▼
Service port: 80
   │
   ▼
targetPort: 80
   │
   ▼
Pod 中的 Nginx 容器端口 80
```

---

## 11.5 一句话理解

```text
nodePort：外部访问节点的端口
port：Service 自己的端口
targetPort：Pod 容器真正监听的端口
```

---

# 12. Deployment 和 Service 的完整关系图

完整关系如下：

```text
Deployment: html-demo
        │
        │ replicas=3
        ▼

创建 3 个 Pod
        │
        │ 每个 Pod 都带标签
        ▼

Pod-1 app=html-demo
Pod-2 app=html-demo
Pod-3 app=html-demo


Service: html-demo-svc
        │
        │ selector:
        │   app=html-demo
        ▼

找到这 3 个 Pod
        │
        ▼

把流量负载均衡到 Pod
```

---

## 12.1 更完整图解

```text
┌──────────────────────────────┐
│ Deployment: html-demo        │
│                              │
│ replicas: 3                  │
│ selector:                    │
│   matchLabels:               │
│     app: html-demo           │
│                              │
│ template:                    │
│   labels:                    │
│     app: html-demo           │
└───────────────┬──────────────┘
                │
                │ 创建 Pod
                ▼

┌──────────────────────────────┐
│ Pod 1                        │
│ labels: app=html-demo        │
│ container: nginx:80          │
└──────────────────────────────┘

┌──────────────────────────────┐
│ Pod 2                        │
│ labels: app=html-demo        │
│ container: nginx:80          │
└──────────────────────────────┘

┌──────────────────────────────┐
│ Pod 3                        │
│ labels: app=html-demo        │
│ container: nginx:80          │
└──────────────────────────────┘


┌──────────────────────────────┐
│ Service: html-demo-svc       │
│ type: NodePort               │
│ selector: app=html-demo      │
│ port: 80                     │
│ targetPort: 80               │
│ nodePort: 32056              │
└───────────────┬──────────────┘
                │
                │ 通过 selector 找 Pod
                ▼

Pod 1 / Pod 2 / Pod 3
```

---

# 13. 外部访问流程图

假设你的节点 IP 是：

```text
master: 192.168.56.101
worker: 192.168.56.102
```

Service 是：

```yaml
nodePort: 32056
```

那么你可以访问：

```bash
curl http://192.168.56.101:32056
```

或者：

```bash
curl http://192.168.56.102:32056
```

---

## 13.1 流量完整路径

```text
浏览器 / curl
   │
   │ http://192.168.56.101:32056
   ▼
Kubernetes Node
   │
   │ NodePort: 32056
   ▼
Service: html-demo-svc
   │
   │ port: 80
   ▼
Endpoints
   │
   ├── Pod 1:80
   ├── Pod 2:80
   └── Pod 3:80
        │
        ▼
Nginx 返回网页
```

---

# 14. 常用排查命令

## 14.1 查看 Deployment

```bash
kubectl get deployment
```

查看详细信息：

```bash
kubectl describe deployment html-demo
```

---

## 14.2 查看 Pod

```bash
kubectl get pods -o wide
```

查看 Pod 标签：

```bash
kubectl get pods --show-labels
```

查看 Pod 详细信息：

```bash
kubectl describe pod <pod-name>
```

---

## 14.3 查看 Service

```bash
kubectl get svc
```

查看 Service 详细信息：

```bash
kubectl describe svc html-demo-svc
```

重点看：

```text
Selector:
Endpoints:
Port:
TargetPort:
NodePort:
```

---

## 14.4 查看 Endpoints

```bash
kubectl get endpoints
```

或者：

```bash
kubectl get ep
```

如果看到：

```text
html-demo-svc   <none>
```

说明 Service 没找到 Pod。

常见原因是：

```text
Service.selector 和 Pod.labels 不一致
```

---

## 14.5 查看标签匹配情况

查看所有带 `app=html-demo` 标签的 Pod：

```bash
kubectl get pods -l app=html-demo
```

如果有结果，说明标签存在。

如果没结果，说明 Pod 标签不对。

---

# 15. 常见错误

## 15.1 Service 访问失败

先检查：

```bash
kubectl get svc
kubectl get pods -o wide
kubectl describe svc html-demo-svc
```

重点看 Service 的 Endpoints。

如果是：

```text
Endpoints: <none>
```

说明 Service 没有匹配到 Pod。

---

## 15.2 selector 写错

Deployment：

```yaml
template:
  metadata:
    labels:
      app: html-demo
```

Service：

```yaml
selector:
  app: nginx
```

这就错了。

应该改成：

```yaml
selector:
  app: html-demo
```

---

## 15.3 targetPort 写错

如果容器实际监听 80：

```yaml
targetPort: 80
```

如果写成：

```yaml
targetPort: 8080
```

而容器没有监听 8080，访问就会失败。

---

## 15.4 nodePort 范围错误

NodePort 默认范围通常是：

```text
30000-32767
```

所以：

```yaml
nodePort: 32056
```

是合法的。

但如果写：

```yaml
nodePort: 80
```

一般会失败。

---

## 15.5 Pod 没有运行

查看：`kubectl get pods`

如果不是：`Running`

就需要查看原因：

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

---

# 16. 最终记忆口诀

## 16.1 Deployment

```text
Deployment 负责创建和管理 Pod
replicas 控制 Pod 数量
template 描述 Pod 长什么样
template.metadata.labels 给 Pod 贴标签
selector.matchLabels 决定 Deployment 管理哪些 Pod
```

---

## 16.2 Service

```text
Service 负责给 Pod 提供稳定访问入口
Service.selector 通过标签查找 Pod
port 是 Service 端口
targetPort 是 Pod 容器端口
nodePort 是节点暴露给外部访问的端口
```

---

## 16.3 三个关键标签关系

```text
Deployment.selector.matchLabels
        ↓
Deployment.template.metadata.labels
        ↓
Service.selector
```

这三个地方通常要保持一致：

```yaml
app: html-demo
```

---

## 16.4 最核心的一句话

```text
Deployment 负责创建 Pod，
Pod 通过 labels 被打标签，
Service 通过 selector 找到这些 Pod，
然后把外部流量转发给 Pod。
```

---

# 附录：推荐部署命令

保存 Deployment：

```bash
kubectl apply -f deployment.yaml
```

保存 Service：

```bash
kubectl apply -f service.yaml
```

查看资源：

```bash
kubectl get deployment
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints
```

访问 NodePort：

```bash
curl http://192.168.56.101:32056
```

删除资源：

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```
