在DIMC配置的基础上，进行以下设置。

# Common Setting
## 设置主机名称
```bash
sudo hostnamectl set-hostname imc0

# 查看主机名
hostnamectl
```

# SSH免密登录
+ 在本地主机上，生成密钥对，并将公钥复制到远程主机

```bash
# 生成密钥对
ssh-keygen -t rsa

# 将公钥复制到远程主机
ssh-copy-id zas@11.0.0.101

# 测试免密登录
ssh zas@11.0.0.101
```

# NTP时间同步
+  Linux的NTP配置

```bash
sudo vim /etc/ntp.conf

tos maxdist 30
server 192.168.99.1 iburst
restrict 192.168.99.0 mask 255.255.255.0 nomodify notrap
```

```bash
sudo systemctl restart ntp
```


# NFS文件共享
```bash
sudo apt install nfs-common

# 查看服务器共享了哪些目录
showmount -e 192.168.99.101

sudo mkdir /imc_list/imc1 		# 客户端创建挂载目录
sudo mount -t nfs -o nolock,soft,timeo=7,retry=2 192.168.99.101:/cluster_files /imc_list/imc1	# 挂载目录

# 卸载服务共享目录
sudo mount	| grep <目录>				# 查看挂载的目录
sudo umount /imc1                                      # 卸载目录

#host_imc
sudo apt install nfs-kernel-server

sudo mkdir /cluster_files
sudo chmod -R 777 /cluster_files
sudo mkdir -p /cluster_files/data
sudo mkdir -p /cluster_files/zpx_algo/scripts
sudo vim /etc/exports
# 文件中加入以下内容
/cluster_files/data     192.168.99.0/24(rw,sync,no_root_squash)
/cluster_files/zpx_algo/scripts  192.168.99.0/24(rw,sync,no_root_squash)
```

```bash
sudo service nfs-kernel-server restart
sudo mkdir /imc_list/imc0
sudo mkdir /imc_list/imc0/scripts
sudo mount -t nfs -o nolock,soft,timeo=7,retry=2 192.168.99.100:/cluster_files/data /imc_list/imc0
sudo mount -t nfs -o nolock,soft,timeo=7,retry=2 192.168.99.100:/cluster_files/zpx_algo/scripts /imc_list/imc0/scripts						
```

+ Configure NFS restore script


```bash
sudo vim /usr/local/bin/auto_nfs_monitor.sh
```

```bash
#!/bin/bash
CONF="/etc/nfs_mounts.conf"

while read -r line; do
    [[ "$line" =~ ^#.*$ || -z "$line" ]] && continue
    remote=$(echo $line | awk '{print $1}')
    localmnt=$(echo $line | awk '{print $2}')
    opts=$(echo $line | awk '{$1="";$2=""; print $0}' | sed 's/^ *//')
    mount | grep -q "on $localmnt type nfs"
    is_mounted=$?
    if [ $is_mounted -eq 0 ]; then
        timeout 3 ls "$localmnt" &>/dev/null
        if [ $? -ne 0 ]; then
            sudo umount -l "$localmnt"
        fi
    else
        timeout 3 showmount -e $(echo $remote | cut -d: -f1) &>/dev/null
        if [ $? -eq 0 ]; then
            sudo mkdir -p "$localmnt"
            sudo mount -t nfs $opts $remote $localmnt
        fi
    fi
done < "$CONF"
```

```bash
sudo chmod +x auto_nfs_monitor.sh
sudo vim /etc/nfs_mounts.conf
```

