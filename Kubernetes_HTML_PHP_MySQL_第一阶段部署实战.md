# Kubernetes 企业实战（第一阶段）

## 从 php-base、nginx-base 到 HTML + PHP + MySQL 项目部署

> 本文档整理自当前实战过程，仅包含已经讨论并确认过的内容。

---

# 文档目录

1. Harbor 仓库规划
2. php-base
3. php-custom.ini
4. 构建 php-base
5. nginx-base
6. 构建 nginx-base
7. MySQL
8. 项目目录规划
9. Docker Build Context
10. 制作 store-demo PHP 镜像
11. 制作 store-demo Nginx 镜像
12. 创建 PHP Deployment
13. 创建 Nginx Deployment
14. 部署项目

---

# 1 Harbor 仓库规划

Harbor：

```text
111.229.217.116:30887
```

建议：

```text
base
├── php-base
└── nginx-base

store-demo
├── php
└── nginx
```

基础镜像：

```text
111.229.217.116:30887/base/php-base:8.2
111.229.217.116:30887/base/nginx-base:1.0
```

项目镜像：

```text
111.229.217.116:30887/store-demo/php:v1
111.229.217.116:30887/store-demo/nginx:v1
```

---

# 2 php-base

目录：

```bash
mkdir -p /app/images/php-base
cd /app/images/php-base
```

Dockerfile：

```dockerfile
FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    git unzip zip curl wget vim procps \
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libwebp-dev \
    libonig-dev \
    libxml2-dev \
    libssl-dev \
    libcurl4-openssl-dev \
    libicu-dev \
    libxslt1-dev \
    libmagickwand-dev \
    libmemcached-dev \
    zlib1g-dev \
    libsqlite3-dev \
    libpq-dev \
&& docker-php-ext-configure gd \
    --with-freetype \
    --with-jpeg \
    --with-webp \
&& docker-php-ext-install -j$(nproc) \
    pdo pdo_mysql mysqli mbstring bcmath gd zip opcache intl \
    xsl soap sockets pcntl exif calendar gettext shmop \
    sysvmsg sysvsem sysvshm xml simplexml dom curl fileinfo ftp \
&& pecl install redis \
&& pecl install memcached \
&& pecl install imagick \
&& docker-php-ext-enable redis memcached imagick opcache \
&& apt-get clean \
&& rm -rf /var/lib/apt/lists/*

COPY php-custom.ini /usr/local/etc/php/conf.d/99-custom.ini
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

CMD ["php-fpm"]
```

---

# 3 php-custom.ini

```ini
date.timezone = Asia/Shanghai

memory_limit = 512M
upload_max_filesize = 100M
post_max_size = 100M

max_execution_time = 300
max_input_time = 300

display_errors = Off
log_errors = On
error_reporting = E_ALL

disable_functions =

opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 1
opcache.revalidate_freq = 2
```

Dockerfile：

```dockerfile
COPY php-custom.ini /usr/local/etc/php/conf.d/99-custom.ini
```

说明：PHP 启动时会自动加载 `/usr/local/etc/php/conf.d/*.ini`，`php-custom.ini` 只覆盖自己配置的项，未配置项继续使用默认值。

---

# 4 构建 php-base

```bash
docker build -t php-base:8.2 .

docker tag php-base:8.2 \
111.229.217.116:30887/base/php-base:8.2

docker push \
111.229.217.116:30887/base/php-base:8.2
```

---

# 5 nginx-base

目录：

```bash
mkdir -p /app/images/nginx-base
cd /app/images/nginx-base
```

Dockerfile：

```dockerfile
FROM nginx:1.26

RUN rm -f /etc/nginx/conf.d/default.conf

WORKDIR /var/www/html

CMD ["nginx","-g","daemon off;"]
```

---

# 6 构建 nginx-base

```bash
docker build -t nginx-base:1.0 .

docker tag nginx-base:1.0 \
111.229.217.116:30887/base/nginx-base:1.0

docker push \
111.229.217.116:30887/base/nginx-base:1.0
```

