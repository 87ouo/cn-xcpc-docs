# `DOMjudge 8.2.2` 实机安装教程

## 一、`DOMserver` 安装

### 0、环境（纯净状态）

```
- Ubuntu 22.04.3 LTS
- NGINX 1.18.0
- PHP 8.1.2-1ubuntu2.14
- PHP-FPM 8.1
```

### 1、安装前置依赖

```sh
sudo apt install acl zip unzip mariadb-server nginx \
      php php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
      php-curl php-json php-xml php-zip composer ntp
```

```sh
sudo apt install make pkg-config sudo debootstrap libcgroup-dev \
      php-cli php-curl php-json php-xml php-zip lsof procps
```

```sh
sudo apt install gcc g++ pypy3 openjdk-17-jdk
```

完整卸载 `apache2` 及其组件和残留文件。

```sh
sudo apt-get autoremove apache2 -y
sudo apt-get remove apache* -y
sudo apt-get --purge remove apache-common -y
sudo apt-get --purge remove apache -y
sudo find /etc -name "*apache*" |xargs  rm -rf 
sudo rm -rf /var/www
sudo rm -rf /etc/libapache2-mod-jk
sudo dpkg -l |grep apache2|awk '{print $2}'|xargs dpkg -P
```

### 2、下载 `DOMjudge` 安装包并保存在指定路径下

```sh
mkdir domjudge-8.2.2-install-files/ && cd domjudge-8.2.2-install-files/
wget https://www.domjudge.org/releases/domjudge-8.2.2.tar.gz
```

### 3、解压 `DOMjudge` 安装包

```sh
tar -zxf domjudge-8.2.2.tar.gz
```

### 4、创建安装过程所需要的用户并分配用户组

```sh
sudo useradd domjudge
sudo usermod -aG www-data domjudge
```

### 5、进入 `DOMjudge` 安装文件目录并开始安装

```sh
cd domjudge-8.2.2/
./configure --with-domjudge-user=domjudge
make domserver
sudo make install-domserver
```

### 6、`MariaDB` 数据库初始化

```sh
cd /opt/domjudge/domserver/
sudo bin/dj_setup_database genpass
# 设置 MariaDB root 账号密码
sudo mysql_secure_installation
sudo bin/dj_setup_database -u root -p <上一步中设置的 MariaDB root 账户密码> install
```

### 7、设置 `NGINX + PHP-FPM`

#### 7.1 创建配置文件软连接，并删除 `NGINX` 默认配置文件

```sh
sudo ln -s /opt/domjudge/domserver/etc/nginx-conf /etc/nginx/sites-enabled/domjudge
sudo rm -f /etc/nginx/sites-enabled/default
# 恢复 default 配置文件
# sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
sudo ln -s /opt/domjudge/domserver/etc/domjudge-fpm.conf /etc/php/8.1/fpm/pool.d/domjudge.conf
```

#### 7.2 修改配置文件

```sh
# 该文件一般不用修改，除非要配置 HTTPS
sudo nano /etc/nginx/sites-enabled/domjudge
```

以下配置文件中：

- 将 `set $prefix /domjudge;` 修改为 `set $prefix '';` ，以实现直接访问 `http(s)://<host-ip>/` 即可直接进入 `DOMjudge`
  首页；
- 解除 `location /` 整个代码块的注释，以配合上方修改；
- 注释掉整个 `location /domjudge` 和 `location /domjudge/` 两个代码块，以配合上方修改；
- 在 `location /` 代码块中的 `try_files $uri @domjudgeFront;` 下方另起一行，放入 `keepalive_timeout  0;` 以关闭长连接，减轻服务器压力。

```sh
sudo nano /opt/domjudge/domserver/etc/nginx-conf-inner
```

以下配置文件中：

- `pm.max_children` 对应的值请根据 `domserver` 实际承载量和所在实体机/VM所拥有的内存进行分配，一般按照 `40/GB`
  内存进行计算，`16GB` 的内存可设置为 `500` ；
- `php_admin_value[memory_limit]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[upload_max_filesize]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[post_max_size]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[max_file_uploads]` 的值已设置好，若有需求请自行修改；
- 取消掉 `php_admin_value[date.timezone]` 的注释，并把对应的值设置为 `Asia/Shanghai`。

