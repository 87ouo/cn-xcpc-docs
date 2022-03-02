# DOMjudge 8.0.0 安装步骤

<br />
<br />
<br />

## A.	环境及版本

- DOMjudge 8.0.0
- Ubuntu 20.04
- Nginx 1.18
- PHP 7.4.3
- PHP7.4-FPM

<br />
<br />
<br />

## B.	安装步骤及备注


### 一、	安装前置

```sh
sudo apt install acl zip unzip mariadb-server nginx php-fpm php-gd php-cli php-intl php-mbstring php-mysql php-curl php-json php-xml php-zip composer ntp make pkg-config sudo debootstrap libcgroup-dev php-cli php-curl php-json php-xml php-zip lsof procps gcc g++ cmake pypy3 libcurl4-gnutls-dev libjsoncpp-dev
```

<br />
<br />


### 二、	下载、解压安装文件

##### 2.1	 新建一个目录，用于存放 `DOMjudge` 安装文件，并进入该目录。

```sh
mkdir <path_to_save_install_files_folder>
cd <path_to_save_install_files_folder>
```

<br />

##### 2.2	下载 `DOMjudge 8.0.0` 安装文件，并解压至当前目录下。

```sh
wget https://www.domjudge.org/releases/domjudge-8.0.0.tar.gz
tar -zxvf domjudge-8.0.0.tar.gz
```

<br />
<br />

### 三、	安装 `domserver`

##### 3.1	进入解压后得到的安装文件目录。

```sh
cd <path_to_save_install_files_folder>/domjudge-8.0.0/
```

<br />

##### 3.2	安装 `domserver` 。

```sh
./configure --prefix=<path_to_install_domserver> --with-baseurl=127.0.0.1
make domserver
sudo make install-domserver
```

<br />

##### 3.3	进入安装后得到的 `domserver` 目录。

```sh
cd <path_to_install_domserver>/domserver
```

<br />

##### 3.4	执行数据库Setup脚本。

```sh
sudo bin/dj_setup_database -u root install
```

<br />
<br />

### 四、	配置WEB服务器

##### 4.1	建立配置文件软连接

```sh
sudo ln -s <path_to_install_domserver>/domserver/etc/nginx-conf /etc/nginx/sites-enabled/domjudge
sudo ln -s <path_to_install_domserver>/domserver/etc/domjudge-fpm.conf /etc/php/7.4/fpm/pool.d/domjudge.conf
```

<br />

##### 4.2	编辑配置文件

编辑下列文件，以适合所需要的配置：
`/etc/nginx/sites-enabled/domjudge` 、 `/etc/php/7.4/fpm/pool.d/domjudge.conf` 和 `/etc/php/7.4/fpm/php.ini` 

其中，这些配置文件中的以下的项目需要取消注释，并设置成对应且合适的值：

```sh
pm.max_children = 320    ;(每 1GB RAM ≈ 40，16GB RAM 则可设置成 500 )
php_admin_value[date.timezone] = Asia/Shanghai
php_admin_value[max_file_uploads] = 110
php_admin_value[memory_limit] = 1024M
php_admin_value[upload_max_filesize] = 512M
php_admin_value[post_max_size] = 512M
max_execution_time = 300
```

然后编辑 `<path_to_install_domserver>/domserver/etc/nginx-conf-inner` ，在其中 `location /domjudge/` 部分的最后添加 `keepalive_timeout  0;` ，修改后类似以下所示。
```sh
location /domjudge/ {
        root $domjudgeRoot;
        rewrite ^/domjudge/(.*)$ /$1 break;
        try_files $uri @domjudgeFront;
        keepalive_timeout  0;
}
```
修改完成并确认无误后，保存并退出编辑。


<br />

##### 4.3	备份 `Nginx` 原 `default` 配置文件，并删除原 `default` 配置文件

```sh
sudo cp /etc/nginx/sites-enabled/default <path_to_save_conf_backup>/default-nginx-backup1
sudo rm -f /etc/nginx/sites-enabled/default
```

<br />

##### 4.4	重载 `php-fpm` 和 `nginx`

```
sudo service php7.4-fpm reload
sudo service nginx reload
```

<br />
<br />

### 五、	配置数据库