```bash
# 文件中加入以下内容
192.168.99.100:/cluster_files/data /imc_list/imc0 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.100:/cluster_files/zpx_algo/scripts /imc_list/imc0/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.101:/cluster_files/data /imc_list/imc1 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.101:/cluster_files/zpx_algo/scripts /imc_list/imc1/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.102:/cluster_files/data /imc_list/imc2 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.102:/cluster_files/zpx_algo/scripts /imc_list/imc2/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.103:/cluster_files/data /imc_list/imc3 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.103:/cluster_files/zpx_algo/scripts /imc_list/imc3/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.104:/cluster_files/data /imc_list/imc4 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.104:/cluster_files/zpx_algo/scripts /imc_list/imc4/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.105:/cluster_files/data /imc_list/imc5 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.105:/cluster_files/zpx_algo/scripts /imc_list/imc5/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.106:/cluster_files/data /imc_list/imc6 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.106:/cluster_files/zpx_algo/scripts /imc_list/imc6/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.107:/cluster_files/data /imc_list/imc7 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.107:/cluster_files/zpx_algo/scripts /imc_list/imc7/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.108:/cluster_files/data /imc_list/imc8 -o nfsvers=3,nolock,soft,timeo=7,retry=2
192.168.99.108:/cluster_files/zpx_algo/scripts /imc_list/imc8/scripts -o nfsvers=3,nolock,soft,timeo=7,retry=2

```

```bash
sudo vim /etc/systemd/system/auto-nfs-monitor.service

[Unit]
Description=Auto NFS mount monitor
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/auto_nfs_monitor.sh

[Install]
WantedBy=multi-user.target


sudo vim /etc/systemd/system/auto-nfs-monitor.timer

[Unit]
Description=Run auto_nfs_monitor.sh every 30 sec

[Timer]
OnBootSec=5sec
OnUnitActiveSec=30secdf -
Unit=auto-nfs-monitor.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable auto-nfs-monitor.timer
sudo systemctl start auto-nfs-monitor.timer
sudo systemctl status auto-nfs-monitor.timer
```

# samba共享目录
```bash
sudo mkdir -p /cluster_files/data/mshare
sudo chmod 777 /cluster_files/data/mshare
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak	# 备份samba配置文件
sudo vim /etc/samba/smb.conf	# 添加以下内容

[mshare]
   comment = share results using Samba
   path = /mshare
   public = yes
   writable = yes
   available = yes
   browseable = yes
   valid users = zas
```

+ [mshare]：这是共享的名称，你可以在网络上访问该共享时使用。
+ comment：这是关于共享的描述或注释，显示给用户看。
+ path：这是共享的实际路径。
+ public：这表示该共享是否为公共共享，即是否允许匿名用户访问。
+ writable：表示是否允许用户在共享中创建、编辑和删除文件。
+ available：表示该共享是否可用。
+ browseable：表示该共享是否在网络上可以浏览。
+ valid users：当前 Ubuntu 系统的用户名。

```bash
sudo smbpasswd -a zas
sudo systemctl restart smbd.service
sudo systemctl enable smbd.service
sudo systemctl status smbd.service
# 在弹窗里输入虚拟机的密码(注意：不是Samba用户的密码)
```

