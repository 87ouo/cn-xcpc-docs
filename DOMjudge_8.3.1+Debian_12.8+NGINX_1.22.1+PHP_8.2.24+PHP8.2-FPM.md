# `DOMjudge 8.3.1` 实机安装教程

## 一、`DOMserver` 安装

### 0、环境（纯净状态）

- Debian 12.8
- NGINX 1.22.1
- PHP 8.2.24
- PHP8.2-FPM

✨✨✨以下所有操作，请使用一个 **非 `root` 账户 且 属于 `sudoers` 组（即该账户可以使用 `sudo` 命令）** 的账户来进行。✨✨✨

### 1、安装前置依赖

```shell
sudo apt install libcgroup-dev make acl zip unzip pv mariadb-server nginx \
      php php-fpm php-gd php-cli php-intl php-mbstring php-mysql php-curl \
      php-json php-xml php-zip composer ntp python3-yaml pkg-config \
      debootstrap lsof procps gcc g++ pypy3 openjdk-17-jdk
```

完整卸载 `apache2` 及其组件和残留文件。

```shell
sudo apt-get autoremove apache2 -y
sudo apt-get remove apache* -y
sudo apt-get --purge remove apache-common -y
sudo apt-get --purge remove apache -y
sudo find /etc -name "*apache*" |xargs sudo rm -rf 
sudo rm -rf /var/www
sudo rm -rf /etc/libapache2-mod-jk
sudo dpkg -l |grep apache2|awk '{print $2}'|xargs sudo dpkg -P
sudo apt autoremove -y
```

### 2、下载 `DOMjudge` 安装包并保存在指定路径下

```shell
mkdir domjudge-8.3.1-install-files/ && cd domjudge-8.3.1-install-files/
wget https://www.domjudge.org/releases/domjudge-8.3.1.tar.gz
```

### 3、解压 `DOMjudge` 安装包

```shell
tar -zxf domjudge-8.3.1.tar.gz
```

### 4、进入 `DOMjudge` 安装文件目录并开始安装

```shell
cd domjudge-8.3.1/
./configure --prefix=/opt/domjudge
make domserver
sudo make install-domserver
```

### 5、`MariaDB` 数据库初始化

```shell
cd /opt/domjudge/domserver/bin/
./dj_setup_database genpass
sudo ./dj_setup_database -s install
```

### 6、设置 `NGINX + PHP-FPM`

#### 6.1 创建配置文件软连接，并删除 `NGINX` 默认配置文件

```shell
sudo ln -s /opt/domjudge/domserver/etc/nginx-conf /etc/nginx/sites-enabled/domjudge
sudo rm -f /etc/nginx/sites-enabled/default
# 恢复 default 配置文件
# sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
sudo ln -s /opt/domjudge/domserver/etc/domjudge-fpm.conf /etc/php/8.2/fpm/pool.d/domjudge.conf
```

#### 6.2 修改配置文件

```shell
# 该文件一般不用修改，除非要配置 HTTPS
sudo nano /etc/nginx/sites-enabled/domjudge
```

以下配置文件中：

- 将 `set $prefix /domjudge;` 修改为 `set $prefix '';` ，以实现直接访问 `http(s)://<host-ip>/` 即可直接进入 `DOMjudge` 首页；
- 解除 `location /` 整个代码块的注释，以配合上方修改；
- 注释掉整个 `location /domjudge` 和 `location /domjudge/` 两个代码块，以配合上方修改。

```shell
sudo nano /opt/domjudge/domserver/etc/nginx-conf-inner
```

以下配置文件中：

- `pm.max_children` 对应的值请根据 `domserver` 实际承载量和所在实体机/VM所拥有的内存进行分配，一般按照 `40/GB` 内存进行计算，`16GB` 的内存可设置为 `500` ；
- `php_admin_value[memory_limit]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[upload_max_filesize]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[post_max_size]` 的值已设置好，若有需求请自行修改；
- `php_admin_value[max_file_uploads]` 的值已设置好，若有需求请自行修改；
- 取消掉 `php_admin_value[date.timezone]` 的注释，并把对应的值设置为 `Asia/Shanghai`。

```shell
sudo nano /etc/php/8.2/fpm/pool.d/domjudge.conf
```

配置文件修改完成后，重启服务

```shell
sudo service php8.2-fpm reload
sudo nginx -t
sudo service nginx reload
```

### 7、 修改 `MariaDB` 配置

在以下配置文件中，加入或修改已有对应配置项为如下内容：

