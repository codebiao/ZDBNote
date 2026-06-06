# Common Tools

+  `sudo apt update` 
+  `sudo vim /opt/software/requirements.txt`  

```bash
build-essential
git
zlib1g-dev
cmake 
autoconf 
libtool 
pkg-config
libssl-dev
net-tools
unzip
screen
perftest
valgrind
valgrind-dbg
gdb
patchelf
nfs-kernel-server
nfs-common
openssh-client
openssh-server
sshfs
tree
shellinabox
libwebsockets-dev 
libjson-c-dev 
libssl-dev
ttyd
ntpdate
ntp
samba 
network-manager
autofs
stress
iperf3
lm-sensors
dwarves
smartmontools
inotify-tools
```

- `sudo xargs -a /opt/software/requirements.txt apt install -y`

# Common Setting

## Date
```yaml
sudo vim /etc/profile
# Add at the end
# date
TZ='Asia/Shanghai'
export TZ

source /etc/profile

sudo timedatectl set-timezone Asia/Shanghai
```

+ Test

```bash
date
date -R
echo $TZ
# 如果时间还是不正确，有网络的可以同步一下北京时间
sudo apt install ntpdate
sudo systemctl stop ntp
sudo ntpdate ntp.aliyun.com
```

## 禁止内核更新
```bash
sudo apt autoremove -y --purge needrestart
```

## 设置coredump

```bash
sudo vim /etc/sysctl.d/core_pattern.conf
```

```bash
kernel.core_pattern=/cluster_files/data/crash/core.%e.%p.%h.%t
```

```bash
sudo sysctl -p /etc/sysctl.d/core_pattern.conf
```

## 设置sudo免密
```bash
sudo visudo

#在文件末尾加上
zas ALL=(ALL) NOPASSWD:ALL
```

## IP配置
+ 以imc1为例，创建并编写的文件：`sudo vim /etc/netplan/00-installer-config.yaml`

```yaml
#this is the network config written by 'user'
network:
    ethernets:
        eno1:
            addresses: [192.168.99.101/24]
        eno2:
            addresses: [172.16.107.27/24]
            nameservers:
              addresses: [172.16.100.57]
            routes:
            - to: default
              via: 172.16.107.254
        enp129s0f0np0:
            addresses: [10.0.0.101/24]
            mtu: 4200
        enp129s0f1np1:
            addresses: [11.0.0.101/24]
            mtu: 4200
    version: 2
```

vim命令模式 输入 `:set paste` 即可粘贴原格式，退出粘贴模式输入 :set nopaste

+ 赋予权限：`sudo chmod 600 /etc/netplan/00-installer-config.yaml`
+ 应用新的网络配置：`sudo netplan apply`
+ 检查：`ip a`
+ 禁用cloud-init: 

```bash
sudo vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 在文件中追加
network: {config: disabled}

# 删除文件50-cloud-init.yaml
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
# 重启cloud-init服务
sudo systemctl restart cloud-init
sudo netplan apply
```

