## kubeadm + containerd + flannel 集群网络问题

### 第一步
> 在 master 和 worker 都执行：
```
sudo mkdir -p /etc/containerd/certs.d/docker.io

sudo tee /etc/containerd/certs.d/docker.io/hosts.toml <<'EOF'
server = "https://docker.io"

[host."https://docker.1ms.run"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve"]
EOF
```

### 第二步
> 然后确认 containerd 开启了 certs.d 配置。`sudo grep -n "config_path" /etc/containerd/config.toml`
> 
> 如果没有文件`/etc/containerd/config.toml`，执行：
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

然后修改：`sudo vim /etc/containerd/config.toml`

找到：`[plugins.'io.containerd.cri.v1.images'.registry]`

在下面确保有：`config_path = "/etc/containerd/certs.d"` (如果没有，或者是空，就用这个替换)


### 第三步
最后重启：
```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```
测试拉镜像：
```
sudo ctr -n k8s.io images pull docker.io/library/nginx:latest` 或者 `sudo crictl pull nginx
```