```shell
[mysqld]
max_connections = 1000
max_allowed_packet = 1024MB
innodb_log_file_size = 4096MB
```

```shell
sudo nano /etc/mysql/conf.d/mysql.cnf
```

使用 `MariaDB` 请连带修改如下配置文件中的 `max_allowed_packet` 和 `max_connections` 并于上述修改中的对应配置项的对应值保持一致。

```shell
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

若要使用 `mysqldump`，请连带修改如下配置文件中的 `max_allowed_packet` 的值直至合适。

```shell
sudo nano /etc/mysql/conf.d/mysqldump.cnf
```

修改完成后请重启 `MariaDB` 服务。

```shell
sudo systemctl restart mysql
```

### 8、获取 `admin` 账号密码

```shell
sudo cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

获取到 `admin` 的账号密码后，请使用浏览器打开 `http(s)://<host-ip>/` 打开 `DOMjudge` 前端，并尝试登陆管理后台，若有必要可以修改 `admin` 账号密码，但请注意保管妥当。

### 9、一些使用 `DOMserver` 时候的注意事项

#### 9.1 通过 `API` 使用 `JSON` 进行座位信息导入

需要特别注意，从 `8.3` 版本开始，使用 `JSON` 方式通过 `API` 进行导入时，需要注意应使用 `location.description` 的 `API` 路径，而非以前版本所使用的 `room` 路径。

- `8.3`版本的示例如下：
  
  ```
  [{
  "id": "1",
  "icpc_id": "447047",
  "label": "1",
  "group_ids": ["24"],
  "name": "¡i¡i¡",
  "organization_id": "INST-42",
  "location": {"description": "AUD 10"}
  }, {
  "id": "2",
  "icpc_id": "447837",
  "label": "2",
  "group_ids": ["25"],
  "name": "Pleading not FAUlty",
  "organization_id": "INST-43"
  }]
  ```

- 旧版本的示例如下：
  
  ```
  [{
  "id": "1",
  "icpc_id": "447047",
  "group_ids": ["24"],
  "name": "¡i¡i¡",
  "organization_id": "INST-42",
  "room": "AUD 10"
  }, {
  "id": "2",
  "icpc_id": "447837",
  "group_ids": ["25"],
  "name": "Pleading not FAUlty",
  "organization_id": "INST-43"
  }]
  ```

#### 9.2 重新生成 `domjudge-team-manual.pdf`

```shell
# For Debian Bullseye and above:
sudo apt install python3-sphinx python3-sphinx-rtd-theme rst2pdf fontconfig python3-yaml latexmk
# For Ubuntu Jammy Jellyfish and above:
sudo apt install python-sphinx python-sphinx-rtd-theme rst2pdf fontconfig python3-yaml latexmk


# 安装texlive最新版
sudo apt autoremove texlive*
mkdir texlive-2024
cd texlive-2024
sudo apt install perl wget fontconfig xzdec
wget http://mirror.ctan.org/systems/texlive/tlnet/install-tl-unx.tar.gz
tar -xf install-tl-unx.tar.gz
cd install-tl*/
sudo ./install-tl
# 安装需时较长，请耐心等待


sudo tlmgr init-usertree
sudo tlmgr update --all
tlmgr install cmap
# 查看安装日志末尾提示信息，找到需要添加进PATH的路径，例如：/usr/local/texlive/2024/bin/x86_64-linux
nano ~/.bashrc
# 在文件末尾添加一条记录，例如：export PATH="/usr/local/texlive/2024/bin/x86_64-linux:$PATH"，然后保存并退出
source ~/.bashrc
tlmgr --version    # 确认安装是否成功


# 进入<DOMjudge安装文件解压路径>/doc/，例如：cd /home/ouo/domjudge-8.3.1-install-files/doc/
make docs
# 待命令执行完毕后，在 <DOMjudge安装文件解压路径>/doc/manual/build/ 下面即可找到生成出来的 domjudge-team-manual.pdf
# 需要注意，PDF中的 <BASE_URL> 会根据在安装 domserver 时的设置来填入，所以可以根据本文 《DOMserver 安装》部分中的第4节的内容，在安装文件路径下重新执行 ./configure --prefix=/opt/domjudge --with-baseurl=<BASE_URL> 来设置 <BASE_URL>，例如可以将前述命令中的 <BASE_URL> 替换成 http://192.168.52.131/ 。
# PDF生成模板位于 <DOMjudge安装文件解压路径>/doc/manual/team.rst，可以根据自身实际需要进行修改。
```

## 二、`Judgehost` 安装