```sh
sudo nano /etc/php/8.1/fpm/pool.d/domjudge.conf
```

配置文件修改完成后，重启服务

```sh
sudo service php8.1-fpm reload
sudo nginx -t
sudo service nginx reload
```

#### 8、 修改 `MariaDB` 配置

在以下配置文件中，加入或修改已有对应配置项为如下内容：

```sh
[mysqld]
max_connections = 1000
max_allowed_packet = 1024MB
innodb_log_file_size = 4096MB
```

```sh
sudo nano /etc/mysql/conf.d/mysql.cnf
```

使用 `MariaDB` 请连带修改如下配置文件中的 `max_allowed_packet` 并于上述修改中的对应配置项的对应值保持一致。

```sh
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

若要使用 `mysqldump`，请连带修改如下配置文件中的 `max_allowed_packet` 的值直至合适。

```sh
sudo nano /etc/mysql/conf.d/mysqldump.cnf
```

修改完成后请重启 `MariaDB` 服务。

```sh
sudo systemctl restart mysql
```

#### 9、获取 `admin` 账号密码

```sh
sudo cat etc/initial_admin_password.secret
```

获取到 `admin` 的账号密码后，请使用浏览器打开 `http(s)://<host-ip>/` 打开 `DOMjudge`
前端，并尝试登陆管理后台，若有必要可以修改 `admin` 账号密码，但请注意保管妥当。

## 二、`Judgehost` 安装

### 0、环境（纯净状态）

- Ubuntu 22.04.3 LTS

### 1、安装前置依赖

```sh
sudo apt install make pkg-config sudo debootstrap libcgroup-dev \
      php-cli php-curl php-json php-xml php-zip lsof procps
```

```sh
sudo apt install gcc g++ pypy3 openjdk-17-jdk
```

### 2、卸载冲突软件包

一些系统（像 `Ubuntu` ）会默认带有 `apport` 软件包，其会与判题作业产生冲突，需要卸载。

```sh
sudo apt remove apport
```

### 3、下载 `DOMjudge` 安装包并保存在指定路径下

```sh
mkdir domjudge-8.2.2-install-files/ && cd domjudge-8.2.2-install-files/
wget https://www.domjudge.org/releases/domjudge-8.2.2.tar.gz
```

### 4、解压 `DOMjudge` 安装包

```sh
tar -zxf domjudge-8.2.2.tar.gz
```

### 5、创建安装过程所需要的用户并分配用户组

```sh
sudo useradd domjudge
sudo usermod -aG www-data domjudge
```

### 6、进入 `DOMjudge` 安装文件目录并开始安装

```sh
cd domjudge-8.2.2/
./configure --with-domjudge-user=domjudge
make judgehost
sudo make install-judgehost
```

### 7、 `CPU` 条件检查

- 因为【超线程技术】、【睿频】及【 `CPU`
  功率动态调整（动态节能）】功能的存在，可能会导致同样的一发代码的运行时间存在差异。大型赛事中这样的误差通常是不可被接受的，所以我们需要在 `Judgehost`
  所在的物理机的 `BIOS` 上关闭前述功能，将 `CPU` 的时钟频率固定在一个特定值上，以使每次提交均拥有一致的判题性能基线。
- **需要特别注意的是：** 如果是在 `PVE(PROXMOX Virtual Environment)` 、 `VMware ESXi` 等虚拟化环境下，在 `VM`
  中运行以下确认超线程技术的命令时的返回值可能并**不能**代表实际情况，**请注意！**
- 运行以下命令，若返回值为 `Thread(s) per core: 1` 这说明目前每个 `CPU` 核心上只有一个线程。

```sh
lscpu | grep "Thread(s) per core"
```

- 或者也可以使用以下命令检查 `CPU` 超线程技术是否开启，若返回值为 `0` ，这说明超线程技术未被开启。

```sh
cat /sys/devices/system/cpu/smt/active
```

执行以下命令以确认 `CPU` 核心范围。这里获得的范围值将会影响下方的 `Judgehost` 实例设置命令，请注意。

```sh
cat /sys/devices/system/cpu/online
```

### 8、创建 `Judgehost` 实例所需的用户组和用户

