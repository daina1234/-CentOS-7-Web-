**项目名称：搭建一个高可用的 Nginx 静态网站服务 + 自动化备份系统**

---


### 第一步：基础环境初始化 


#### 1. 检查网络
```bash
ping -c 4 www.baidu.com
```

#### 2. 更新系统并安装必备工具

```bash

yum makecache


yum install -y vim net-tools wget git
```

#### 3. 关闭防火墙 
```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0  
```

---

### 第二步：核心项目 - 搭建 Nginx 网站


#### 1. 安装编译依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

#### 2. 下载并编译 Nginx
```bash

mkdir -p /opt/software
cd /opt/software


wget http://nginx.org/download/nginx-1.24.0.tar.gz


tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0


./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module


make && make install


/usr/local/nginx/sbin/nginx -v

```

#### 3. 创建简单的网站首页
```bash

cd /usr/local/nginx/html


mv index.html index.html.bak


vim index.html
```
**在 vim 中输入以下内容**:
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

/usr/local/nginx/sbin/nginx


ps -ef | grep nginx


netstat -tulpn | grep 80
```
curl http://localhost

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


BACKUP_DIR="/data/nginx_backup"
LOG_FILE="/usr/local/nginx/logs/access.log"
DATE=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="access_log_$DATE.tar.gz"


if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
    echo "创建备份目录: $BACKUP_DIR"
fi


if [ -f "$LOG_FILE" ]; then
    tar -czf "$BACKUP_DIR/$ARCHIVE_NAME" -C /usr/local/nginx/logs access.log
    echo "[$(date)] 备份成功: $ARCHIVE_NAME" >> /var/log/backup_status.log
    
     
    > "$LOG_FILE"
    echo "[$(date)] 原日志已清空" >> /var/log/backup_status.log
else
    echo "[$(date)] 错误: 找不到日志文件 $LOG_FILE" >> /var/log/backup_status.log
fi


find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
echo "[$(date)] 7天前的旧备份已清理" >> /var/log/backup_status.log
```

#### 2. 赋予执行权限并测试
```bash
chmod +x /opt/scripts/backup_nginx_logs.sh


/opt/scripts/backup_nginx_logs.sh


ls -lh /data/nginx_backup/
cat /var/log/backup_status.log
```


#### 3. 设置定时任务 (Crontab)

```bash
crontab -e
```


**在文件末尾添加一行**:
```bash
0 2 * * * /opt/scripts/backup_nginx_logs.sh
```


---
