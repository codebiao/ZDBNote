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
+  Linux的NTP配置（Ubuntu 24.04起ntpd已废弃，改用chrony）

```bash
# 如果机器上原来跑着旧 ntpd，先停止（apt 安装 chrony 时会自动卸载 ntp 包，二者争用 UDP 123 端口）
sudo systemctl stop ntp ntpsec 2>/dev/null

sudo apt install chrony

sudo vim /etc/chrony/chrony.conf
```

```conf
server 192.168.99.1 iburst prefer
# 外网源不可达时以本地时钟为源对内网授时（对应原来的 server 127.127.1.0 + fudge）
# 注意必须带 distance 30：local 默认距离约 1s，网关 root dispersion 高达 ~10s，
# 不带 distance 时本地时钟会把真实源"挤掉"（tracking 一直显示本地参考），系统实际在自由振荡
local stratum 10 distance 30
# 允许内网客户端访问（对应原来的restrict）
allow 192.168.99.0/24
# 允许启动时大幅度校正
makestep 1.0 3
# 放宽源选择阈值（对应原来的 tos maxdist 30）
# 网关(Windows NTP)上报的 root dispersion ~10.3s，超过 chrony 默认的 3s，源会被判为不可用（^?）
maxdistance 30
```

```bash
# 注意：enable/disable 必须用真名 chrony（chronyd 只是别名，仅 start/stop/restart/status 可用别名）
sudo systemctl enable chrony
sudo systemctl restart chrony

# 验证
chronyc tracking        # Reference ID 应为 192.168.99.1(C0A86301)，Stratum 6，Leap status: Normal
chronyc sources -v      # 源状态应为 ^*（^? 表示未通过源选择测试，检查 maxdistance）
sudo chronyc clients    # 查看内网 NIMC 客户端取时情况
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
sudo mkdir -p /cluster_files/data/package /cluster_files/uploads/packages   # 部署解包目录(data/package) + 备份目录(uploads/packages)
sudo chown -R zas:zas /cluster_files/data/package /cluster_files/uploads   # 交给 zas，避免 deploy.sh 备份/解包时 Permission denied
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

## NFS 自动恢复监控

NFS 监控属于机器环境配置，运行时文件只能放在 `/usr/local/bin` 和 `/etc` 下，不依赖 `/cluster_files/zpx_algo`，避免程序更新覆盖监控配置。

监控流程如下：

1. 从 `/etc/nfs_mounts.conf` 读取并校验挂载项。
2. 使用 `flock` 防止 timer 重复启动多个实例。
3. 按“子挂载到父挂载”的顺序检查现有 NFS；来源不匹配或 `statfs` 超时时，执行 lazy unmount。
4. 按“父挂载到子挂载”的顺序恢复缺失项；父挂载不可用时不挂载子目录。
5. 新挂载完成后再次执行健康检查，失败则立即摘除，避免 `df -h` 长时间等待坏挂载。
6. systemd timer 在开机 30 秒后首次运行，之后每 30 秒执行一次；执行结果写入 systemd journal。

### 1. 安装监控脚本

```bash
sudo vim /usr/local/bin/auto_nfs_monitor.sh
```

写入以下内容：

```bash
#!/bin/bash

set -u

readonly CONF="${NFS_MOUNTS_CONF:-/etc/nfs_mounts.conf}"
readonly HEALTH_TIMEOUT_SECONDS=5
readonly MOUNT_TIMEOUT_SECONDS=15

declare -a REMOTES=()
declare -a MOUNTPOINTS=()
declare -a MOUNT_OPTIONS=()
declare -A SEEN_MOUNTPOINTS=()
FAILED=0

