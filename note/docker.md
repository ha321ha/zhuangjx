# WSL2 Ubuntu 内安装 Docker Engine（开源免费）

> 前置条件：Windows 11 专业版 + WSL2 Ubuntu 已安装就绪
> 最后更新：2026-07-08

---

## 1. 进入 WSL Ubuntu

```powershell
# Windows PowerShell 中执行
wsl
```

进入 Ubuntu 终端后，以下所有命令都在此终端内执行。

---

## 2. 安装 Docker Engine

### 2.1 卸载旧版本（如有）

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### 2.2 添加 Docker 官方 APT 源

```bash
# 安装依赖
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# 添加 GPG 密钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 添加源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> 如果 `$VERSION_CODENAME` 为空，改用手动指定或使用 `lsb_release -cs`：
>
> ```bash
> echo \
>   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
>   $(lsb_release -cs) stable" | \
>   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
> ```

### 2.3 安装 Docker Engine 及插件

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> 各组件说明：
> - `docker-ce` — Docker Engine 守护进程
> - `docker-ce-cli` — `docker` 命令行工具
> - `containerd.io` — 容器运行时
> - `docker-buildx-plugin` — 多架构构建插件
> - `docker-compose-plugin` — `docker compose` 编排工具

### 2.4 启动 Docker 引擎

```bash
sudo service docker start
```

### 2.5 设置 WSL 启动时自动启动

```bash
echo 'sudo service docker start' >> ~/.bashrc
```

---

## 3. 配置免 sudo 运行

### 3.1 将当前用户加入 docker 组

```bash
sudo usermod -aG docker $USER
```

### 3.2 使组变更生效

退出 WSL 再重新进入：

```powershell
# Windows PowerShell 中执行
wsl --terminate Ubuntu
wsl
```

### 3.3 验证免 sudo

```bash
docker ps
```

不再报权限错误即为成功。

---

## 4. 验证安装

```bash
# 查看版本（必须有 Client 和 Server 两部分）
docker version

# 运行测试容器
docker run hello-world

# 查看系统信息
docker info
```

---

## 5. 常用管理命令

```bash
# 启动/停止/重启
sudo service docker start
sudo service docker stop
sudo service docker restart

# 查看状态
sudo service docker status

# 更新
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# 磁盘清理
docker system prune -a
docker system df
```

---

## 6. 常见问题

### Docker daemon 连不上

```bash
sudo service docker start
sudo service docker status
```

### 代理问题（公司网络）

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://你的代理:端口"
Environment="HTTPS_PROXY=http://你的代理:端口"
Environment="NO_PROXY=localhost,127.0.0.1,.local"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 重启 Windows 后 Docker 没运行

打开 WSL 终端即可（`~/.bashrc` 中配置了自动启动）。或手动执行：

```bash
sudo service docker start
```