编辑 `/etc/mysql/conf.d/mysql.cnf` 修改以下项目为合适的值（若没有这些项，请按照原配置文件所提供的格式进行添加）。以下仅为参考。

```sh
[mysqld]
max_connections = 1000
max_allowed_packet = 512MB
innodb_log_file_size = 512MB
```

如果你使用的是 `MariaDB` 而非 `MySQL` ，你还需要修改 `/etc/mysql/mariadb.conf.d/50-server.cnf` 中的 `max_allowed_packet` 项的值，直至合适。

另外，如果你需要使用 `mysqldump` 来对数据库进行备份导出，你还需要修改 `/etc/mysql/conf.d/mysqldump.cnf` 中的 `max_allowed_packet` 项的值，直至合适。

最后，重启数据库服务。

```sh
sudo systemctl restart mysql
```

<br />
<br />

### 六、	获取 `admin` 密码

运行一下命令，即可在终端中打印出 `admin` 账号的密码，你可以用它在 `domserver` 前端的 `login` 页面登陆 `admin` 账号以进入 `DOMjudge` 后端管理页面。

```sh
cat <path_to_install_domserver>/domserver/etc/initial_admin_password.secret
```

<br />

至此，`domserver` 已经安装完成并可使用，如若有其他自定义设置，请自行设置相关的配置文件。

接下来将安装 `judgehost` ，需要注意的是： `judgehost` 可以和 `domserver` **分开**安装，既可以不需要安装在同一个系统，或同一台物理机上。同样， `judgehost` 可以有多个；一个 `judgehost` 也可以服务于多个 `domserver` ； `domserver` 也可以考虑采用负载均衡，但请注意 `judgehost` 和 `domserver` 的对应关系。

需要注意的是，基于实际使用发现，不宜创建过多的 `judgehost` ，因为 `judgehost` 也是通过轮询的方式获取 `submission` 进行判题的，过多的 `judgehost` 会导致前端访问压力过大，容易导致崩溃或带宽耗尽等情况。所以请结合实际情况和服务器性能，合理部署适当数量的 `judgehost` 。

根据北京理工的经验，对于 `domserver` ，在千兆网络、物理机36 核 80G的情况下， 平均 CPU 占用不超过 50%，内存也不超过 3G。对于 `judgehost` ，开 11 个左右就能挡住 5k 提交了（虽然是在手动管理测评机池的情况下）。

根据黑龙江大学的经验，千兆内网、`domserver` 与 `judgehost` 所在实体机分离的情况下，`domserver` 分配 6 核 16G、`judgehost` 分别在 2 台实体机上各运行 2 个，且 2 台实体机各为 4 核 6G，对于 150+ teams、 5H 内大概 3K 的提交，没有问题（题目的测试数据点均在 30 个以内，题目内存限制均在 512MB 以内，题目时间限制均在 2s 以内）。

**接下来进行 `judgehost` 的安装。** 

<br />
<br />

### 七、	安装 `judgehost`

##### 7.1	安装前置。（如果 `judgehost` 不与 `domserver` 在同一系统中时）

```sh
sudo apt install make pkg-config sudo debootstrap libcgroup-dev php-cli php-curl php-json php-xml php-zip lsof procps gcc g++ cmake pypy3 libcurl4-gnutls-dev libjsoncpp-dev
```

<br />

##### 7.2	 新建一个目录，用于存放 `DOMjudge` 安装文件，并进入该目录。（如果 `judgehost` 不与 `domserver` 在同一系统中时）

```sh
mkdir <path_to_save_install_files_folder>
cd <path_to_save_install_files_folder>
```

<br />

##### 7.3	下载 `DOMjudge 8.0.0` 安装文件，并解压至当前目录下。（如果 `judgehost` 不与 `domserver` 在同一系统中时）

```sh
wget https://www.domjudge.org/releases/domjudge-8.0.0.tar.gz
tar -zxvf domjudge-8.0.0.tar.gz
```

<br />

##### 7.4	进入解压后得到的安装文件目录。

```sh
cd <path_to_save_install_files_folder>/domjudge-8.0.0/
```

<br />

##### 7.5	安装 `judgehost` 。

```sh
./configure --prefix=<path_to_install_judgehost> --with-baseurl=127.0.0.1
make judgehost
sudo make install-judgehost
```