### 0、环境（纯净状态）

- Debian 12.8

### 1、安装前置依赖

```shell
sudo apt install make pkg-config debootstrap libcgroup-dev \
      php-cli php-curl php-json php-xml php-zip lsof procps \
      gcc g++ pypy3 openjdk-17-jdk
```

### 2、卸载冲突软件包

一些系统（像 `Ubuntu` ）会默认带有 `apport` 软件包，其会与判题作业产生冲突，需要卸载。

```shell
sudo apt remove apport
```

### 3、下载 `DOMjudge` 安装包并保存在指定路径下

```shell
mkdir domjudge-8.3.1-install-files/ && cd domjudge-8.3.1-install-files/
wget https://www.domjudge.org/releases/domjudge-8.3.1.tar.gz
```

### 4、解压 `DOMjudge` 安装包

```shell
tar -zxf domjudge-8.3.1.tar.gz
```

### 5、进入 `DOMjudge` 安装文件目录并开始安装

```shell
cd domjudge-8.3.1/
./configure --prefix=/opt/domjudge
make judgehost
sudo make install-judgehost
```

### 6、 `CPU` 条件检查

- 因为【超线程技术】、【睿频】及【 `CPU` 功率动态调整（动态节能）】功能的存在，可能会导致同样的一发代码的运行时间存在差异。大型赛事中这样的误差通常是不可被接受的，所以我们需要在 `Judgehost` 所在的物理机的 `BIOS` 上关闭前述功能，将 `CPU` 的时钟频率固定在一个特定值上，以使每次提交均拥有一致的判题性能基线。
- **需要特别注意的是：** 如果是在 `PVE(PROXMOX Virtual Environment)` 、 `VMware ESXi` 等虚拟化环境下，在 `VM` 中运行以下确认超线程技术的命令时的返回值可能并**不能**代表实际情况，**请注意！**
- 运行以下命令，若返回值为 `Thread(s) per core: 1` 这说明目前每个 `CPU` 核心上只有一个线程。

```shell
lscpu | grep "Thread(s) per core"
```

- 或者也可以使用以下命令检查 `CPU` 超线程技术是否开启，若返回值为 `0` ，这说明超线程技术未被开启。

```shell
cat /sys/devices/system/cpu/smt/active
```

执行以下命令以确认 `CPU` 核心范围。这里获得的范围值将会影响下方的 `Judgehost` 实例设置命令，请注意。

```shell
cat /sys/devices/system/cpu/online
```

**附**：针对`PVE (PROXMOX Virtual Environment)` 环境下的使用场景，我们可以在 `PVE` 的 `shell` 内使用 `linux-cpupower` 设定 `CPU` 的工作模式，以将 `CPU` 频率维持在一个特定水平而不进行动态调整。

    a. 安装 `linux-cpupower` 

```shell
sudo apt install linux-cpupower
```

    b. 确认 当前 `PVE` 支持的`CPU`电源策略 和 当前的`CPU`电源策略（ `PVE` 默认情况下使用 `performance` ）

```shell
# 查看支持的 CPU 电源模式
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# 查看当前的 CPU 电源模式
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

    c. `CPU`电源策略解释

| 电源模式           | 解释说明                                                      |
|:-------------- | --------------------------------------------------------- |
| `performance`  | 性能模式，将 `CPU` 频率固定工作在其支持的较高运行频率上，而不动态调节。                   |
| `userspace`    | 系统将变频策略的决策权交给了用户态应用程序，较为灵活。                               |
| `powersave`    | 省电模式，`CPU` 会固定工作在其支持的最低运行频率上。                             |
| `ondemand`     | 按需快速动态调整 `CPU` 频率，没有负载的时候就运行在低频，有负载就高频运行。                 |
| `conservative` | 与 `ondemand` 不同，平滑地调整 `CPU` 频率，频率的升降是渐变式的，稍微缓和一点。         |
| `schedutil`    | 负载变化回调机制，后面新引入的机制，通过触发 `schedutil` `sugov_update` 进行调频动作。 |

    d. 设置`CPU`电源策略

```shell
# 例1: 设置所有CPU为 性能模式
sudo cpupower -c all frequency-set -g performance


