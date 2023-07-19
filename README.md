# debian_rootfs

```sh

$ sudo apt-get install debian-archive-keyring
安装所需依赖，使用debootstrap命令创建文件系统。创建个工作目录。
# 新建工作目录 build
$ mkdir build && cd build
# 安装必要依赖 debootstrap就是构建的命令
$ sudo apt-get install qemu qemu-user-static binfmt-support debootstrap
# 切换到root
$ su
# 构建文件系统的命令
$ debootstrap --arch=arm64 --foreign bullseye linux-rootfs http://ftp.cn.debian.org/debian/
# 切换到普通用户
$ su [user]
# qemu-aarch64-static是其中的关键，能在 x86_64 主机系统下 chroot 到 arm64 文件系统
$ sudo cp -a /usr/bin/qemu-aarch64-static build/linux-rootfs/usr/bin/qemu-aarch64-static

```
--arch：指定制作的文件系统是什么架构的

--foreign：在与主机架构不相同时需要指定此参数，仅做初始化的解包

stretch：这个是Debian 9的发行版本号，为什么没用最新的Debian 10的buster，因为更换国内的镜像源总是有点问题Debian 发行版

linux-rootfs：这个是要存放文件系统的文件夹，可以不用先创建，执行上述命令会自动创建此文件夹，也可以先创建

ftp.cn.debian.org/debian/ ：这个是中国镜像服务器地址，Debian 全球镜像站

接下来我们可以通过chroot进入到制作好的文件系统。 chroot wiki

这里提供一个脚本文件来进入我们的根文件系统，最好不要使用root用户执行此脚本 ch-mount.sh

```sh
# 此脚本有两个参数  -u 是取消挂载  -m 是挂载，为什么要挂载本机的设备文件，我也不太清楚
$ ./ch-mount.sh -m linux-rootfs
# 执行脚本后，没有报错会进入文件系统，显示 I have no name ，这是正常的，不要慌张，我当时就有点懵逼，这是因为还没有初始化。
I have no name!@node2:/#
# 以下命令是在根文件系统中执行的命令
# 进行第二步，初始化文件系统，会把一个系统的基础包等全部初始化
$ debootstrap/debootstrap --second-stage
# 初始化好了以后，退出文件系统，再次进入后就显示root了
$ exit
# 再次进入时，不需要执行脚本，使用chroot命令即可，因为ch-mount脚本是为了挂载本机文件与文件系统的关联而已
$ sudo chroot linux-rootfs


```

如果脚本报错 ：/bin/sh^M:bad interpreter: No such file or directory，这是因为文件格式的错误，可通过以下方式解决

```sh
$ vim ch-mount.sh
# 设置文件格式为unix 然后保存退出
:set ff=unix
:wq
```

## 定制文件系统

上面完成后，现在在文件系统内，需要对文件系统进行一些DIY，安装一些我们必要的工具，和配置网络等等，系统为Debain 9，其他发行版请自行更换命令。
要确保进入文件系统后有网络，一般如果没有网络，可以先把 /etc/resolv.conf 文件拷贝到 linux-rootfs/etc/resolv.conf，因为我构建的文件系统会默认把本机的 resolv.conf 拷贝进去，所以我没有手动拷贝。

## 更换国内镜像源

```sh
# 如果遇到无法拉取 https 源的情况，请先使用 http 源并安装
$ apt install apt-transport-https
$ cp /etc/apt/source.list /etc/apt/source.list_bak
# 这里用的vim.tiny是构建文件系统是自带的，跟vim一样也可以编辑文件，把文件内容全部替换为以下内容 https://mirrors.tuna.tsinghua.edu.cn/help/debian/
$ vim.tiny /etc/apt/source.list

# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free

deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free

deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free

deb http://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
deb-src http://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free

# deb http://security.debian.org/debian-security bullseye-security main contrib non-free
# deb-src http://security.debian.org/debian-security bullseye-security main contrib non-free

```
如果报以下错误:
```sh

root@rock-5b:/# apt-get update
Err:1 https://mirrors.tuna.tsinghua.edu.cn/debian bullseye InRelease
  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 101.6.15.130 443]
Err:2 https://mirrors.tuna.tsinghua.edu.cn/debian bullseye-updates InRelease
  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 101.6.15.130 443]
Err:3 https://mirrors.tuna.tsinghua.edu.cn/debian bullseye-backports InRelease
  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 101.6.15.130 443]
Err:4 https://security.debian.org/debian-security bullseye-security InRelease
  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 151.101.130.132 443]
Reading package lists... Done

```
需要使用http的链接，不能使用https的。


## 配置root用户密码
```sh
# 先ping www.baidu.com 看下是否有网络,没有网络需要退出文件系统，把宿主机的reslov.conf文件拷贝到相应位置即可。
$ ping www.baidu.com
$ apt-get update
# 先设置root用户的密码
$ passwd
```
## 创建一个普通用户

```sh
# 这两个环境变量可以自行修改
$ USER=robin
$ HOST=robin
$ useradd -G sudo -m -s /bin/bash $USER
$ passwd $USER
```

设置时区和设置使能串口，打开树莓派的硬件串口，可以通过USB转串口模块登录终端

```sh
# 设置时区
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 使能串口，如果只是为了制作根文件系统，可以不用运行这个命令，我是为了树莓派的串口调试才开启的
$ ln -s /lib/systemd/system/serial-getty\@.service /etc/systemd/system/getty.target.wants/serial-getty@ttyAMA0.service
```
## 安装一些依赖

```sh
# 安装音频管理
$ apt-get install python3
# 安装 vim 和 ssh
$ apt-get install vim ssh
# 安装网络管理工具
$ apt-get install ifupdown net-tools
# 安装其他依赖，如果还有其他依赖需要安装可以继续向后加入
$ apt-get install udev sudo wget curl
```
## 设置主机名和以太网

```sh
$ echo $HOST > /etc/hostname
$ echo "127.0.0.1    localhost.localdomain localhost" > /etc/hosts
$ echo "127.0.0.1    $HOST" >> /etc/hosts
$ echo "auto eth0" > /etc/network/interfaces.d/eth0
$ echo "iface eth0 inet dhcp" >> /etc/network/interfaces.d/eth0
以上全部完成后，我们的根文件系统就制作好了，退出文件系统后调用 ch-mount.sh 脚本取消挂载就好了。
```

```sh
$ exit
$ ./ch-mount -u linux-rootfs
文件系统制作好了以后，关于用法，现在的我只知道可以打包进.img镜像文件中，其他用法请自行查阅吧。
异常
在使用 apt 命令安装工具时，一直有警告信息：perl: warning: Setting locale failed.
```

## 界面方式解决

```sh
$ apt-get install locales
$ dpkg-reconfigure locales
# 进入界面后，向下选上 en_US.UTF-8 和 zh_CN.UTF-8 选择ok
# 进入下个界面后选择默认的zh_CN.UTF-8 就可以解决了。
# 用命令测试是否不报错
$ perl -e exit
```
## 手动解决

这里为什么时vim.tiny呢，是因为这个命令在文件系统中自带的，如果安装了vim的话，用vim也可以
```sh
$ apt-get install locales
# 打开文件后找到 en_US.UTF-8 和 zh_CN.UTF-8 删掉 # 好  取消注释
$ vim.tiny /etc/locale.gen
$ locale-gen
# 创建 /etc/locale.conf
$ vim.tiny /etc/locale.conf
LANG=zh_CN.UTF-8
# 用命令测试是否不报错
$ perl -e exit

```