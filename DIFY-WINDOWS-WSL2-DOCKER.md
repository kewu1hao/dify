# Windows + WSL2 + Docker Desktop 部署 Dify

本文介绍在 Windows 本机上安装 WSL2 和 Ubuntu，使用 Windows 版 Docker Desktop，再通过 Docker Compose 部署 Dify 的完整过程。

## 一、部署架构

```text
Windows
├── WSL2
│   └── Ubuntu
│       └── Dify 项目文件和 Docker Compose 命令
└── Docker Desktop
    └── Docker Engine（WSL2 后端）
        └── Dify 容器
```

> Docker 安装在 Windows 的 Docker Desktop 中，Ubuntu 只提供 Linux 环境和命令行。不要在 Ubuntu 内再次安装 `docker-ce`。

## 二、准备条件

建议配置：

| 项目 | 最低要求 | 推荐配置 |
| --- | --- | --- |
| Windows | Windows 10 2004 及以上或 Windows 11 | Windows 11 |
| CPU | 2 核 | 4 核或以上 |
| 内存 | 8 GB | 16 GB |
| 可用磁盘 | 30 GB | 50–100 GB |
| 网络 | 能访问 GitHub、Docker Hub | 稳定网络 |

Dify 官方 Docker 部署最低约需要 2 个 CPU 核心、4 GB 内存以及 Docker Compose 2.24.0 或更高版本。实际使用建议为 Docker Desktop 分配至少 8 GB 内存。

## 三、安装 WSL2 和 Ubuntu

### 1. 以管理员身份打开 PowerShell

在 Windows 开始菜单中搜索“终端”或“PowerShell”，选择“以管理员身份运行”，执行：

```powershell
wsl --install -d Ubuntu-24.04
```

完成后重启 Windows。

### 2. 初始化 Ubuntu

从开始菜单打开 `Ubuntu 24.04 LTS`，按提示创建 Linux 用户和密码。

输入密码时终端不会显示字符，这是正常现象。

### 3. 检查 WSL 版本

在 Windows PowerShell 中执行：

```powershell
wsl --list --verbose
```

正常结果中的 Ubuntu 版本应为 `2`。如果不是，执行：

```powershell
wsl --set-version Ubuntu-24.04 2
wsl --set-default-version 2
```

如果提示虚拟化不可用，需要在 BIOS/UEFI 中启用 Intel VT-x、Intel Virtualization Technology、AMD-V 或 SVM。

## 四、更新 Ubuntu

