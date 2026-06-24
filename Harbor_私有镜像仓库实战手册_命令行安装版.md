# Harbor 私有镜像仓库实战手册（命令行安装版）

## 目录

1. Harbor 简介
2. Harbor 与 Docker Hub
3. Harbor 与 Registry
4. 推荐架构
5. 腾讯云命令行安装 Harbor
6. Harbor 常见安装问题
7. 创建 Harbor 项目
8. HTML 项目打包镜像
9. 上传镜像到 Harbor
10. Kubernetes 使用 Harbor 镜像
11. imagePullSecret 配置
12. 常见故障排查

---

# 一、Harbor 简介

Harbor 是企业级私有镜像仓库。

作用：

- 存储镜像
- 用户管理
- 项目管理
- 权限控制
- 镜像扫描

流程：
```
代码
↓
Docker Build
↓
Harbor
↓
Kubernetes
```
---

# 二、Harbor 与 Docker Hub

Docker Hub：

- 公共仓库
- 官方镜像

Harbor：

- 私有仓库
- 企业内部使用

---

# 三、Harbor 与 Registry

Registry：

优点：

- 部署简单
- 一个镜像即可运行

缺点：

- 无界面
- 无权限管理

Harbor：

优点：

- Web UI
- 用户管理
- 项目管理
- 镜像扫描

---

# 四、推荐架构

腾讯云服务器：
```
Harbor
↓

本地 Kubernetes

master
worker

流程：

Docker Build
↓
Harbor
↓
Kubernetes
```
---

# 五、腾讯云命令行安装 Harbor

## 服务器建议

CPU：2核以上

内存：4GB以上

磁盘：40GB以上

## 安装 Docker

```bash
apt update

apt install docker.io -y

systemctl enable docker
systemctl start docker
```

检查：

```bash
docker version
```

## 安装 Docker Compose

```bash
apt install docker-compose-plugin -y
```

检查：

```bash
docker compose version
```

## 下载 Harbor

在线安装版：

```bash
cd /opt

wget https://github.com/goharbor/harbor/releases/download/v2.14.0/harbor-online-installer-v2.14.0.tgz

tar -zxvf harbor-online-installer-v2.14.0.tgz

cd harbor
```

离线安装版（推荐国内环境）：

```bash
cd /opt

wget https://github.com/goharbor/harbor/releases/download/v2.14.0/harbor-offline-installer-v2.14.0.tgz

tar -zxvf harbor-offline-installer-v2.14.0.tgz

cd harbor
```

## 配置 Harbor

```bash
cp harbor.yml.tmpl harbor.yml

vim harbor.yml
```

修改：

```yaml
hostname: 公网IP

http:
  port: 8080

harbor_admin_password: Harbor123456
```

如果没有 SSL：

注释 https 配置。

## 安装

```bash
./install.sh
```

查看：

```bash
docker ps
```

出现：

```text
harbor-core
harbor-db
harbor-jobservice
harbor-portal
harbor-registry
```

表示安装成功。

<b style="color:red;">如果后续有修改 harbor.yml 文件
需要重新生成，执行如下操作：</b>
```bash
./prepare
docker compose down
docker compose up -d
```

访问：

```text
http://公网IP:8080
```

默认账号：

```text
admin
```

密码 `harbor_admin_password` 的值 : <b style="color:red;">Harbor123456</b>：

```text
Harbor123456
```

---

# 六、Harbor 常见安装问题

## 80端口冲突

错误：

```text
failed to bind host port 80
```

解决：

```yaml
http:
  port: 8080
```

## prepare 镜像拉取失败

错误：

```text
goharbor/prepare:v2.x.x
```

原因：

镜像源异常。

检查：

```bash
docker pull goharbor/prepare:v2.10.0
```

## env not found

错误：

```text
common/config/core/env not found
```

原因：

prepare 没执行成功。

---

# 七、创建 Harbor 项目

登录 Harbor：

Projects
↓
New Project

创建：

```text
web-demo
```

建议：

```text
Public
```

---

# 八、HTML 项目打包镜像

项目结构：

```text
/ app/web_demo/web

├── index.html
├── css
├── js
└── images
```

创建 Dockerfile：

```dockerfile
FROM nginx:latest

COPY . /usr/share/nginx/html

EXPOSE 80
```

说明：

```dockerfile
COPY . /usr/share/nginx/html
```

把整个 HTML 项目复制到 Nginx 默认网站目录。

构建镜像：

```bash
cd /app/web_demo/web

docker build -t web-demo:v1 .
```

查看：

```bash
docker images
```

测试：

```bash
docker run -d \
  --name web-demo \
  -p 8080:80 \
  web-demo:v1
```

访问：

```text
http://服务器IP:8080
```

---

# 九、上传镜像到 Harbor

假设 Harbor：

```text
43.x.x.x:8080
```

项目：

```text
web-demo
```

登录：

```bash
docker login 43.x.x.x:8080
```

打标签：

```bash
docker tag web-demo:v1 \
43.x.x.x:8080/web-demo/web-demo:v1
```

上传：

```bash
docker push \
43.x.x.x:8080/web-demo/web-demo:v1
```

刷新 Harbor 页面即可看到镜像。

---

# 十、Kubernetes 使用 Harbor 镜像

Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web-demo

spec:

  replicas: 2

  selector:
    matchLabels:
      app: web-demo

  template:
    metadata:
      labels:
        app: web-demo

    spec:
      containers:

      - name: web-demo

        image: 43.x.x.x:8080/web-demo/web-demo:v1

        imagePullPolicy: IfNotPresent

        ports:
        - containerPort: 80
```

部署：

```bash
kubectl apply -f deployment.yaml
```

---

# 十一、imagePullSecret

Harbor 私有项目：

创建 Secret：

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=43.x.x.x:8080 \
  --docker-username=admin \
  --docker-password='密码'
```

Deployment：

```yaml
template:
  spec:

    imagePullSecrets:
    - name: harbor-secret

    containers:
    - name: web-demo
```

---

# 十二、常见故障排查

查看 Pod：

```bash
kubectl get pod -o wide
```

查看事件：

```bash
kubectl describe pod POD_NAME
```

常见错误：

```text
ErrImagePull
ImagePullBackOff
```

检查：

- Harbor 是否可访问
- 镜像是否存在
- 镜像路径是否正确
- imagePullSecret 是否正确

---

如果创建pod的时候镜像一直拉去不下来，

第一步、需要确定/etc/containerd/config.toml 中的config_path的值
如果是`config_path = ""`，就要改成 `config_path = "/etc/containerd/certs.d"`

然后重启`containerd`、`kubelet`
```shell
systemctl restart containerd
systemctl restart kubelet
```

第二步、需要在`/etc/containerd/certs.d/`创建一个对应的文件夹（比如我的harbor 地址是：http://111.229.217.116:30887/， 就需要创建一个111.229.217.116:30887文件夹），并在创建的文件夹下创建一个新的文件hosts.toml

hosts.toml内容:
```toml
server = "http://111.229.217.116:30887"

[host."http://111.229.217.116:30887"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

然后重启`containerd`
```shell
systemctl restart containerd
```

---

# 企业推荐流程

```
代码
↓
Dockerfile
↓
docker build
↓
docker push Harbor
↓
kubectl apply
↓
Pod Running
```