<br />

##### 7.6	确认是否启用了处理器核心超线程技术

可能会造成评测的运行时间波动，请斟酌是否要启用该技术。

```sh
sudo lscpu | grep "Thread(s) per core"
```

若得到一个大于 `1` 的值，即代表启用了处理器核心的超线程技术。

或运行以下命令，若得到一个大于 `1` 的值，也代表启用了处理器核心的超线程技术。

```sh
sudo cat /sys/devices/system/cpu/smt/active
```

<br />

##### 7.7	确认系统核心可用范围

```sh
sudo cat /sys/devices/system/cpu/online
```

你将会得到一个 `0-x` 的范围，在下面的绑定 `judgehost` 和处理器核心的操作中，你需要注意：绑定的核心下标范围**不能**超过这里得到的范围。

<br />

##### 7.8	添加 `judgehost` 运行所需要的用户

以下命令必须运行。

```sh
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run
```

接下来的命令仅为在同一系统中配置多台评测机并分别绑定处理器核心的操作而准备。

```sh
sudo useradd -d /nonexistent -U -M -s /bin/false domjudge-run-2
... ...
```

其中后缀 `-2` 代表该 `judgehost` 绑定在处理器的下标为 `2` 的核心上，这个由后续章节【运行 `judgehost` 】中的启动某台 `judgehost` 的命令中的 ` -n 2` 参数设定。

可以设定多组绑定，仅需要修改好该后缀及在后续章节【运行 `judgehost` 】中修改好相关的启动命令即可。

<br />

##### 7.9	允许 `judgehost` 使用所需的 `sudo` 命令

```sh
sudo ln -s <path_to_install_judgehost>/judgehost/etc/sudoers-domjudge /etc/sudoers.d/sudoers-domjudge
```

<br />

##### 7.10	修改 `grub`

```sh
sudo nano /etc/default/grub
```

在 `GRUB_CMDLINE_LINUX_DEFAULT` 原有的参数后新加入 `cgroup_enable=memory swapaccount=1` ；特别地，如果希望评测对运行时间的判定更加准确，可以在此处加入 ` isolcpus=2` 来将对应的处理器核心孤立开来（如果想孤立出更多的核心，可以修改成形如 `isolcpus=2,3,4` 的样子并添加进去）；如果在一些较新的分支版本中，已经默认启用了 *cgroup v2* 的话，还需要添加入 `systemd.unified_cgroup_hierarchy=0` 。

最后，作为参考，形如以下。

```sh
GRUB_CMDLINE_LINUX_DEFAULT="maybe-ubiquity cgroup_enable=memory swapaccount=1 isolcpus=2 systemd.unified_cgroup_hierarchy=0"
```

保存好后，运行以下指令。

```sh
sudo update-grub
```

待指令运行完成后，重启系统（**建议做好备份**，如果修改出现差错，系统将会**无法成功启动**）。

```sh
sudo reboot
```

重启完后，可以运行以下命令以检查修改是否生效。

```sh
cat /proc/cmdline
```

如果终端中显示的 `GRUB_CMDLINE_LINUX_DEFAULT` 中对应的参数正如你此前设置的，即意味着修改成功。

需要注意的是，在一些云服务器系统中（如 *Google Cloud* 、 *DigitalOcean* ），`GRUB_CMDLINE_LINUX_DEFAULT` 可能会被 `/etc/default/grub.d/` 中的某些配置文件所覆盖，你需要找出这些配置文件并修改成所需要的样子。



最后，运行 `judgehost` 的 `create-cgroups` 服务。

```sh
sudo systemctl enable create-cgroups --now
```

<br />

##### 7.11	设置 `chroot` 环境

如果在国内，原有脚本中的源的速度可能会比较慢，我们可以使用以下命令将其替换成国内速度较快的镜像源。

```sh
sudo sed -i 's,http://us.archive.ubuntu.com/ubuntu/,http://mirrors.tuna.tsinghua.edu.cn/ubuntu,g' <path_to_install_judgehost>/judgehost/bin/dj_make_chroot
```

可选的镜像源有：