# 例2: 设置所有CPU为 节能模式
sudo cpupower -c all frequency-set -g powersave
```

### 7、创建 `Judgehost` 实例所需的用户组和用户

- 创建所需用户组

```shell
sudo groupadd domjudge-run
```

- 创建所需用户。注意，命令中形如 `domjudge-run-2` 中的 `-2` 代表的是要将该 `Judgehost` 实例绑定到哪个 `CPU` 核心上。

```shell
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-2
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-3
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-4
sudo useradd -d /nonexistent -g domjudge-run -M -s /bin/false domjudge-run-5
```

### 8、赋予 `Judgehost` 实例 `sudoer` 权限

```shell
sudo ln -s /opt/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/sudoers-domjudge
```

### 9、创建 `chroot` 环境

默认情况下，`bin/dj_make_chroot` 将会使用 `Debian` 官方镜像源（这个将依据你所使用的发行版不同而不同），部分地区的速度可能较慢，这种情况下可以使用 `-m <mirror-url>` 参数来指定速度较快的镜像源。

```shell
cd /opt/domjudge/judgehost/
sudo bash bin/dj_make_chroot -m https://mirrors.ustc.edu.cn/debian
# 可选的镜像源有：
# 华为源：repo.huaweicloud.com
# Ubuntu 中国镜像源：cn.archive.ubuntu.com
# 清华源：mirrors.tuna.tsinghua.edu.cn
# 中科大源：mirrors.ustc.edu.cn
# …… ……
```

### 10、修改 `cgroups`

```shell
sudo nano /etc/default/grub
# 一些非常规版本的 Linux 系统，类如 cloud-init镜像、Google Cloud Platform 、DigitalOcean上的系统，GRUB_CMDLINE_LINUX_DEFAULT 可能会被 /etc/default/grub.d/ 下的某个配置文件覆盖，所以需要一同修改。
sudo nano /etc/default/grub.d/50-cloudimg-settings.cfg
```

修改操作参考下方提示：

- `cgroup_enable=memory swapaccount=1` 是必要的；
- `isolcpus=2,3,4,5` 代表让系统**不要**将除了绑定在这几个核心上的 `Judgehost` 之外的任务分配到这几个核心上、以获得更加精确的判题性能基线和更加一致的判题运行时间；
- 在一些较为新的发行版上（例如 `Debian Bullseye` 及后续版本、`Ubuntu Jammy Jellyfish` 及后续版本 ），`cgroup v2` 默认处于开启状态，因此我们还需要加入 `systemd.unified_cgroup_hierarchy=0`；
- 如果运行环境中的 `systemd` 的版本 ≥ `v257`，则还需要加入 `SYSTEMD_CGROUP_ENABLE_LEGACY_FORCE=1`。

```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash cgroup_enable=memory swapaccount=1 isolcpus=2,3,4,5 systemd.unified_cgroup_hierarchy=0"
```

更新完成后运行 `update-grub` 并重启。**请注意**要做好备份（ `PVE` 则请先打好 **快照** ），修改出错的话系统将**无法正常启动**。

```shell
sudo update-grub
sudo reboot
# 重启完成后，运行以下命令以查看修改是否生效
cat /proc/cmdline
# 然后运行此命令以启动 judgehost 的 create-cgroups 服务
sudo systemctl enable create-cgroups --now
```

### 11、修改 `Judgehost` 实例连接信息

修改以下配置文件中的 `domserver` 地址和 `judgehost` 账户的密码。 `judgehost` 账户的密码可以在 `domserver` 所在机子的 `/opt/domjudge/domserver/etc/restapi.secret` 文件中获取。

```shell
sudo nano /opt/domjudge/judgehost/etc/restapi.secret
```

### 12、修改 `Judgehost` 实例 `systemd` 配置文件并启动服务

在以下的配置文件中，`Requires=` 后可以设置为 `domjudge-judgedaemon@2.service domjudge-judgedaemon@3.service … …`，以实现单机多 `Judgehost` 实例。注意 `domjudge-judgedaemon@n` 中的 `n` 需要与本文中《 `Judgehost` 安装》部分的第 7 节中设置的值对应上。

```shell
sudo nano /lib/systemd/system/domjudge-judgehost.target
```

设置完后，启用 `Judgehost` 服务。以后会开机自启。

```shell
sudo systemctl enable --now domjudge-judgehost.target
```

服务启动后，可以使用以下的命令查看本机的 `Judgehost` 整体服务情况

```shell
sudo systemctl status domjudge-judgehost.target
```

可以使用以下的命令查看本机中第n个 `Judgehost` 实例的当前运行情况

```shell
sudo systemctl status domjudge-judgedaemon@n.service
```

可以使用以下的命令查看本机中第n个 `Judgehost` 实例的详细日志情况

```shell
sudo journalctl -ef -u domjudge-judgedaemon@n.service
```