---

# 7 MySQL

继续使用已有：

```text
Deployment：mysql
Service：mysql-service
```

PHP：

```php
$pdo = new PDO(
    "mysql:host=mysql-service;dbname=test",
    "root",
    "123456"
);
```

初始化：

```bash
kubectl exec -i mysql-pod-name -- \
mysql -uroot -p123456 test < init.sql
```

---

# 8 项目目录

```text
/app/store-demo-docker
├── Dockerfile.php
├── Dockerfile.nginx
├── nginx
│   └── default.conf
├── web
│   ├── public
│   ├── application
│   ├── config
│   └── ...
└── k8s
    ├── php.yaml
    └── nginx.yaml
```

---

# 9 Docker Build Context

构建：

```bash
cd /app/store-demo-docker

docker build -f Dockerfile.php -t store-demo-php:v1 .
```

不要：

```dockerfile
COPY /app/store-demo-docker/web /var/www/html
```

应该：

```dockerfile
COPY web/ /var/www/html/
```

不要：

```dockerfile
COPY . /var/www/html
```

---

# 10 PHP 项目镜像

Dockerfile.php：

```dockerfile
FROM 111.229.217.116:30887/base/php-base:8.2

COPY web/ /var/www/html/

WORKDIR /var/www/html

CMD ["php-fpm"]
```

构建：

```bash
docker build -f Dockerfile.php -t store-demo-php:v1 .

docker tag store-demo-php:v1 \
111.229.217.116:30887/store-demo/php:v1

docker push \
111.229.217.116:30887/store-demo/php:v1
```

---

# 11 Nginx 项目镜像

default.conf：

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;
    index index.php index.html;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        fastcgi_pass store-demo-php-service:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Dockerfile.nginx：

```dockerfile
FROM 111.229.217.116:30887/base/nginx-base:1.0

COPY nginx/default.conf /etc/nginx/conf.d/default.conf
COPY web/ /var/www/html/

WORKDIR /var/www/html
```

构建：

```bash
docker build -f Dockerfile.nginx -t store-demo-nginx:v1 .

docker tag store-demo-nginx:v1 \
111.229.217.116:30887/store-demo/nginx:v1

docker push \
111.229.217.116:30887/store-demo/nginx:v1
```

---

# 12 PHP Deployment

文件：`k8s/php.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-demo-php
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-demo-php
  template:
    metadata:
      labels:
        app: store-demo-php
    spec:
      containers:
      - name: php
        image: 111.229.217.116:30887/store-demo-php/store-demo-php:1.0
        ports:
        - containerPort: 9000

        env:
        - name: DB_HOST
          value: mysql-service
    
        - name: DB_PORT
          value: "3306"
    
        - name: DB_DATABASE
          value: store
    
        - name: DB_USERNAME
          value: root
    
        - name: DB_PASSWORD
          value: "123456"
---
apiVersion: v1
kind: Service
metadata:
  name: store-demo-php-service
spec:
  selector:
    app: store-demo-php
  ports:
  - port: 9000
    targetPort: 9000
```

（使用我们讨论的 Deployment + Service 配置）

---

# 13 Nginx Deployment

文件：`k8s/nginx.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-demo-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-demo-nginx
  template:
    metadata:
      labels:
        app: store-demo-nginx
    spec:
      containers:
      - name: nginx
        image: 111.229.217.116:30887/store-demo-nginx/store-demo-nginx:1.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: store-demo-nginx-service
spec:
  type: NodePort
  selector:
    app: store-demo-nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31193
```

（使用我们讨论的 Deployment + NodePort Service 配置）

---

# 14 部署项目

```bash
kubectl apply -f k8s/php.yaml
kubectl apply -f k8s/nginx.yaml
```

浏览器访问：

```text
http://192.168.56.11:30081
或者
http://192.168.56.12:30081
```