- 清华源 `mirrors.tuna.tsinghua.edu.cn` 
- 中科大源 `mirrors.ustc.edu.cn` 
- 阿里源 `mirrors.aliyun.com` （如果你在阿里云ECS上运行，推荐使用 `mirrors.cloud.aliyuncs.com` ）
- `cn.archive.ubuntu.com` 
- `azure.archive.ubuntu.com` （Azure上推荐） 
- … …

请自行斟酌使用。

配置好镜像源后，运行该脚本。

```sh
sudo bash <path_to_install_judgehost>/judgehost/bin/dj_make_chroot
```

<br />

##### 7.12	配置 `judgehost` 所需的连接信息

打开 `judgehost` 配置文件。

```sh
sudo nano <path_to_install_judgehost>/judgehost/etc/restapi.secret
```

其中 `<API url>` 可以在 `domserver` 的 `admin` 页面的 `Config checker` 中得到。

其中 `<user>` 和 `<password>` 必须对应的上 `judgehost` 账户的信息，其密码可以在 `domserver` 中的 `<path_to_install_domserver>/domserver/etc/restapi.secret` 文件中获取。

如果想要设置一台 `judgehost` 服务多台 `domserver` ，可以在 `judgehost` 配置文件添加多行，分别对应多台 `domserver` 的信息。其中需要注意的是，每行的 `<ID>` 均不能出现行间重复。

<br />

##### 7.13	设置守护和自启动

修改以下配置文件，以适合你的配置所需。

```sh
sudo nano /lib/systemd/system/domjudge-judgehost.target
```

然后运行以下命令以让守护生效。

```sh
sudo systemctl enable --now domjudge-judgehost.target
```

完成后，你可以运行 `sudo systemctl status domjudge-judgehost.target` 以查看运行情况。

也可以运行 `sudo journalctl -u domjudge-judgedaemon@n` 以查看本系统中第 `n` 台 `judgehost` 的运行情况和记录。

<br />
<br />
<br />

## C.	收尾工作

检查 `domserver` 的 `admin` 页面中的 `Config checker` ，如果为全绿，即为正常；若由黄色项或红色项，请自行排查修复。

此处给出一些参考资料，希望能对你有帮助。

- cn-xcpc-tools 文档： **cn-xcpc-docs**

    ```
    https://github.com/cn-xcpc-tools/cn-xcpc-docs
    ```

- **DOMjudge 官方手册**

    ```
    https://www.domjudge.org/docs/manual/8.0/index.html#
    ```

<br />
<br />
<br />

## D.	附加

如果需要使用 `apache2 + php-fpm` ，请：

- 将 【**A.	安装前置**】 中的 **`nginx`** 更换为 **`apache2 php`** 。

- 4.1中命令更换为下列命令。

    ```sh
    sudo ln -s <path_to_install_domserver>/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
    sudo ln -s <path_to_install_domserver>/domserver/etc/domjudge-fpm.conf /etc/php/7.4/fpm/pool.d/domjudge.conf
    sudo a2enmod proxy_fcgi setenvif rewrite
    sudo a2enconf php7.4-fpm domjudge
    ```

- 4.2中修改的配置文件更改为： `/etc/apache2/conf-available/domjudge.conf` 、 `/etc/php/7.4/fpm/pool.d/domjudge.conf` 、 `/etc/php/7.4/fpm/php.ini` 。

    修改项目和值的要求如下。

    ```
    [max_file_uploads      110]
    [upload_max_filesize   512M]
    [post_max_size         512M]
    [memory_limit          1024M]
    ```

    若使用 `nano` ，可以使用 *ctrl+w* 来进行搜索。找到 `date.timezone` 并取消注释，并设置成 `Asia/Shanghai` ；找到 `max_execution_time` 并将对应的值设置为 `300` 或以上，以防止生成队伍密码时发生超时。

    <br />    

    编辑 `/etc/apache2/apache2.conf`，搜索 `KeepAlive` 关键字，将其值设为 `Off`，并在其后新增一行内容：
    ```conf
    MaxClients 1000
    ```
    然后重启 `apache2` 服务，使修改生效。
    ```shell
    sudo systemctl restart apache2
    ```


- 4.3不需要操作

- 4.4运行的命令更改为如下命令。

    ```sh
    sudo service php7.4-fpm reload
    sudo service apache2 reload
    ```

    