- 创建所需用户组

```sh
sudo groupadd domjudge-run
```

- 创建所需用户。注意，命令中形如 `domjudge-run-2` 中的 `-2` 代表的是要将该 `Judgehost` 实例绑定到哪个 `CPU` 核心上。

```sh
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-2
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-3
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-4
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-5
```

### 9、赋予 `Judgehost` 实例 `sudoer` 权限

```sh
sudo ln -s /opt/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/sudoers-domjudge
```

### 10、创建 `chroot` 环境

默认情况下，`bin/dj_make_chroot` 将会使用 `Ubuntu` 官方镜像源，部分地区的速度可能较慢，这种情况下可以使用 `-m <mirror-url>`
参数来指定速度较快的镜像源。

```sh
cd /opt/domjudge/judgehost/
sudo bash bin/dj_make_chroot -m http://repo.huaweicloud.com/ubuntu
# 可选的镜像源有：
# 华为源：repo.huaweicloud.com
# Ubuntu 中国镜像源：cn.archive.ubuntu.com
# 清华源：mirrors.tuna.tsinghua.edu.cn
# 中科大源：mirrors.ustc.edu.cn
# …… ……
```

### 11、修改 `cgroups`

```sh
sudo nano /etc/default/grub
# 一些非常规版本的 Linux 系统，类如 cloud-init镜像、Google Cloud Platform 、DigitalOcean上的系统，GRUB_CMDLINE_LINUX_DEFAULT 可能会被 /etc/default/grub.d/ 下的某个配置文件覆盖，所以需要一同修改。
sudo nano /etc/default/grub.d/50-cloudimg-settings.cfg
```

修改可参考下方。其中：

- `cgroup_enable=memory swapaccount=1` 是必要的；
- `isolcpus=2,3,4,5` 代表让系统**不要**将除了绑定在这个核心上的 `Judgehost` 之外的任务分配到这个核心上、以获得更加精确的判题性能基线和更加一致的判题运行时间。
- 在一些较为新的操作系统上（例如 `Ubuntu 22.04` ），`cgroup v2`
  默认处于开启状态，因此我们还需要加入 `systemd.unified_cgroup_hierarchy=0`。

```sh
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash cgroup_enable=memory swapaccount=1 isolcpus=2,3,4,5 systemd.unified_cgroup_hierarchy=0"
```

更新完成后运行 `update-grub` 并重启。**请注意**要做好备份，修改出错的话系统将**无法正常启动**。

```sh
sudo update-grub
sudo reboot
# 重启完成后，运行以下命令以查看修改是否生效
cat /proc/cmdline
# 然后运行此命令以启动 judgehost 的 create-cgroups 服务
sudo systemctl enable create-cgroups --now
```

### 12、修改 `Judgehost` 实例连接信息

修改以下配置文件中的 `domserver` 地址和 `judgehost` 账户的密码。 `judgehost` 账户的密码可以在 `domserver`
所在机子的 `/opt/domjudge/domserver/etc/restapi.secret` 文件中获取。

```sh
sudo nano etc/restapi.secret
```

### 13、修改 `Judgehost` 实例 `systemd` 配置文件并启动服务

在以下的配置文件中，`Requires=` 后可以设置为 `domjudge-judgedaemon@2.service domjudge-judgedaemon@3.service … …`
，以实现单机多 `Judgehost` 实例。注意 `domjudge-judgedaemon@n` 中的 `n` 需要与本教程《 `Judgehost` 安装》篇中第 8 节中设置的值对应上。

```sh
sudo nano /lib/systemd/system/domjudge-judgehost.target
```

设置完后，启用 `Judgehost` 服务。以后会开机自启。

```sh
sudo systemctl enable --now domjudge-judgehost.target
```

服务启动后，可以使用以下的命令查看本机的 `Judgehost` 整体服务情况

```sh
sudo systemctl status domjudge-judgehost.target
```

可以使用以下的命令查看本机中第n个 `Judgehost` 实例的当前运行情况

```sh
sudo systemctl status domjudge-judgedaemon@n.service
```

可以使用以下的命令查看本机中第n个 `Judgehost` 实例的详细日志情况

```sh
sudo journalctl -ef -u domjudge-judgedaemon@n.service
```