+ 参考链接：
    - [Ubuntu 22.04网络配置指南：如何设置静态IP和自定义DNS服务器](https://blog.csdn.net/song19891121/article/details/136804891#:~:text=%E5%9C%A8%E5%AF%B9%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BF%9B%E8%A1%8C%E7%86%9F%E6%82%89%E6%93%8D)
    - [ubuntu22.04配置双网卡双静态ip不通网段访问服务器的相同服务](https://blog.csdn.net/simplyou/article/details/131471814#:~:text=ubuntu22.0)

## 设置大页内存
+ 请参考`config_huge_pages.md`

## 设置core dump目录
```bash
ulimit -c 	# 查看当前Core Dump设置，若为0则代表没打开

sudo vim /etc/security/limits.conf
# 在文件末尾加上
# remove core dump limit
zas soft core unlimited
zas hard core unlimited

sudo vim /etc/sysctl.conf
# 在文件末尾加上
kernel.core_pattern=|/usr/bin/core_handler.sh %h %t %e %p

sudo sysctl -p
```

+ `sudo vim /usr/bin/core_handler.sh`

```bash
#!/bin/bash

# --- 配置区 ---
STORAGE_PATH="/cluster_files/data/crash"
HOSTNAME=$1    	# 主机名，对应 %h
TIMESTAMP=$2	# UNIX 时间戳，对应 %t
EXE_NAME=$3		# 可执行文件名，对应 %e
PID=$4			# 进程 ID， 对应 %p

# 1. 创建存储目录（防止不存在）
mkdir -p -m 777 "$STORAGE_PATH"

# 2. 转换时间格式 (年月日_时分秒)
DATE_TIME=$(date -d "@$TIMESTAMP" "+%Y%m%d_%H%M%S")

# 3. 定义最终文件名
# 格式示例：/cluster_files/data/crash/core_myApp_PID1234_20260131_144500
CORE_FILE="${STORAGE_PATH}/core_${HOSTNAME}_${DATE_TIME}_${EXE_NAME}_PID${PID}"

# 4. 关键：从 stdin 读取 core 内容并写入文件
# 使用 cat 接收内核传来的二进制流
cat - > "$CORE_FILE"
```

+ 赋予权限：`sudo chmod +x /usr/bin/core_handler.sh`
+ 测试

```bash
kill -SIGSEGV $$

# 查看
cd /cluster_files/data/crash && ls
```

## 设置主机名称
```bash
sudo hostnamectl set-hostname imc1

# 查看主机名
hostnamectl
```

## SSH免密登录
```bash
sudo vim /etc/ssh/sshd_config
 
# 确保以下选项是启用的：
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

sudo systemctl restart sshd
```

## PFC流控
+ 用命令（重启后失效）

```bash
# 查看PFC
sudo mlnx_qos -i enp193s0f0np0 --trust dscp
# 设置PFC		# mlnx_qos 命令本身是非持久化的，重启后会失效
sudo mlnx_qos -i enp193s0f0np0 --pfc 1,0,0,0,0,0,0,0
```

+ 持久化

```bash
sudo vim /usr/local/bin/set_pfc.sh

#!/bin/bash
# PFC自动配置脚本
# 在IP地址为10.0.0.*的网卡上启用PFC

sleep 60  # 等待网络服务完全启动

# 获取所有网卡设备
for interface in $(ls /sys/class/net/); do
    # 获取当前网卡的 IPv4 地址
    IP=$(ip -4 addr show "$interface" 2>/dev/null | awk '/inet /{print $2}' | cut -d/ -f1)
    
    # 检查 IP 地址是否在 10.0.0.*范围内
    if [[ $IP =~ ^10\.0\.0\.[0-9]{1,3}$ ]]; then
        echo "Setting PFC on $interface ($IP)"
        if sudo mlnx_qos -i "$interface" --pfc 1,0,0,0,0,0,0,0; then
            echo "PFC set successfully on $interface ($IP)"
        else
            echo "Failed to set PFC on $interface ($IP)"
        fi
    fi
done
```

- 设置脚本执行权限
```bash
sudo chmod +x /usr/local/bin/set_pfc.sh
```

 - 创建systemd服务文件
```bash
sudo vim /etc/systemd/system/set-pfc.service

[Unit]
Description=Set PFC on interfaces with IP in 10.0.0.* range
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/set_pfc.sh
RemainAfterExit=yes
TimeoutStartSec=120

[Install]
WantedBy=multi-user.target
```

- 启用并启动服务

```bash
# 重新加载systemd配置
sudo systemctl daemon-reload

# 启用开机自启动
sudo systemctl enable set-pfc.service

# 立即启动服务
sudo systemctl start set-pfc.service

# 检查服务状态
sudo systemctl status set-pfc.service
```

## NTP时间同步
- Linux作为NTP客户端
```bash
sudo vim /etc/ntp.conf

# 添加以下内容
# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See http://www.pool.ntp.org/join.html for
# more information.
#pool 0.ubuntu.pool.ntp.org iburst
#pool 1.ubuntu.pool.ntp.org iburst
#pool 2.ubuntu.pool.ntp.org iburst
#pool 3.ubuntu.pool.ntp.org iburst
tos maxdist 30
server 192.168.99.100 iburst

# Use Ubuntu's ntp server as a fallback.
#pool ntp.ubuntu.com		# 注释掉

sudo systemctl restart ntp
```

```
# 强制同步时间
sudo systemctl stop ntp
sudo ntpdate -b -v 192.168.99.1
sudo systemctl start ntp
```

## NFS文件共享
```bash
sudo apt install nfs-common
sudo apt install nfs-kernel-server

sudo mkdir /cluster_files	# 创建共享目录
sudo chmod -R 777 /cluster_files	# 修改共享目录权限
sudo vim /etc/exports		# 修改配置文件

# 在文件中加入以下内容
/cluster_files/data     192.168.99.0/24(rw,sync,no_root_squash)
/cluster_files/zpx_algo/scripts  192.168.99.0/24(rw,sync,no_root_squash)

# 重启服务器
sudo service nfs-kernel-server restart

# 查看nfs服务器的共享目录
showmount -e localhost
```

+ 各字段含义
    - cluster_files：指定/nfsroot为nfs服务器的共享目录
    - *：允许所有的网段访问，也可以使用具体的IP，如192.168.184.0/24
    - rw：挂接此目录的客户端对该共享目录具有读写权限
    - sync：资料同步写入内存和硬盘
    - no_root_squash：root用户具有对根目录的完全管理访问权限
    - all_squash ：所有访问用户都映射为匿名用户或用户组。
    - async ：将数据先保存在内存缓冲区中，必要时才写入磁盘。
    - subtree_check（默认）：若输出目录是一个子目录，则nfs服务器将检查其父目录的权限
    - no_subtree_check：即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率。

## 脚本
### 检查文件夹数量
定时检查对应文件夹数量，删除创建时间组更早的。

+ `sudo vim /usr/local/bin/cleanup_folder.sh`

```shell
#!/bin/bash

# ========== 配置区域 ==========
# 需要处理的基础目录列表（每个目录下必须包含策略文件）
BASE_DIRS=(
    "/cluster_files"
    # 可以继续添加其他基础目录，例如：
    # "/cluster_files/another"
    # "/cluster_files/logs"
)

# 策略文件名（每个基础目录下的全局配置文件）
POLICY_FILE=".folder_cleanup.conf"

REQUIRED_PREFIX="/cluster_files"

# 2. 核心清理逻辑函数
do_cleanup() {
    local TARGET_DIR="$1"
    local KEEP_COUNT="$2"
    # 基础参数检查
    if [ -z "$TARGET_DIR" ] || [ -z "$KEEP_COUNT" ]; then
        echo "[ERROR] 参数缺失！用法: $0 <文件夹路径> <保留数量>"
        return 1
    fi
    
    # 安全校验：检查路径是否以指定前缀开头
    # 注意：这里使用双中括号进行通配符匹配
    if [[ "$TARGET_DIR" != "$REQUIRED_PREFIX"* ]]; then
        echo "[SECURITY ERROR] 越权访问！路径必须位于 $REQUIRED_PREFIX 之内。"
        echo "[INFO] 当前尝试访问的路径: $TARGET_DIR"
        return 1
    fi
    
    # 检查文件夹是否存在
    if [ ! -d "$TARGET_DIR" ]; then
        echo "[ERROR] 目标文件夹不存在: $TARGET_DIR"
        return 1
    fi
    
    # 5. 执行清理逻辑
    cd "$TARGET_DIR" || return 1
    
    # ls -1tr: 按时间由旧到新排序
    mapfile -t items < <(ls -1tr)
    total_count=${#items[@]}
    
    if [ "$total_count" -gt "$KEEP_COUNT" ]; then
        num_to_delete=$((total_count - KEEP_COUNT))
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] 路径: $TARGET_DIR | 最大保留项数: $KEEP_COUNT | 当前项数: $total_count | 清理: $num_to_delete"
    
        for ((i=0; i<num_to_delete; i++)); do
            item="${items[$i]}"
            if [ -n "$item" ]; then
                echo "正在删除: $item"
                rm -rf "$item"
            fi
        done
    else
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] 路径: $TARGET_DIR | 最大保留项数: $KEEP_COUNT | 当前项数: $total_count | 无需清理。"
    fi
    return 0
}

# ========== 主处理逻辑 ==========
for base_dir in "${BASE_DIRS[@]}"; do
    echo "========================================="
    echo "[INFO] 处理基础目录: $base_dir"

    # 检查基础目录是否存在
    if [ ! -d "$base_dir" ]; then
        echo "[ERROR] 基础目录不存在，跳过: $base_dir"
        continue
    fi

    policy_file_path="${base_dir}/${POLICY_FILE}"
    if [ ! -f "$policy_file_path" ]; then
        echo "[INFO] 策略文件不存在，正在创建: $policy_file_path"
        cat > "$policy_file_path" <<EOF
# 清理策略配置文件
# 格式：<子目录名> <保留数量>
# 示例：
# lowlevel 1000
# midlevel 500
EOF
        if [ $? -eq 0 ]; then
            chown zas:zas "$policy_file_path"
            echo "[INFO] 策略文件已创建: $policy_file_path"
            echo "[INFO] 请编辑该文件填入清理策略后重新运行脚本。跳过此基础目录。"
        else
            echo "[ERROR] 创建策略文件失败: $policy_file_path，跳过"
        fi
        continue
    fi

    # 逐行读取策略文件（格式：子目录名 保留数量，支持空行和#注释）
    while IFS= read -r line || [ -n "$line" ]; do
        # 去除前后空格
        line=$(echo "$line" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')
        # 跳过空行和注释行
        if [ -z "$line" ] || [[ "$line" =~ ^# ]]; then
            continue
        fi

        # 解析子目录名和保留数量
        read -r subdir keep_count <<< "$line"
        if [ -z "$subdir" ] || [ -z "$keep_count" ]; then
            echo "[WARN] 策略文件 ${policy_file_path} 中有无效行，跳过: $line"
            continue
        fi

        # 校验保留数量为正整数
        if ! [[ "$keep_count" =~ ^[1-9][0-9]*$ ]]; then
            echo "[WARN] 保留数量无效（应为正整数），行: $line，跳过"
            continue
        fi

        # 拼接完整路径
        target_dir="${base_dir}/${subdir}"

        # 执行清理
        echo "[INFO] 开始清理: $target_dir"
        do_cleanup "$target_dir" "$keep_count"
        echo "-----------------------------------------"
    done < "$policy_file_path"
done

echo "========================================="
echo "[INFO] 所有基础目录处理完成"
exit 0
```

+ 赋予权限:`sudo chmod +x /usr/local/bin/cleanup_folder.sh`

+ 设置systemd服务
  
    - `sudo vim /etc/systemd/system/cleanup_folder.service`
    
    ```bash
    [Unit]
    Description=定时清理文件夹脚本
    After=local-fs.target
    
    [Service]
    Type=oneshot
    User=root
    # 使用反斜杠进行换行，确保 \ 后面没有空格
    ExecStart=/bin/bash /usr/local/bin/cleanup_folder.sh
    
    [Install]
    WantedBy=multi-user.target
    ```
    
    - `sudo vim /etc/systemd/system/cleanup_folder.timer` 
    
    ```bash
    [Unit]
    Description=每10分钟触发一次清理服务
    
    [Timer]
    # 开机后等待 10 分钟开始第一次运行
    OnBootSec=10min
    # 每次运行完成后，间隔 10 分钟再次运行
    OnUnitActiveSec=10min
    # 提高定时精度（systemd 默认会有一定的延迟以节省能耗，这里强制精准）
    # AccuracySec=10min
    
    [Install]
    # 这里的 WantedBy 保证了定时器随系统启动而启动
    WantedBy=timers.target
    ```
    
    - 激活并启动
    
    ```bash
    # 1. 重新加载系统配置
    sudo systemctl daemon-reload
    
    # 2. 启用定时器（enable 确保开机自启，--now 确保立即启动）
    sudo systemctl enable --now cleanup_folder.timer
    
    # 查看定时器状态（看看下次什么时候运行）：
    systemctl list-timers cleanup_folder.timer
    
    # 查看脚本运行产生的日志，脚本里写了 echo，可以通过这个命令实时查看：
    journalctl -u cleanup_folder.service -n 40 -f
    
    # 手动立刻触发一次清理：
    sudo systemctl start cleanup_folder.service
    
    # 停止清理任务：
    sudo systemctl stop cleanup_folder.timer
    ```

### 自动创建软链接

需求：scan_node文件夹自动映射到raw_data目录下最新的文件夹

- `sudo vim /usr/local/bin/scan_node_link_latest.sh`

```bash
#!/bin/bash

prefix="/cluster_files/data"
A="scannode"
B="raw_data"

# 进入工作目录
cd "$prefix" || { echo "无法进入 $prefix"; exit 1; }

# 检查依赖
if ! command -v inotifywait &> /dev/null; then
    echo "请安装 inotify-tools"
    exit 1
fi

# 创建/更新符号链接
do_link() {
    local target="$1"
    
    # 检查目标是否存在
    if [ ! -d "$target" ]; then
        echo "$(date '+%Y-%m-%d %H:%M:%S'): 错误: 目标目录 $target 不存在"
        return 1
    fi
    
    # 删除旧的（软链接、目录或文件）
    if [ -L "$A" ]; then
        rm -f "$A"
    elif [ -d "$A" ]; then
        rm -rf "$A"
    elif [ -e "$A" ]; then
        rm -f "$A"
    fi
    
    # 创建新的软链接（相对路径）
    ln -s "$target" "$A"
    
    echo "$(date '+%Y-%m-%d %H:%M:%S'): symlink $A -> $target"
}

# 启动时：找 B 中最新修改的文件夹
init_link() {
    local latest=$(find "$B" -mindepth 1 -maxdepth 1 -type d -printf '%T@ %p\n' 2>/dev/null | \
                   sort -n | tail -1 | cut -d' ' -f2-)
    
    if [ -z "$latest" ]; then
        echo "$(date '+%Y-%m-%d %H:%M:%S'): $B 中没有子文件夹"
        return 1
    fi
    
    do_link "$latest"
}

# 运行时：新创建的文件夹直接链接
handle_new() {
    local new_dir="$1"
    
    # 确保是 B 的直接子文件夹
    local parent=$(dirname "$new_dir")
    [ "$parent" == "$B" ] || return
    
    do_link "$new_dir"
}

# 首次执行
init_link

# 实时监控
echo "开始监控 $B，新文件夹直接链接到 $A ..."
inotifywait -m -e create,moved_to --format '%w%f' "$B" | while read path; do
    # 只处理新创建的文件夹
    [ -d "$path" ] || continue
    
    # 忽略隐藏文件夹
    basename=$(basename "$path")
    [[ "$basename" == .* ]] && continue
    
    handle_new "$path"
done
```

- 赋予权限: `sudo chmod +x /usr/local/bin/scan_node_link_latest.sh`

- 设置systemd服务

  - `sudo vim /etc/systemd/system/scannode-link.service`

    ```bash
    [Unit]
    Description=Auto symlink scannode to latest raw_data folder
    After=network.target nfs-server.service
    
    [Service]
    Type=simple
    WorkingDirectory=/cluster_files/data
    ExecStart=/usr/local/bin/scan_node_link_latest.sh
    Restart=always
    RestartSec=5
    User=zas
    
    [Install]
    WantedBy=multi-user.target
    ```

- 激活并启动

  ```bash
  # 重新加载 systemd
  sudo systemctl daemon-reload
  
  # 开机自启
  sudo systemctl enable scannode-link.service
  
  # 立即启动
  sudo systemctl start scannode-link.service
  
  # 查看状态
  sudo systemctl status scannode-link.service
  
  # 查看脚本运行产生的日志，脚本里写了 echo，可以通过这个命令实时查看：
  journalctl -u scannode-link.service -n 40 -f
  ```


# CMake

+ [Website](https://blog.csdn.net/Man_1man/article/details/126467371)
+ Download the source from the [official-site](https://cmake.org/download/)：`cmake-3.31.5.tar.gz`

```bash
# Install required dependencies
sudo apt-get install libssl-dev

cd /opt/software
# sudo wget -c https://github.com/Kitware/CMake/releases/download/v3.31.5/cmake-3.31.5.tar.gz
sudo tar -zxvf cmake-3.31.5.tar.gz
cd cmake-3.31.5
sudo ./bootstrap
# Not recommended! "sudo ./configure --prefix=/usr/local/cmake3.30.2"
sudo make -j$(nproc)
sudo make install
```

+ Test：`cmake --version`


# Anaconda
+ Download the source from the [official-site](https://repo.anaconda.com/archive/index.html)：`Anaconda3-2024.06-1-Linux-x86_64.sh`

```bash
cd /opt/software
wget -c https://repo.anaconda.com/archive/Anaconda3-2024.06-1-Linux-x86_64.sh

sudo bash Anaconda3-2024.06-1-Linux-x86_64.sh
# PREFIX
/usr/local/myapp_install/anaconda3
```

+ Configure environment variables

```bash
sudo vim /etc/profile
# Add at the end
# Anaconda
ANACONDA_HOME=/usr/local/myapp_install/anaconda3
export PATH=$PATH:${ANACONDA_HOME}/bin

# Update configuration file
source /etc/profile

conda config --set auto_activate_base false
```

+ Test

```bash
conda --version
conda update conda

# Config Anaconda channel
conda config --add channels defaults

# create virtual environment
conda create --name pointnet grpcio-tools protobuf

# check
conda info --env
```

# gRPC
+ grpc for c++：[Website](https://github.com/grpc/grpc/blob/master/BUILDING.md#clone-the-repository-including-submodules)

```bash
sudo apt-get update
sudo apt-get install build-essential autoconf libtool pkg-config
sudo apt-get install -y libssl-dev

# Download the specified version of the library
cd /opt/software
git clone -b v1.66.1 https://github.com/grpc/grpc.git
cd grpc
git submodule update --init --recursive	# Initialize the submodule
mkdir build -p && cd build
sudo cmake -DCMAKE_INSTALL_PREFIX=/usr/local/myapp_install/grpc1.66.1 ..
sudo make -j$(nproc) install
```

+ Configure environment variables

```bash
sudo vim /etc/profile
# Add at the end
# grpc
GRPC_HOME=/usr/local/myapp_install/grpc1.66.1
export PATH=$PATH:${GRPC_HOME}/bin
export C_INCLUDE_PATH=$C_INCLUDE_PATH:${GRPC_HOME}/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:${GRPC_HOME}/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${GRPC_HOME}/lib
export LIBRARY_PATH=$LIBRARY_PATH:${GRPC_HOME}/lib

# Update configuration file
source /etc/profile
```

+ Test

```bash
cd /opt/software/grpc/examples/cpp/helloworld/
mkdir build && cd build
cmake  -DCMAKE_PREFIX_PATH="/usr/local/myapp_install/grpc1.66.1"  ..
make -j$(nproc)
./greeter_server Server：listening on 0.0.0.0:50051
./greeter_client

# "Greeter received: Hello world" appears, the installation is successful.
```

# OpenMPI
+ Download the source from the [official-site](https://www.open-mpi.org/)：`openmpi-5.0.5.tar.gz`

```bash
sudo apt-get install libnuma-dev numactl
sudo apt install libucx-dev ucx-utils -y

cd /opt/software
wget -c https://download.open-mpi.org/release/open-mpi/v5.0/openmpi-5.0.5.tar.gz
sudo tar -zxvf openmpi-5.0.5.tar.gz
cd openmpi-5.0.5
sudo ./configure	--prefix=/usr/local/myapp_install/openmpi5.0.5 \
                  --with-zlib \
                  --with-ucx
sudo make -j$(nproc) install
```

+ Configure environment variables

```bash
sudo vim /etc/profile
# Add at the end
# openmpi
MPI_HOME=/usr/local/myapp_install/openmpi5.0.5
export PATH=$PATH:${MPI_HOME}/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${MPI_HOME}/lib
export MANPATH=$MANPATH:${MPI_HOME}/share/man

# Update configuration file
source /etc/profile
```

+ Test

```bash
cd /opt/software/openmpi-5.0.5/examples/
mpicc hello_c.c -o hello -g
mpirun -n 10 hello

# Output
Hello, world, I am 0 of 4, (Open MPI v5.0.5, package: Open MPI zas@zas Distribution, ident: 5.0.5, repo rev: v5.0.5, Jul 22, 2024, 102)
Hello, world, I am 3 of 4, (Open MPI v5.0.5, package: Open MPI zas@zas Distribution, ident: 5.0.5, repo rev: v5.0.5, Jul 22, 2024, 102)
```

# NIC Driver
+ Download the source from the [official-site](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)：`MLNX_OFED-24.10-3.2.5.0/MLNX_OFED_LINUX-24.10-3.2.5.0-ubuntu22.04-x86_64.iso`
+ [Website](https://blog.csdn.net/qq_37960243/article/details/123066359#:~:text=%E9%AA%8C%E8%AF%81%E7%B3%BB%E7%BB%9F%E6%98%AF%E5%90%A6%E5%AE%89%E8%A3%85%E4%BA%86N)

```bash
cd /opt/software
sudo wget https://content.mellanox.com/ofed/MLNX_OFED-24.10-3.2.5.0/MLNX_OFED_LINUX-24.10-3.2.5.0-ubuntu22.04-x86_64.iso
# mount
sudo mount -o ro,loop MLNX_OFED_LINUX-24.10-3.2.5.0-ubuntu22.04-x86_64.iso /mnt
cd /mnt
sudo ./mlnxofedinstall --force --with-dkms --add-kernel-support
```

+ Test

```bash
sudo /etc/init.d/openibd restart 		# Load new driver
sudo hca_self_test.ofed							# Check the NIC status
ofed_info -s												# check version of ofed
ibstat
ibstatus
ibv_devinfo
ibv_devices		# View IB devices in host.
lspci | grep -i mell

sudo umount -f /mnt		# umount
```

# GPU Driver & CUDA
+ **<font style="color:#DF2A3F;">Note: The NIC driver must be installed first！</font>**
+ Download the source from the [official-site](https://developer.nvidia.com/cuda-toolkit-archive)：`cuda_12.6.0_560.28.03_linux.run`
+ Disable the Nouveau driver

```bash
sudo vim /etc/modprobe.d/blacklist-nouveau.conf

# Add at the end
blacklist nouveau
options nouveau modeset=0

sudo update-initramfs -u
sudo reboot
lsmod | grep nouveau		# If no content is displayed, the disablement was successful.
```
+ Driver:  `NVIDIA-Linux-x86_64-580.95.05.run`

+ Cuda

```bash
cd /opt/software
wget https://developer.download.nvidia.com/compute/cuda/12.6.2/local_installers/cuda_12.6.0_560.28.03_linux.run
sudo sh cuda_12.6.0_560.28.03_linux.run
```

+ Configure environment variables

```bash
sudo vim /etc/profile
# Add at the end
# Cuda
CUDA_HOME=/usr/local/cuda
export PATH=$PATH:${CUDA_HOME}/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CUDA_HOME}/lib64

# Update configuration file
source /etc/profile
```

+ Test：[Website](https://blog.csdn.net/qq_50677040/article/details/140210921)

```bash
nvcc -V
nvidia-smi
lspci |grep -i NVIDIA			# check GPU

nvidia-smi topo -m				# 查看是否在同一总线下, gpu和rdma的直连不是必须的，当然二者如果直连是最好的
sudo modprobe nvidia_peermem
lsmod | grep nvidia_peermem
sudo nvidia-smi -q -d RDMA | grep -i "GPUDirect RDMA"
```

# Opencv
+ Download the source from the [official-site](https://opencv.org/releases/)：`opencv-4.12.0.zip`

```bash
# Install required dependencies
# if has network
sudo apt-get install build-essential
sudo apt-get install libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev

cd /opt/software
git clone -b 4.12.0 https://github.com/opencv/opencv.git opencv-4.12.0
git clone -b 4.12.0 https://github.com/opencv/opencv_contrib.git opencv_contrib-4.12.0

# if no network, download opencv-4.12.0.zip and opencv_contrib-4.12.0.zip and upload it to linux
cd /opt/software
# upload zips into this dir
unzip opencv-4.12.0.zip
unzip opencv_contrib-4.12.0.zip

# then make build
cd opencv-4.12.0
sudo mkdir build && cd build
conda activate base				# activate python env
sudo cmake -D CMAKE_INSTALL_PREFIX=/usr/local/myapp_install/opencv4.12.0 \
           -D OPENCV_EXTRA_MODULES_PATH=/opt/software/opencv_contrib-4.12.0/modules \
           -D CMAKE_BUILD_TYPE=Release \
           -D OPENCV_GENERATE_PKGCONFIG=ON \
           -D OPENCV_ENABLE_NONFREE=ON \
           -D WITH_CUDA=ON \
           -D WITH_CUDNN=ON \
           -D CMAKE_CUDA_ARCHITECTURES="AUTO;8.6;8.0" \
           -D BUILD_opencv_python3=ON \
           -D PYTHON_EXECUTABLE=$(which python) \
           -D ENABLE_FAST_MATH=ON \
           -D CUDA_FAST_MATH=ON \
           -D WITH_CUBLAS=ON \
           -D WITH_TBB=ON \
           -D WITH_QT=OFF \
           -D WITH_OPENGL=ON \
           -D WITH_NVCUVID=OFF \
           -D WITH_NVCUVENC=OFF \
           -D BUILD_TESTS=OFF \
           -D BUILD_PERF_TESTS=OFF \
           -D BUILD_EXAMPLES=OFF \
           -D INSTALL_TESTS=OFF \
           ..
           
# Please wait patiently, the above step will download something.
sudo make -j$(nproc) install				# It takes a long time
```

+ Configure environment variables

```bash
sudo vim /etc/profile
# Add at the end
# opencv
OPENCV_HOME=/usr/local/myapp_install/opencv4.12.0
export C_INCLUDE_PATH=$C_INCLUDE_PATH:${OPENCV_HOME}/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:${OPENCV_HOME}/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${OPENCV_HOME}/lib
export LIBRARY_PATH=$LIBRARY_PATH:${OPENCV_HOME}/lib
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:${OPENCV_HOME}/lib/pkgconfig

# Update configuration file
source /etc/profile
```

+ Test：[Website](https://blog.csdn.net/KIK9973/article/details/118830187)

```bash
pkg-config --modversion opencv4		# check version
pkg-config --libs opencv4					# check libs
```

```cpp
#include <opencv2/opencv.hpp>
#include <opencv2/core/utils/logger.hpp>

int main()
{
    bool hasCuda = cv::cuda::getCudaEnabledDeviceCount() > 0;
    if (hasCuda)
    {
        std::cout << "OpenCV supports CUDA." << std::endl;

        // 获取CUDA设备信息
        int deviceCount = cv::cuda::getCudaEnabledDeviceCount();
        std::cout << "Number of CUDA-enabled devices: " << deviceCount << std::endl;

        for (int i = 0; i < deviceCount; ++i)
        {
            cv::cuda::printCudaDeviceInfo(i);
        }
    }
    else
    {
        std::cout << "OpenCV does not support CUDA or no CUDA devices found." << std::endl;
    }
    
    return 0;
}
```

# tcpdump
+ **<font style="color:#DF2A3F;">Note: The NIC driver must be installed first！</font>**
+ Recompiling tcpdump from source is required to capture RDMA NIC packets.

```bash
# tcpdump relies on libpcap, so libpcap must be compiled and installed first.
sudo apt install autoconf -y
sudo apt install flex bison -y
cd /opt/software
git clone https://github.com/the-tcpdump-group/libpcap.git
cd libpcap
sudo ./autogen.sh && sudo ./configure && sudo  make install

# Compile tcpdump with the same command.
cd /opt/software
git clone https://github.com/the-tcpdump-group/tcpdump.git
cd tcpdump
sudo ./autogen.sh && sudo ./configure &&  sudo make install
```

+ Test

```yaml
sudo tcpdump -i mlx5_0		# if no errors appear, the build is successful.
```

# node_exporter
```bash
cd /opt/software
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
cd node_exporter-1.7.0.linux-amd64
sudo cp node_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/node_exporter
sudo vim /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=zas
Group=zas
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable node_exporter.service
sudo systemctl start node_exporter.service
sudo systemctl status node_exporter.service


# env variable
​```bash
sudo vim /etc/profile

# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).

if [ "${PS1-}" ]; then
  if [ "${BASH-}" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "$(id -u)" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi

# date
TZ='Asia/Shanghai'
export TZ

# grpc
GRPC_HOME=/usr/local/myapp_install/grpc1.66.1
export PATH=$PATH:${GRPC_HOME}/bin
export C_INCLUDE_PATH=$C_INCLUDE_PATH:${GRPC_HOME}/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:${GRPC_HOME}/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${GRPC_HOME}/lib
export LIBRARY_PATH=$LIBRARY_PATH:${GRPC_HOME}/lib

# openmpi
MPI_HOME=/usr/local/myapp_install/openmpi5.0.5
export PATH=$PATH:${MPI_HOME}/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${MPI_HOME}/lib
export MANPATH=$MANPATH:${MPI_HOME}/share/man

# Cuda
CUDA_HOME=/usr/local/cuda
export PATH=$PATH:${CUDA_HOME}/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CUDA_HOME}/lib64

# opencv
OPENCV_HOME=/usr/local/myapp_install/opencv4.12.0
export C_INCLUDE_PATH=$C_INCLUDE_PATH:${OPENCV_HOME}/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:${OPENCV_HOME}/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${OPENCV_HOME}/lib
export LIBRARY_PATH=$LIBRARY_PATH:${OPENCV_HOME}/lib
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:${OPENCV_HOME}/lib/pkgconfig

# Anaconda
ANACONDA_HOME=/usr/local/myapp_install/anaconda3
export PATH=$PATH:${ANACONDA_HOME}/bin

# GSL
GSL_HOME=/usr/local/myapp_install/gsl2.8
export C_INCLUDE_PATH=$C_INCLUDE_PATH:${GSL_HOME}/include
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:${GSL_HOME}/include
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${GSL_HOME}/lib
export LIBRARY_PATH=$LIBRARY_PATH:${GSL_HOME}/lib

```