打开 Ubuntu，执行：

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl jq ca-certificates openssl
```

检查系统信息：

```bash
cat /etc/os-release
```

## 五、安装 Windows 版 Docker Desktop

从 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/) 下载并安装 Docker Desktop。

安装时保持默认设置，并启用 WSL2 后端。安装完成后重启 Windows。

打开 Docker Desktop，依次进入：

```text
Settings → General
```

确认启用：

```text
Use the WSL 2 based engine
```

然后进入：

```text
Settings → Resources → WSL Integration
```

启用：

```text
Enable integration with my default WSL distro
```

并勾选 `Ubuntu-24.04`，点击 `Apply & Restart`。

## 六、验证 Docker

重新打开 Ubuntu，执行：

```bash
docker --version
docker compose version
docker run hello-world
```

如果 `hello-world` 运行成功，说明 Docker Desktop 已经与 WSL2 正常集成。

不要在 Ubuntu 内执行下面的命令：

```bash
sudo apt install docker-ce
```

Docker Engine 已由 Windows Docker Desktop 提供，重复安装可能导致服务冲突。

## 七、下载 Dify

建议把 Dify 放在 Ubuntu 的 Linux 文件系统中，而不是 `/mnt/c` 或 `/mnt/e`，以获得更好的 Docker、PostgreSQL、Redis 和向量数据库读写性能。

执行：

```bash
cd ~
git clone https://github.com/langgenius/dify.git
cd ~/dify/docker
ls -la
```

正常情况下应看到：

```text
.env.example
docker-compose.yaml
envs
```

也可以按 GitHub 最新正式版本克隆：

```bash
cd ~
git clone --branch "$(curl -s https://api.github.com/repos/langgenius/dify/releases/latest | jq -r .tag_name)" https://github.com/langgenius/dify.git
```

如果出现 `Remote branch null not found`，改用普通 `git clone` 命令。

## 八、配置 Dify

进入 Docker 目录并复制配置模板：

```bash
cd ~/dify/docker
cp .env.example .env
```

生成随机密钥：

```bash
openssl rand -base64 42
```

编辑配置文件：

```bash
nano .env
```

找到：

```env
SECRET_KEY=
```

填入刚才生成的随机字符串：

```env
SECRET_KEY=替换为随机字符串
```

保存并退出 Nano：

```text
Ctrl + O
Enter
Ctrl + X
```

首次本地部署通常可以先保留其他默认配置。不要把 `.env` 上传到公开仓库，因为其中可能包含数据库密码、加密密钥和 API Key。

## 九、启动 Dify

确认 Docker Desktop 正在运行，然后执行：

```bash
cd ~/dify/docker
docker compose up -d
```

首次启动会下载多个镜像，需要等待一段时间。

查看状态：

```bash
docker compose ps
docker ps
```

大多数服务应显示 `Up` 或 `healthy`。`init_permissions` 显示 `Exited` 通常是正常的，因为它是执行一次初始化任务后退出的容器。

## 十、访问 Dify

在 Windows 浏览器打开：

```text
http://localhost/install
```

完成管理员账号初始化后访问：

```text
http://localhost
```

如果修改了外部端口，例如 `8080`，则访问：

```text
http://localhost:8080/install
```

## 十一、端口冲突处理

在 Windows PowerShell 中检查 80 端口：

```powershell
Get-NetTCPConnection -LocalPort 80 -ErrorAction SilentlyContinue
```

如果 IIS、Nginx、Apache 或其他程序占用了 80 端口，可以停止占用程序，或根据当前 Dify 版本的 Compose 配置将外部端口改为 `8080`。

修改后重新创建容器：

```bash
cd ~/dify/docker
docker compose down
docker compose up -d
```

## 十二、日志和故障排查

查看服务状态：

```bash
docker compose ps -a
```

查看所有服务日志：

```bash
docker compose logs --tail=100
```

查看指定服务日志：

```bash
docker compose logs --tail=100 api
docker compose logs --tail=100 web
docker compose logs --tail=100 nginx
```

持续查看 API 日志：

```bash
docker compose logs -f api
```

按 `Ctrl + C` 退出日志查看。

## 十三、常用管理命令

```bash
cd ~/dify/docker
```

启动：

```bash
docker compose up -d
```

停止：

```bash
docker compose stop
```

重启：

```bash
docker compose restart
```

查看状态：

```bash
docker compose ps
```

停止并删除容器但保留数据卷：

```bash
docker compose down
```

不要随意执行以下命令，因为删除数据卷可能导致数据库和 Dify 业务数据丢失：

```bash
docker compose down -v
```

## 十四、查看 Windows 中的 Dify 文件

Ubuntu 中的项目目录：

```text
/home/你的Linux用户名/dify
```

Windows 文件资源管理器中可以访问：

```text
\\wsl$\Ubuntu-24.04\home\你的Linux用户名\dify
```

部署文件目录：

```text
\\wsl$\Ubuntu-24.04\home\你的Linux用户名\dify\docker
```

最重要的文件是：

```text
.env
docker-compose.yaml
```

也可以在 Ubuntu 中执行：

```bash
explorer.exe .
```

## 十五、升级 Dify

升级前备份配置：

```bash
mkdir -p ~/dify-backup
cp ~/dify/docker/.env ~/dify-backup/.env
cp ~/dify/docker/docker-compose.yaml ~/dify-backup/docker-compose.yaml
```

拉取代码并更新镜像：

```bash
cd ~/dify
git status
git pull
cd ~/dify/docker
docker compose pull
docker compose up -d
docker compose ps
```

升级后可以比较配置模板：

```bash
diff .env .env.example
```

根据需要补充新版本新增的环境变量。不要删除 Docker volumes、PostgreSQL 数据、Redis 数据、上传文件或向量数据库数据。

## 十六、数据备份建议

Dify 的完整备份至少应包括：

- `docker/.env`
- `docker/docker-compose.yaml`
- PostgreSQL 数据
- Redis 数据
- 上传文件
- 向量数据库数据

查看 Docker 数据卷：

```bash
docker volume ls
```

仅备份 `.env` 不能恢复完整的 Dify 业务数据，生产环境应配置定期数据库和文件备份。

## 十七、完整命令速查

### Windows 管理员 PowerShell

```powershell
wsl --install -d Ubuntu-24.04
wsl --list --verbose
wsl --set-version Ubuntu-24.04 2
wsl --set-default-version 2
```

### Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl jq ca-certificates openssl
docker --version
docker compose version
docker run hello-world
cd ~
git clone https://github.com/langgenius/dify.git
cd ~/dify/docker
cp .env.example .env
openssl rand -base64 42
nano .env
docker compose up -d
docker compose ps
```

浏览器访问：

```text
http://localhost/install
```

## 十八、安全提示

- 使用强密码初始化管理员账号。
- 不要公开 `.env` 文件。
- 不要把 API Key 提交到 GitHub。
- 如果 API Key 曾经出现在聊天记录、截图或公开仓库中，应立即撤销并重新生成。
- 生产环境建议配置域名、HTTPS、防火墙和定期备份。