log()
{
    printf '%s auto_nfs_monitor: %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

load_configuration()
{
    if [[ ! -r "$CONF" ]]; then
        log "configuration is not readable: $CONF"
        return 1
    fi

    local line_number=0
    local remote local_mount option_flag mount_options trailing
    while read -r remote local_mount option_flag mount_options trailing; do
        ((line_number += 1))
        [[ -z "${remote:-}" || "$remote" == \#* ]] && continue

        if [[ -n "${trailing:-}" || "${option_flag:-}" != "-o" || -z "${mount_options:-}" ]]; then
            log "invalid configuration at $CONF:$line_number"
            FAILED=1
            continue
        fi
        if [[ "$remote" != *:/* || "$local_mount" != /imc_list/imc* ]]; then
            log "unsafe mount entry at $CONF:$line_number: $remote $local_mount"
            FAILED=1
            continue
        fi
        if [[ -n "${SEEN_MOUNTPOINTS[$local_mount]:-}" ]]; then
            log "duplicate mount point at $CONF:$line_number: $local_mount"
            FAILED=1
            continue
        fi

        SEEN_MOUNTPOINTS["$local_mount"]=1
        REMOTES+=("$remote")
        MOUNTPOINTS+=("$local_mount")
        MOUNT_OPTIONS+=("$mount_options")
    done < "$CONF"

    if ((${#MOUNTPOINTS[@]} == 0)); then
        log "no valid mount entries found in $CONF"
        return 1
    fi
}

mounted_fstype()
{
    findmnt -rn -M "$1" -o FSTYPE 2>/dev/null || true
}

mounted_source()
{
    findmnt -rn -M "$1" -o SOURCE 2>/dev/null || true
}

is_healthy()
{
    timeout --kill-after=1s "${HEALTH_TIMEOUT_SECONDS}s" stat -f -- "$1" >/dev/null 2>&1
}

detach_mount()
{
    local local_mount="$1"
    if umount -l -- "$local_mount"; then
        log "detached unhealthy mount: $local_mount"
    else
        log "failed to detach unhealthy mount: $local_mount"
        FAILED=1
    fi
}

configured_parent_is_ready()
{
    local index="$1"
    local local_mount="${MOUNTPOINTS[$index]}"
    local parent_index parent_mount parent_fstype

    for ((parent_index = 0; parent_index < index; ++parent_index)); do
        parent_mount="${MOUNTPOINTS[$parent_index]}"
        if [[ "$local_mount" == "$parent_mount"/* ]]; then
            parent_fstype="$(mounted_fstype "$parent_mount")"
            if [[ "$parent_fstype" != "nfs" && "$parent_fstype" != "nfs4" ]]; then
                return 1
            fi
        fi
    done
    return 0
}

exec 9>/run/lock/auto_nfs_monitor.lock
if ! flock -n 9; then
    log "another monitor instance is still running"
    exit 0
fi

if ! load_configuration; then
    exit 1
fi

# Detach children before parents so nested NFS mount points cannot be orphaned.
for ((index = ${#MOUNTPOINTS[@]} - 1; index >= 0; --index)); do
    local_mount="${MOUNTPOINTS[$index]}"
    expected_remote="${REMOTES[$index]}"
    fstype="$(mounted_fstype "$local_mount")"

    if [[ -z "$fstype" ]]; then
        continue
    fi
    if [[ "$fstype" != "nfs" && "$fstype" != "nfs4" ]]; then
        log "refusing to detach non-NFS mount at $local_mount (type=$fstype)"
        FAILED=1
        continue
    fi

    current_remote="$(mounted_source "$local_mount")"
    if [[ "$current_remote" != "$expected_remote" ]]; then
        log "source mismatch at $local_mount: expected=$expected_remote actual=$current_remote"
        detach_mount "$local_mount"
    elif ! is_healthy "$local_mount"; then
        log "health check failed: $expected_remote on $local_mount"
        detach_mount "$local_mount"
    fi
done

# Mount parents before children. A child is deferred if its configured parent failed.
for ((index = 0; index < ${#MOUNTPOINTS[@]}; ++index)); do
    remote="${REMOTES[$index]}"
    local_mount="${MOUNTPOINTS[$index]}"
    mount_options="${MOUNT_OPTIONS[$index]}"
    fstype="$(mounted_fstype "$local_mount")"

    if [[ "$fstype" == "nfs" || "$fstype" == "nfs4" ]]; then
        continue
    fi
    if [[ -n "$fstype" ]]; then
        log "refusing to cover non-NFS mount at $local_mount (type=$fstype)"
        FAILED=1
        continue
    fi
    if ! configured_parent_is_ready "$index"; then
        log "deferring $local_mount because its configured parent is unavailable"
        FAILED=1
        continue
    fi

    if ! mkdir -p -- "$local_mount"; then
        log "failed to create mount point: $local_mount"
        FAILED=1
        continue
    fi
    if ! timeout --kill-after=2s "${MOUNT_TIMEOUT_SECONDS}s" mount -t nfs -o "$mount_options" "$remote" "$local_mount"; then
        log "failed to mount $remote on $local_mount"
        FAILED=1
        continue
    fi
    if ! is_healthy "$local_mount"; then
        log "new mount failed health check: $remote on $local_mount"
        detach_mount "$local_mount"
        FAILED=1
        continue
    fi
    log "mounted $remote on $local_mount"
done

exit "$FAILED"
```

```bash
sudo chmod 0755 /usr/local/bin/auto_nfs_monitor.sh
sudo bash -n /usr/local/bin/auto_nfs_monitor.sh
```

### 2. 配置 NFS 挂载列表

```bash
sudo vim /etc/nfs_mounts.conf
```

当前集群只配置 `imc0` 到 `imc6`。父挂载必须写在对应子挂载之前；只有节点和两个 export 都存在后才能增加新节点。

```text
# Keep parent mounts before their nested child mounts.
# soft is retained here so one unavailable aggregation node does not block df indefinitely.
192.168.99.100:/cluster_files/data /imc_list/imc0 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.100:/cluster_files/zpx_algo/scripts /imc_list/imc0/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.101:/cluster_files/data /imc_list/imc1 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.101:/cluster_files/zpx_algo/scripts /imc_list/imc1/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.102:/cluster_files/data /imc_list/imc2 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.102:/cluster_files/zpx_algo/scripts /imc_list/imc2/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.103:/cluster_files/data /imc_list/imc3 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.103:/cluster_files/zpx_algo/scripts /imc_list/imc3/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.104:/cluster_files/data /imc_list/imc4 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.104:/cluster_files/zpx_algo/scripts /imc_list/imc4/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.105:/cluster_files/data /imc_list/imc5 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.105:/cluster_files/zpx_algo/scripts /imc_list/imc5/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.106:/cluster_files/data /imc_list/imc6 -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
192.168.99.106:/cluster_files/zpx_algo/scripts /imc_list/imc6/scripts -o nfsvers=3,nolock,soft,timeo=7,retrans=2,retry=2
```

### 3. 配置 systemd service

```bash
sudo vim /etc/systemd/system/auto-nfs-monitor.service
```

```ini
[Unit]
Description=Auto NFS mount monitor
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/auto_nfs_monitor.sh
TimeoutStartSec=300
```

### 4. 配置每 30 秒执行的 timer

```bash
sudo vim /etc/systemd/system/auto-nfs-monitor.timer
```

```ini
[Unit]
Description=Run auto_nfs_monitor.sh every 30 seconds

[Timer]
OnBootSec=30sec
OnUnitActiveSec=30sec
AccuracySec=5sec
Unit=auto-nfs-monitor.service

[Install]
WantedBy=timers.target
```

### 5. 加载并启动定时任务

```bash
sudo systemd-analyze verify /etc/systemd/system/auto-nfs-monitor.service /etc/systemd/system/auto-nfs-monitor.timer
sudo systemctl daemon-reload
sudo systemctl enable --now auto-nfs-monitor.timer

# 立即手动执行一次，不必等待下一次 timer
sudo systemctl start auto-nfs-monitor.service
```

### 6. 检查运行状态

```bash
systemctl list-timers auto-nfs-monitor.timer --no-pager
systemctl status auto-nfs-monitor.timer --no-pager
systemctl status auto-nfs-monitor.service --no-pager
journalctl -u auto-nfs-monitor.service --since today --no-pager

# 验证所有 NFS 挂载；本地磁盘查看可使用 df -h -x nfs -x nfs4
findmnt -t nfs,nfs4
df -h
```

# samba共享目录

+ 修改配置文件
```bash
sudo mkdir -p /cluster_files/data/mshare
sudo chmod 777 /cluster_files/data/mshare
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak	# 备份samba配置文件
sudo vim /etc/samba/smb.conf	# 添加以下内容
```

+ 添加以下内容

```bash
[mshare]
   comment = share results using Samba
   path = /cluster_files/data/mshare
   public = yes
   writable = yes
   available = yes
   browseable = yes
   valid users = zas

[mdata]
   comment = share raw data using Samba
   path = /imc_list
   public = no
   writable = yes
   available = yes
   browseable = yes
   valid users = zas
```
+ 设置账号密码（必做）
```bash
sudo smbpasswd -a zas
```
+ 字段解释：
	+ [mshare]：这是共享的名称，你可以在网络上访问该共享时使用。
	+ comment：这是关于共享的描述或注释，显示给用户看。
	+ path：这是共享的实际路径。
	+ public：这表示该共享是否为公共共享，即是否允许匿名用户访问。
	+ writable：表示是否允许用户在共享中创建、编辑和删除文件。
	+ available：表示该共享是否可用。
	+ browseable：表示该共享是否在网络上可以浏览。
	+ valid users：当前 Ubuntu 系统的用户名。

+ 查询指令
```bash
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
sudo ./configure --prefix=/usr/local/myapp_install/nginx \
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
sudo /usr/local/myapp_install/nginx/sbin/nginx

# check
ps -aux | grep nginx
# close
sudo /usr/local/myapp_install/nginx/sbin/nginx -s stop
```

## 配置
```bash
sudo chmod 777 -R /usr/local/myapp_install/nginx/html
cd /usr/local/myapp_install/nginx/html
# 拷贝网页文件dist到html文件夹下

sudo /usr/local/myapp_install/nginx/sbin/nginx				# 启动
sudo /usr/local/myapp_install/nginx/sbin/nginx -s stop		# 关闭
sudo /usr/local/myapp_install/nginx/sbin/nginx -s reload	# 重新加载
ps -aux | grep nginx		# 查看
```

+ `sudo mkdir -p /cluster_files/uploads/config/ssl`

```bash
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -subj "/C=CN/ST=Default/L=Default/O=Default/OU=IT/CN=localhost" \
  -keyout /cluster_files/uploads/config/ssl/server.key \
  -out /cluster_files/uploads/config/ssl/server.crt
```

+ `sudo vim /usr/local/myapp_install/nginx/conf/nginx.conf`

```bash
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
    keepalive_timeout  300;

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
        
        location /numa {
            proxy_pass https://192.168.99.100:8888/numa;
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

+ `sudo vim /etc/systemd/system/nginx.service`

```bash
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

ps -aux | grep nginx		         # 查看
sudo systemctl status nginx.service  # 检查 systemd 服务的状态
sudo journalctl -u nginx.service     # 查看日志输出
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
Type=simple
User=zas
Group=zas
ExecStart=/usr/bin/ttyd -p 8765 -W bash
WorkingDirectory=/home/zas
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
