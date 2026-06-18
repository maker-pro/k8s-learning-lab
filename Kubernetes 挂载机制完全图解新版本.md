# Kubernetes 挂载机制完全图解（ConfigMap + Volume + volumeMounts + subPath）

> 适合对象：
>
> * Kubernetes 初学者
> * 使用 Nginx + ConfigMap 挂载 HTML 页面
> * 经常搞不清 Volume、ConfigMap、mountPath、subPath 的人

---

# 目录

1. Kubernetes 挂载到底是什么
2. ConfigMap 是什么
3. Volume 是什么
4. Volume 和 ConfigMap 的关系
5. volumeMounts 是什么
6. volumes 和 volumeMounts 的关系
7. mountPath 是什么
8. subPath 是什么
9. 为什么 mountPath 是文件而不是目录
10. ConfigMap → Volume → Container 的完整流程
11. ConfigMap 更新为什么页面没变
12. subPath 的坑
13. ConfigMap 更新标准流程
14. Deployment 更新标准流程
15. 实战案例：Nginx 首页挂载
16. 常见错误排查
17. 面试高频问题

---

# 一、Kubernetes 挂载到底是什么

很多人第一次学习 Kubernetes 时会以为：

```text
本地文件
    ↓
直接进入容器
```

实际上 Kubernetes 的挂载流程是：

```text
ConfigMap
    ↓
Volume
    ↓
volumeMounts
    ↓
Container
```

即：

```text
ConfigMap
    ↓
生成 Volume
    ↓
挂载到容器
```

---

# 二、ConfigMap 是什么

ConfigMap 可以理解成：

```text
配置中心
```

例如：

```bash
kubectl create configmap html-config \
  --from-file=index.html
```

创建后：

```yaml
data:
  index.html: |
    Hello Kubernetes
```

这里：

```text
Key = index.html
Value = 文件内容
```

---

# 三、Volume 是什么

很多初学者认为：

```text
Volume = 磁盘
```

其实不准确。

对于 ConfigMap 场景：

```text
Volume
=
ConfigMap 的运行时映射
```

例如：

```yaml
volumes:
- name: html-volume

  configMap:
    name: html-config
```

表示：

```text
创建一个 Volume

名字：
html-volume

数据来源：
html-config
```

图解：

```text
ConfigMap
│
└── index.html
      │
      ▼

Volume
│
└── index.html
```

---

# 四、volumeMounts 是什么

Volume 只是创建了。

真正让容器使用它的是：

```yaml
volumeMounts:
- name: html-volume
```

作用：

```text
把 Volume 挂到容器中
```

---

# 五、volumes 和 volumeMounts 的关系

非常重要。

```yaml
volumes:
- name: html-volume
```

表示：

```text
定义 Volume
```

---

```yaml
volumeMounts:
- name: html-volume
```

表示：

```text
使用 Volume
```

图解：

```text
volumes
    │
    ▼
html-volume
    │
    ▼
volumeMounts
```

---

# 六、mountPath 是什么

例如：

```yaml
mountPath:
/usr/share/nginx/html/index.html
```

含义：

```text
容器内部最终路径
```

注意：

```text
不是宿主机路径
不是 Kubernetes 节点路径
而是容器内部路径
```

---

# 七、subPath 是什么

例如：

```yaml
subPath: index.html
```

含义：

```text
从 Volume 中取出哪个文件
```

图解：

```text
Volume

├── index.html
├── a.txt
├── b.txt
└── c.txt
```

---

```yaml
subPath: index.html
```

表示：

```text
只取 index.html
```

---

# 八、为什么 mountPath 是文件

很多人疑惑：

```yaml
mountPath:
/usr/share/nginx/html/index.html

subPath:
index.html
```

为什么两个地方都写文件？

其实：

```text
subPath
=
源文件

mountPath
=
目标文件
```

图解：

```text
Volume

index.html
     │
     ▼

subPath

     │
     ▼

/usr/share/nginx/html/index.html

mountPath
```

---

# 九、ConfigMap 更新为什么页面没变

很多人会这样做：

```text
修改本地 index.html
```

然后刷新页面。

结果：

```text
页面没变化
```

原因：

```text
ConfigMap 是副本
```

关系：

```text
本地文件
    │
    ▼

ConfigMap
    │
    ▼

Pod
```

修改本地文件不会自动修改 ConfigMap。

---

# 十、如何更新 ConfigMap

推荐：

```bash
kubectl create configmap html-config \
  --from-file=index.html \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

作用：

```text
重新读取本地文件
覆盖集群中的 ConfigMap
```

---

# 十一、subPath 的经典坑

如果使用：

```yaml
subPath: index.html
```

即使：

```text
ConfigMap 更新成功
```

Pod 中的文件也不会自动刷新。

这是 Kubernetes 官方限制。

---

# 十二、正确更新流程

修改：

```text
index.html
```

执行：

```bash
kubectl create configmap html-config \
  --from-file=index.html \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

然后：

```bash
kubectl rollout restart deployment html-demo
```

最后：

```bash
kubectl get pods
```

等待 Pod 重建。

---

# 十三、Deployment 更新流程

修改：

```yaml
deployment.yaml
```

执行：

```bash
kubectl apply -f deployment.yaml
```

即可。

Deployment 会自动 Rolling Update。

---

# 十四、最核心的关系图

```text
ConfigMap
│
└── index.html
      │
      ▼

Volume
html-volume
      │
      ▼

volumeMounts
      │
      ▼

Container
/usr/share/nginx/html/index.html
```

---

# 十五、最终记忆口诀

ConfigMap：

```text
存配置
```

Volume：

```text
存数据来源
```

volumeMounts：

```text
挂载数据
```

subPath：

```text
取哪个文件
```

mountPath：

```text
挂到哪里
```

Deployment：

```text
创建 Pod
```

Service：

```text
访问 Pod
```

最重要的一句话：

```text
ConfigMap 提供数据
Volume 保存数据
volumeMounts 使用数据
subPath 选择数据
mountPath 放置数据
```
