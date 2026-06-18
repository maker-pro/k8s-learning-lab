# Kubernetes 挂载机制完全图解

> 适合对象：
>
> * Kubernetes 初学者
> * 刚接触 ConfigMap 挂载的人
> * 经常搞混 Volume、VolumeMount、mountPath、subPath 的人
> * 后续准备学习 Secret、PVC、NFS 的人

---

# 第一章 为什么需要挂载

## 容器的问题

假设我们启动一个 nginx：

```yaml
containers:
- name: nginx
  image: nginx
```

容器内部：

```text
/usr/share/nginx/html/index.html
```

默认内容：

```html
Welcome to nginx!
```

如果我们想修改首页：

```html
Hello Kubernetes
```

有几种办法：

### 方法1：进入容器修改

```bash
kubectl exec -it pod-name -- sh
vi /usr/share/nginx/html/index.html
```

问题：

```text
Pod重建后内容丢失
```

不推荐。

---

### 方法2：重新制作镜像

```dockerfile
FROM nginx

COPY index.html /usr/share/nginx/html/index.html
```

问题：

```text
改一个HTML
重新构建镜像
重新推送镜像
重新部署
```

太重。

---

### 方法3：ConfigMap挂载

```text
ConfigMap
      │
      ▼
Volume
      │
      ▼
Container
```

这是 Kubernetes 推荐方案。

---

# 第二章 Volume 到底是什么

很多人认为：

```text
Volume = 磁盘
```

其实不准确。

---

## Kubernetes视角

Volume 是：

```text
Pod级别的存储对象
```

图解：

```text
Pod
│
├── Container-A
│
├── Container-B
│
└── Volume
```

---

Volume 可以来自：

```text
ConfigMap
Secret
emptyDir
HostPath
PVC
NFS
```

所以：

```text
Volume
≠ ConfigMap

Volume
≠ PVC

Volume
是统一抽象
```

---

# 第三章 Volume 与 VolumeMount

最容易混淆的两个概念：

```yaml
volumes:
```

和

```yaml
volumeMounts:
```

---

## volumes

作用：

```text
定义存储来源
```

例如：

```yaml
volumes:
- name: html-volume

  configMap:
    name: html-config
```

翻译：

```text
创建一个 Volume

名字：
html-volume

数据来源：
html-config
```

---

## volumeMounts

作用：

```text
使用 Volume
```

例如：

```yaml
volumeMounts:
- name: html-volume
```

翻译：

```text
我要使用 html-volume
```

---

图解：

```text
ConfigMap
     │
     ▼

Volume
html-volume
     │
     ▼

Container
```

---

# 第四章 name 是如何关联的

很多人疑惑：

```yaml
volumeMounts:
- name: html-volume
```

和

```yaml
volumes:
- name: html-volume
```

为什么名字一样？

---

因为：

```text
volumeMounts.name
引用
volumes.name
```

就像变量名。

---

图解：

```text
volumes
│
└── html-volume
       ▲
       │
       │ 引用
       │
volumeMounts
```

---

# 第五章 mountPath 到底是什么

例如：

```yaml
mountPath:
/usr/share/nginx/html/index.html
```

很多人以为：

```text
宿主机路径
```

错误。

---

正确理解：

```text
容器内部路径
```

---

图解：

```text
Node
│
└── Pod
     │
     └── nginx
          │
          └── /usr/share/nginx/html/index.html
```

mountPath 指向的是：

```text
nginx容器内部
```

---

# 第六章 subPath 到底是什么

例如：

```yaml
subPath: index.html
```

作用：

```text
从Volume中取哪个文件
```

---

图解：

```text
Volume

├── index.html
├── a.txt
├── b.txt
└── c.txt
```

指定：

```yaml
subPath: index.html
```

表示：

```text
只取 index.html
```

---

# 第七章 为什么 mountPath 指向文件

很多人都会问：

```text
subPath已经指定文件

为什么mountPath
还要写文件？
```

---

因为：

```text
subPath
指定源文件

mountPath
指定目标文件
```

---

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
```

---

# 第八章 ConfigMap 文件改名会怎样

假设：

```text
index.html
```

改成：

```text
test.html
```

---

ConfigMap：

```yaml
data:
  test.html: |
```

---

那么：

```yaml
subPath: index.html
```

会失败。

---

必须改：

```yaml
subPath: test.html
```

---

但是：

```yaml
mountPath:
/usr/share/nginx/html/index.html
```

可以不改。

---

最终：

```text
test.html
     │
     ▼

index.html
```

---

# 第九章 subPath 的经典坑

如果：

```yaml
subPath: index.html
```

---

修改 ConfigMap：

```bash
kubectl edit configmap
```

---

Pod中的文件：

```text
不会自动更新
```

---

需要：

```bash
kubectl rollout restart deployment
```

---

# 第十章 最终记忆口诀

volumes

```text
数据从哪里来
```

---

volumeMounts

```text
谁来使用数据
```

---

mountPath

```text
挂到容器哪里
```

---

subPath

```text
Volume里面取哪个文件
```

---

一句话总结：

从 ConfigMap 中取出某个文件（subPath），挂载到容器中的某个位置（mountPath）。