# Nginx
## 安装
+ Download the source from the [official-site](https://nginx.org/en/download.html)：`nginx-1.26.3.tar.gz`

```bash
sudo apt-get install libpcre3 libpcre3-dev -y
sudo apt-get install zlib1g zlib1g-dev -y

cd /opt/software
wget -c https://nginx.org/download/nginx-1.26.3.tar.gz
sudo tar -zxvf nginx-1.26.3.tar.gz
cd nginx-1.26.3
./configure --prefix=/usr/local/myapp_install/nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-stream \
  --with-stream_ssl_module \
  --with-cc-opt="-I/usr/include" \
  --with-ld-opt="-L/usr/lib/x86_64-linux-gnu -lssl -lcrypto"
sudo make install -j$(nproc)
```

+ Test

```bash
cd /usr/local/myapp_install/nginx/sbin
sudo ./nginx

# check
ps -aux | grep nginx
# close
sudo ./nginx -s stop
```

## 配置
```bash
sudo chmod 777 -R /usr/local/myapp_install/nginx/html
cd /usr/local/myapp_install/nginx/html
# 拷贝网页文件dist到html文件夹下
sudo vim /usr/local/myapp_install/nginx/conf/nginx.conf
# 将location下的root， 从html 改为html/dist

sudo /usr/local/myapp_install/nginx/sbin/nginx						# 启动
sudo /usr/local/myapp_install/nginx/sbin/nginx -s stop		# 关闭
sudo /usr/local/myapp_install/nginx/sbin/nginx -s reload	# 重新加载
ps -aux | grep nginx		# 查看
```

+ nginx.conf

```bash
mkdir -p /cluster_files/uploads/config/ssl/

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -subj "/C=CN/ST=Default/L=Default/O=Default/OU=IT/CN=localhost" \
  -keyout /cluster_files/uploads/config/ssl/server.key \
  -out /cluster_files/uploads/config/ssl/server.crt

```

```bash
sudo vim /usr/local/myapp_install/nginx/conf/nginx.conf

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    client_max_body_size 1G;
    gzip  on;

    server {
        listen       80;
        server_name  localhost;

        return 301 https://$host$request_uri;
    }

    # HTTPS server
    server {
        listen       443 ssl;
        server_name  localhost;

        # 请替换为您的证书文件路径
        ssl_certificate      /cluster_files/uploads/config/ssl/server.crt;
        ssl_certificate_key  /cluster_files/uploads/config/ssl/server.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html/dist;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        location /webapi {
            proxy_pass https://192.168.99.100:8888/webapi;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 添加 CORS 头
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
        }

        location /cluapi {
            proxy_pass https://192.168.99.100:8888/cluapi;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
        }

        location /version_ctl {
            proxy_pass https://192.168.99.100:8888/version_ctl;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
        }

        location /probe {
            proxy_pass https://192.168.99.100:8888/probe;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
            add_header Access-Control-Allow-Headers 'Content-Type, Authorization';
        }

        location /imc_list {
            alias /imc_list;                # 指定文件夹路径
            autoindex on;                   # 启用目录索引
            sendfile on;                    # 开启高效文件传输模式
            autoindex_exact_size off;       # 显示文件大小为人类可读格式
            autoindex_localtime on;         # 显示文件的本地时间
            autoindex_format html;          # 显示索引页面文件风格，默认html
            
            charset utf-8;                  # 【新增】指定字符集，防止中文乱码
            default_type text/plain;        # 【新增】强制默认为文本类型

            #limit_rate 1024k;              # 限速，默认不限速
            #charset utf-8,gbk;             # 避免中文乱码
            
            # 解决跨域问题
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
        }

        location /ttyd/ {
            proxy_pass http://192.168.99.100:8765/;
            proxy_http_version 1.1;  # 必须支持 HTTP/1.1
            proxy_set_header Upgrade $http_upgrade;  # 必须转发 Upgrade 请求头
            proxy_set_header Connection 'upgrade';  # 必须转发 Connection 请求头
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```

+ 为nginx配置`systemd` 服务

```bash
sudo vim /etc/systemd/system/nginx.service

# Add at the end
[Unit]
Description=NGINX Web Server
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/myapp_install/nginx/sbin/nginx
ExecReload=/usr/local/myapp_install/nginx/sbin/nginx -s reload
ExecStop=/usr/local/myapp_install/nginx/sbin/nginx -s stop
PIDFile=/usr/local/myapp_install/nginx/logs/nginx.pid
LimitNOFILE=8192
# User=zas
# Group=zas

[Install]
WantedBy=multi-user.target
```

```bash
# operation
sudo systemctl daemon-reload
sudo systemctl start nginx.service
sudo systemctl restart nginx.service
sudo systemctl reload nginx.service
sudo systemctl stop nginx.service

ps -aux | grep nginx		# 查看
sudo systemctl status nginx.service	 # 检查 systemd 服务的状态
sudo journalctl -u nginx.service		 # 查看日志输出
sudo systemctl enable nginx.service  # 设置为开机自启
```

# ttyd Web Terminal
+ 安装在 head 节点（imc0），提供浏览器端终端访问

```bash
sudo apt install ttyd
```

+ 配置 systemd 服务，包含每天自动重启以防止 PTY 资源泄漏

```bash
sudo vim /etc/systemd/system/ttyd.service
```

```
[Unit]
Description=TTYD Web Terminal
After=network.target

[Service]
ExecStart=/usr/bin/ttyd -p 8765 bash
WorkingDirectory=/tmp
Restart=always
RestartSec=5
RuntimeMaxSec=86400

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ttyd.service
sudo systemctl start ttyd.service
sudo systemctl status ttyd.service

# 验证每天自动重启已生效（应输出 RuntimeMaxUSec=1d）
systemctl show ttyd | grep -i runtimemax
```

# disk info

```
mkdir /cluster_files/data/disk_info
```

```bash
sudo vim /opt/deploy_disk_info.sh
```

```bash
#!/bin/bash
# 1. 配置参数
TARGET_HOSTS=(
    "11.0.0.100"
    "11.0.0.101"
    "11.0.0.102"
    "11.0.0.103"
    "11.0.0.104"
    "11.0.0.105"
    "11.0.0.106"
    "11.0.0.107"
    "11.0.0.108"
)
REMOTE_USER="zas"
REMOTE_SCRIPT_PATH="/usr/local/bin/collect_nvme_info.sh"
TARGET_DATA_DIR="/cluster_files/data/disk_info"

# 2. 定义采集脚本内容 (使用 'EOF' 防止本地变量被解析)
read -r -d '' SCRIPT_CONTENT << 'EOF'
#!/bin/bash
TARGET_DIR="/cluster_files/data/disk_info"
TARGET_FILE="$TARGET_DIR/disk.json"
TEMP_FILE="$TARGET_FILE.$$.tmp"

mkdir -p "$TARGET_DIR"
DEV=$(ls /dev/nvme[0-9]n1 2>/dev/null | head -n 1)

if [ -z "$DEV" ]; then
    echo "{\"error\": \"No NVMe device found\"}" > "$TARGET_FILE"
    exit 1
fi

# 采集数据
sudo /usr/sbin/smartctl -j -A "$DEV" > "$TEMP_FILE"
mv "$TEMP_FILE" "$TARGET_FILE"
chmod 644 "$TARGET_FILE"
EOF

# 3. 开始执行
for HOST in "${TARGET_HOSTS[@]}"; do
    echo ">>>> 正在配置主机: $HOST"

    # A. 分发脚本
    ssh "$REMOTE_USER@$HOST" "echo '$SCRIPT_CONTENT' | sudo tee $REMOTE_SCRIPT_PATH > /dev/null && sudo chmod +x $REMOTE_SCRIPT_PATH"

    # B. 准备目录
    ssh "$REMOTE_USER@$HOST" "sudo mkdir -p $TARGET_DATA_DIR && sudo chown $REMOTE_USER:$REMOTE_USER $TARGET_DATA_DIR"

    # C. 配置 sudo 免密 (针对 smartctl)
    ssh "$REMOTE_USER@$HOST" "echo '$REMOTE_USER ALL=(ALL) NOPASSWD: /usr/sbin/smartctl' | sudo tee /etc/sudoers.d/zas_smartctl > /dev/null && sudo chmod 440 /etc/sudoers.d/zas_smartctl"

    # D. 配置 Crontab (整点执行一次)
    CRON_CMD="0 * * * * $REMOTE_SCRIPT_PATH > /dev/null 2>&1"
    ssh "$REMOTE_USER@$HOST" "(crontab -l 2>/dev/null | grep -v '$REMOTE_SCRIPT_PATH' ; echo '$CRON_CMD') | crontab -"

    # E. 立即执行一次
    ssh "$REMOTE_USER@$HOST" "bash $REMOTE_SCRIPT_PATH"
    
    echo "<<<< 主机 $HOST 配置完成。"
done
exit 0
```

```bash
sudo chmod +x /opt/deploy_disk_info.sh
sudo /opt/deploy_disk_info.sh
```

# zpweb
## MPI/UCX 运行环境说明
+ `launch_zpweb.sh` 负责加载 profile 并启动 `zpweb` 进程，但 `memlock` 上限不由脚本保证。
+ 如果只修改脚本或只执行 `restartweb`，没有同时配置 PAM / systemd 的 limits 链路，UCX 仍可能报错：
+ 因此，除了 `launch_zpweb.sh` 之外，还需要同时检查并配置以下文件：
  + `/etc/security/limits.conf`
  + `/etc/pam.d/common-session`
  + `/etc/pam.d/common-session-noninteractive`
  + `/etc/systemd/system/zpweb.service`

## 系统限制配置
```bash
sudo vim /etc/security/limits.conf
```

```conf
# zpweb / UCX / MPI memlock setting
* soft memlock unlimited
* hard memlock unlimited

# core dump setting
* soft core unlimited
* hard core unlimited
```

## PAM limits 配置
```bash
sudo vim /etc/pam.d/common-session
sudo vim /etc/pam.d/common-session-noninteractive
```

+ 确保两个文件中都包含以下内容：

```pam
session required pam_limits.so
```

## systemd 配置
```bash
sudo vim /etc/systemd/system/zpweb.service
# 添加以下内容
[Unit]
Description=ZPWeb Application Service
After=network.target local-fs.target

[Service]
Type=oneshot
RemainAfterExit=yes
User=zas
Environment=HOME=/home/zas
Environment=TERM=xterm
LimitMEMLOCK=infinity
Environment=ZPWEB_CERT_FILE=/cluster_files/uploads/config/ssl/server.crt
Environment=ZPWEB_KEY_FILE=/cluster_files/uploads/config/ssl/server.key
ExecStart=/cluster_files/zpx_algo/scripts/launch_zpweb.sh
ExecStop=/usr/bin/screen -S zpweb -X quit

[Install]
WantedBy=multi-user.target
```

+ `LimitMEMLOCK=infinity` 用于保证 `zpweb` 通过 systemd 启动时具备足够的 locked memory 上限，避免 UCX / RDMA 初始化时因 `memlock` 过小而失败。

## 生效方式 / 重新加载
```bash
sudo systemctl daemon-reload
sudo systemctl enable zpweb.service
sudo systemctl start zpweb.service
sudo systemctl status zpweb.service
```

+ `zpweb` 默认会以 HTTPS 监听 `192.168.99.100:8888`，并复用 `/cluster_files/uploads/config/ssl/server.crt` 和 `/cluster_files/uploads/config/ssl/server.key`
+ 如需自定义证书，可在 `zpweb.service` 或启动环境中覆盖 `ZPWEB_CERT_FILE` / `ZPWEB_KEY_FILE`

+ 修改 `zpweb.service` 后，需要执行 `sudo systemctl daemon-reload`，然后重新启动 `zpweb` 服务。
+ 如果已有旧的 `screen` 会话，建议先销毁后再重启，避免旧会话沿用旧 limits：

```bash
screen -S zpweb -X quit
sudo systemctl restart zpweb.service
```

+ 修改 `/etc/security/limits.conf` 或 PAM 配置后，已有登录会话不会自动刷新。
+ 建议重新登录后再重启 `zpweb`；如果仍然不确定当前会话是否已加载新 limits，建议直接重启整机。

## 验证命令
```bash
ulimit -l
cat /proc/$(pidof zpweb)/limits | grep locked
systemctl status zpweb.service
```
+ 如果 `ulimit -l` 或 `/proc/.../limits` 中仍显示 `64 kbytes`，说明当前 `zpweb` 进程拿到的仍然是旧 limits，问题通常在 PAM / systemd 配置未生效，而不是 `launch_zpweb.sh` 本身。

# 大页内存
+ 请参考`config_huge_pages.md`