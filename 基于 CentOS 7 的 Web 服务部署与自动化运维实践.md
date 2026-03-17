**项目名称：搭建一个高可用的 Nginx 静态网站服务 + 自动化备份系统**

---


### 第一步：基础环境初始化 (登录虚拟机操作)

启动虚拟机，用 `root` 和你设置的密码登录。你会看到一个黑底白字的命令行界面。

#### 1. 检查网络
```bash
ping -c 4 www.baidu.com
```
如果能看到 `time=xx ms`，说明联网成功。如果不行，检查上面的网络设置是否为 NAT。

#### 2. 更新系统并安装必备工具
CentOS 7 默认源有时较慢，我们先安装常用工具。
```bash
# 更新 yum 缓存
yum makecache

# 安装 vim (编辑器), net-tools (查IP), wget (下载), git
yum install -y vim net-tools wget git
```

#### 3. 关闭防火墙 (实验环境简化操作，生产环境需配置规则)
```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0  # 临时关闭 SELinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config # 永久关闭
```

---

### 第二步：核心项目 - 搭建 Nginx 网站

我们将手动编译安装 Nginx（比直接用 yum 安装更能体现运维能力），并部署一个个人主页。

#### 1. 安装编译依赖
Nginx 需要编译，所以需要安装 gcc 和 openssl 等库。
```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

#### 2. 下载并编译 Nginx
```bash
# 创建目录
mkdir -p /opt/software
cd /opt/software

# 下载 Nginx 稳定版 (1.24.0)
wget http://nginx.org/download/nginx-1.24.0.tar.gz

# 解压
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# 配置编译选项 (安装到 /usr/local/nginx)
./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module

# 编译并安装 (需要一点时间)
make && make install

# 验证安装
/usr/local/nginx/sbin/nginx -v
# 输出 nginx version: nginx/1.24.0 表示成功
```

#### 3. 创建简单的网站首页
```bash
# 进入 html 目录
cd /usr/local/nginx/html

# 备份默认页
mv index.html index.html.bak

# 创建一个新的首页
vim index.html
```
**在 vim 中输入以下内容** (按 `i` 进入编辑模式，输完后按 `Esc`，输入 `:wq` 保存退出):
```html
<!DOCTYPE html>
<html>
<head>
    <title>My First Linux Ops Project</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; background-color: #f0f0f0; }
        .container { background: white; padding: 20px; border-radius: 10px; display: inline-block; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        h1 { color: #2c3e50; }
        p { color: #7f8c8d; }
        .status { color: green; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎉 恭喜！Nginx 服务运行正常</h1>
        <p>这是由 <strong>Root</strong> 用户在 CentOS 7 上手动编译部署的。</p>
        <p>当前时间：<span id="time"></span></p>
        <p class="status">● 系统状态：Running</p>
    </div>
    <script>
        document.getElementById('time').innerText = new Date().toLocaleString();
    </script>
</body>
</html>
```

#### 4. 启动 Nginx 并设置开机自启
```bash
# 启动
/usr/local/nginx/sbin/nginx

# 检查进程
ps -ef | grep nginx

# 查看端口监听
netstat -tulpn | grep 80
```
*此时，如果你在虚拟机内部执行 `curl http://localhost` 应该能看到 HTML 代码。*

---

### 第三步：自动化运维脚本 - 日志备份


#### 1. 创建备份脚本
```bash
mkdir -p /opt/scripts
vim /opt/scripts/backup_nginx_logs.sh
```

**输入以下代码**:
```bash
#!/bin/bash

# 配置变量
BACKUP_DIR="/data/nginx_backup"
LOG_FILE="/usr/local/nginx/logs/access.log"
DATE=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="access_log_$DATE.tar.gz"

# 1. 创建备份目录
if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
    echo "创建备份目录: $BACKUP_DIR"
fi

# 2. 复制并压缩日志
if [ -f "$LOG_FILE" ]; then
    tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" -C /usr/local/nginx/logs access.log
    echo "[$(date)] 备份成功: $ARCHIVE_NAME" >> /var/log/backup_status.log
    
    # 3. 清空原日志 
    > "$LOG_FILE"
    echo "[$(date)] 原日志已清空" >> /var/log/backup_status.log
else
    echo "[$(date)] 错误: 找不到日志文件 $LOG_FILE" >> /var/log/backup_status.log
fi

# 4. 清理 7 天前的旧备份
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
echo "[$(date)] 7天前的旧备份已清理" >> /var/log/backup_status.log
```

#### 2. 赋予执行权限并测试
```bash
chmod +x /opt/scripts/backup_nginx_logs.sh

# 手动运行一次测试
/opt/scripts/backup_nginx_logs.sh

# 检查结果
ls -lh /data/nginx_backup/
cat /var/log/backup_status.log
```
你应该能看到生成了一个 `.tar.gz` 文件，并且状态日志里记录了成功信息。

#### 3. 设置定时任务 (Crontab)
让系统每天凌晨 2 点自动执行。
```bash
crontab -e
```


**在文件末尾添加一行**:
```bash
0 2 * * * /opt/scripts/backup_nginx_logs.sh
```
保存退出。这意味着：每天 2:00 执行备份脚本。

---


