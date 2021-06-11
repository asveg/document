# 定制centos7安装镜像



## 安装依赖工具

```
yum -y install anaconda createrepo mkisofs rsync syslinux
```

## 挂载光盘镜像

准备完整的CentOS-7安装镜像，挂载到虚拟机，这里使用CentOS-7-x86_64-Minimal-1908.iso作为基础镜像，选择Minimal是因为镜像比较小。

```sh
#创建挂载目录
mkdir /media/cdrom

#挂载光盘到/mnt/cdrom目录
mount /dev/cdrom /media/cdrom

# 进行入镜像挂载的目录并查看里面文件
cd /media/cdrom && tree -L 1
.
├── CentOS_BuildTag  # 系统版本构建标签 20200420-1800
├── EFI      # UEFI 启动模式下必须文件，Legacy模式下是非必须文件
├── EULA     # 最终用户许可协议
├── GPL      # 通用公用许可证／执照(General Public License)
├── images   # 启动映像文件
├── isolinux # 存放光盘启动时的安装界面信息
├── LiveOS   # 存储了映像文件
├── Packages # 系统自带rpm包软件
├── repodata # 系统rpm包metadate源数据
├── RPM-GPG-KEY-CentOS-7 # rpm的GPG校验公钥
├── RPM-GPG-KEY-CentOS-Testing-7 # 同上
└── TRANS.TBL # 提供比ISO9660标准约定的基本文件名更加灵活的文件名, 用简约符号代表目录、文件、链接;
├── .discinfo    #文件是安装价质的识别信息
├── .treeinfo   #文件是系统版本，创建时间及文件目录树结构信息
ks.cfg     #文件是无人值守自动化安装配置文件

#创建工作目录
mkdir -p /home/centos7

#复制光盘文件到目录centos7
rsync -a  /media/cdrom /home/centos7	
```

## **下载需要的rpm包**
把准备好的rpm包拷贝到/home/centos7/Packages/
```ssh
yum install --downloadonly --downloaddir=/root/package wget
cp /root/package/* /home/centos7/Packages	
```

## 编写ks.cfg文件

```sh
cat  << 'eof' > ks.cfg  
#version=DEVEL
# System authorization information
# Install OS instead of upgrade
#install
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
cdrom
# Use graphical install
graphical
# Run the Setup Agent on first boot
firstboot --enable
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

#harddrive --partition=/dev/sdb4 --dir=/

# Network information
network  --bootproto=static --device=ens192 --gateway=192.168.10.1 --ip=192.168.10.137 --nameserver=114.114.114.114 --netmask=255.255.255.0 --ipv6=auto --activate
# network  --bootproto=dhcp --device=enp2s0 --onboot=off --ipv6=auto
# network  --bootproto=dhcp --device=enp3s0 --onboot=off --ipv6=auto
# network  --bootproto=dhcp --device=enp4s0 --onboot=off --ipv6=auto
# network  --bootproto=dhcp --device=enp5s0 --onboot=off --ipv6=auto
# network  --bootproto=dhcp --device=enp6s0 --onboot=off --ipv6=auto
network  --hostname=localhost.localdomain
# Root password default is  root
rootpw --iscrypted $6$CSoABsIMzDwI9ECr$5MpYyJuYbl460JpA4wr6LfvEvAm.BFOjxWpjz0zyf.C.CH7Z2fPLyl79hDLe7Lleqwp8O9dJYtQ4.NPOle3nJ1
# stop firewall
firewall --disabled
# System services
services --enabled=ntpd,mysqld,sshd,wsa
# System timezone
timezone Asia/Shanghai --isUtc --nontp

# SELinux configuration
selinux --disabled

# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda

# Zero the MBR
zerombr

# Clear out partitions for sda
clearpart --all --initlabel --drives=sda
# part /boot --fstype="xfs" --ondisk=sda --size=500
# part / --fstype="xfs" --ondisk=sda --size=1 --grow
# part swap  --fstype="swap" --size=1024
part /boot --fstype="xfs" --ondisk=sda --size=400
part pv.364 --fstype="lvmpv" --ondisk=sda --size=1 --grow
volgroup centos --pesize=4096 pv.364
logvol swap  --fstype="swap" --size=1024 --name=swap --vgname=centos
logvol /  --fstype="xfs" --size=1 --grow --name=root --vgname=centos

# Reboot after installation
reboot

# install packages
%packages
@^minimal
@core
kexec-tools
mysql
mysql-community-server
tcpdump
jdk
ntp
net-tools
perl
unzip
zip
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%post --nochroot --log=/mnt/sysimage/var/log/kickstart_post_nochroot.log
# create usb mount director
mkdir -p /mnt/source
# 把usb 挂载到 /mnt/sysimage/usbdisk
mount /dev/cdrom  /mnt/source
# 把/mnt/sysimage/usbdisk/sources 数据传到 /mnt/sysimage/usr/local
cp -rf /mnt/source/sources /mnt/sysimage/usr/local
%end

%post
cd /usr/local/sources
bash /usr/local/sources/install.sh
%end
eof
```

**Ks.cfg文件参数说明:**

**install**（可选的）默认安装方式。必须指定安装类型`cdrom`，`harddrive`，`nfs`，`liveimg`，或`url`（对于FTP，HTTP或HTTPS安装）。`install`命令和安装方法命令必须在单独的行。

* `cdrom` - 从系统上的第一个光驱安装。

* `harddrive`- 从 Red Hat 安装树或本地驱动器上的完整安装 ISO 映像安装。该驱动器必须包含一个文件系统，安装程序可以安装：`ext2`，`ext3`，`ext4`，`vfat`，或`xfs`。

  - `--biospart=`- 要安装的 BIOS 分区（例如`82`）。
  - `--partition=`- 要安装的分区（例如`sdb2`）。
  - `--dir=`- 包含`*variant*`安装目录或完整安装 DVD 的 ISO 映像的目录。

  例如：

  ```
  harddrive --partition=hdb2 --dir=/tmp/install-tree
  ```

**clearpart**（可选的）在创建新分区之前从系统中删除分区。默认情况下，不删除任何分区。

* --all  从系统中删除所有分区
* --drives= 指定要从中清除分区的驱动器。例如，以下内容会清除主 IDE 控制器上前两个驱动器上的所有分区：clearpart --drives=hda,hdb --all
* -initlabel 通过为各自架构中指定用于格式化的所有磁盘（例如 x86 的 msdos）创建默认磁盘标签来初始化一个（或多个）磁盘。

可以通过如下命令来检查ks.cfg文件是否正确，如果没有任何输出则表示文件没有语法错误。

````sh
# ks.cfg文件放到isolinux目录下
ksvalidator ks.cfg
````

**ks.cfg文件组成大致分为3段**

- 命令段 

> 键盘类型，语言，安装方式等系统的配置，有必选项和可选项，如果缺少某项必选项，安装时会中断并提示用户选择此项的选项

- 软件包段，以%packages开头，换行开始写，有以下几种安装方式：

> @groupname：指定安装的包组
>
> package_name：指定安装的包
>
> -package_name：指定不安装的包

在安装过程中默认安装的软件包，安装软件时会自动分析依赖关系。

- 脚本段(可选)

  **%pre: 表示系统安装前，此时ISO镜像文件被挂载到内存中Linux的/mnt/source**

  **%post : 安装系统后执行的命令或脚本(基本支持所有命令)**

  * --不带参数，其实就是在真实的操作系统里操作。

  * --nochroot 允许您指定要在 chroot 环境之外运行的命令, 已安装的真实操作系统被挂载到内存虚拟操作系统中的/mnt/sysimage目录。这个参数的用途主要是配合%pre使用的。先将光盘里的文件copy到内存运行的虚拟操作系统，再从内存虚拟操作系统copy到已安装的真实操作操作。

实例: 实现了从ISO镜像中拷贝文本文件到安装好的真实操作系统中

```sh
# 该阶段把所需要的安装包及文件拷贝到操作系统中
%post --nochroot --log=/mnt/sysimage/root/ks-post.log
# 创建光盘挂载目录/mnt/source
mkdir -p /mnt/source
# 将iso光盘挂载于/mnt/source目录下复制文件到/mnt/sysimage虚拟操作系统目录下
mount -o loop /dev/cdrom /mnt/source
# 复制安装光盘/mnt/source文件到系统/usr目录下,在第二阶段执行
cp /mnt/source/software/netgainagent_v3.tar.gz /mnt/sysimage/usr/
cp /mnt/source/software/openssh-7.5p1.tar.gz /mnt/sysimage/usr/local
cp /mnt/source/software/openssl-1.0.1t.tar.gz /mnt/sysimage/usr/local

cp /mnt/source/software/cn_node_yum.repo /mnt/sysimage/etc/yum.repos.d/cn_node_yum.repo_bak
cp /mnt/source/software/sdns_internel_custom_yum.repo /mnt/sysimage/etc/yum.repos.d/sdns_internel_custom_yum.repo_bak
cp /mnt/source/software/test_custom_yum.repo /mnt/sysimage/etc/yum.repos.d/test_custom_yum.repo_bak
cp /mnt/source/software/service_custom_yum.repo /mnt/sysimage/etc/yum.repos.d/
# 复制完成后,取消挂载
umount -f /mnt/source
%end

#在操作系统中执行具体的安装部署操作
%post --log=/root/postinstall_stage2.log
cd /usr
tar zxvf openssh_5.0.tar.gz
cd /usr/zlib-1.2.3
./configure;make;make install
mv /etc/ssh /etc/ssh.bak        
cd /usr/openssh-5.0p1
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-pam --with-zlib --with-ssl-dir=/usr/local/ssl --with-md5-passwords --mandir=/
usr/share/man;make;make install
echo "==> update openssh finished.\n" > /root/postinstall_stage2.log
#agent
cd /usr
tar zxvf netgainagent_v3.tar.gz
echo "==>Uncompress netgainagent ok!\n" >> /root/postinstall_stage2.log

#configure the remote log server
mv /usr/openssh_5.0.tar.gz /root
mv /usr/netgainagent_v4.tar.gz /root
mv /usr/netgainagent_v3.tar.gz /root
rm -fr /usr/openssh-5.0p1
rm -fr /usr/zlib-1.2.3
echo "Files have been moved and deleted.\n" >> /root/postinstall_stage2.log
%end
```

我们都知道，在安装完centos7操作系统会在root下有anaconda-ks.cfg文件。其实，这个文件就是通过ks.cfg，根据用户安装选项自动生成的。

相关参数查阅, 请查看[Kickstart 安装](https://docs.centos.org/en-US/centos/install-guide/Kickstart2/)

## 编辑isolinux.cfg

isolinux目录下编辑isolinux.cfg文件生成如下配置：

```sh
cat << eof > isolinux.cfg
default vesamenu.c32
timeout 150

display boot.msg

# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu background splash.png
menu title CentOS 7
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none

# Selected item
menu color sel 0 #ffffffff #00000000 none

# Title bar
menu color title 0 #ff7ba3d0 #00000000 none

# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none

# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none

# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none

# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none

# Help text
menu color help 0 #ffffffff #00000000 none

# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none

# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install CentOS 7.7 (Kernel-5.5.7) 
  kernel vmlinuz
  append initrd=initrd.img ks=cdrom:/isolinux/ks.cfg  quiet

label check
  menu label Test this ^media & install CentOS 7
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet

menu separator # insert an empty line

# utilities submenu
menu begin ^Troubleshooting
  menu title Troubleshooting

label vesa
  menu indent count 5
  menu label Install CentOS 7 in ^basic graphics mode
  text help
        Try this option out if you're having trouble installing
        CentOS 7.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 xdriver=vesa nomodeset quiet

label rescue
  menu indent count 5
  menu label ^Rescue a CentOS system
  text help
        If the system will not boot, this lets you access files
        and edit config files to try to get it booting again.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rescue quiet

label memtest
  menu label Run a ^memory test
  text help
        If your system is having issues, a problem with your
        system's memory may be the cause. Use this utility to
        see if the memory is working correctly.
  endtext
  kernel memtest

menu separator # insert an empty line

label local
  menu label Boot from ^local drive
  localboot 0xffff

menu separator # insert an empty line
menu separator # insert an empty line

label returntomain
  menu label Return to ^main menu
  menu exit

menu end
eof
```

参数说明：

- timeout 150 表示倒计时15秒将从默认选项开始执行安装
- 这里的默认选项是 Install 7 auto
- menu default为默认选项的标识
- 新增一段，添加一个选项

```sh
label linux
  menu label ^Install CentOS 7.7 (Kernel-5.5.7) 
  menu default
  kernel vmlinuz
  append initrd=initrd.img ks=cdrom:/isolinux/ks.cfg 
```

其中指定了ks.cfg的路径ks=cdrom:/isolinux/ks.cfg

[引导介质](http://shouce.jb51.net/redhat-el-sag-en-4/s1-kickstart2-startinginstall.html)

## 自定义脚本

https://github.com/joyent/mi-centos-7/blob/master/ks.cfg

- ks.cfg中%post和%end段内定制安装完成系统后执行的命令

```sh
# the first one example
%post --nochroot --log=/mnt/sysimage/var/log/kickstart_post_nochroot.log
echo "Copying %pre stage log files"
/usr/bin/cp -rv /tmp/kickstart_pre.log /mnt/sysimage/var/log/
echo "=============================="
echo "Currently mounted partitions"
df -Th
%end

# the first two example
# --interpreter /usr/bin/python 允许您指定不同的脚本语言，例如 Python。将/usr/bin/python替换为您选择的脚本语言。
%post --interpreter=shell --log=/var/log/kickstart_post.log
echo "Executing post installation scripts"
/tmp/post_scripts.sh

echo "Installation Completed"
date
%end
```

其中的install.sh就是自定义业务的脚本

```sh
#!/bin/bash
echo "###########################################################"
echo "++++++++++++++Setup System++Deplay App ++++++++++++++++++"
echo "###########################################################"
  
# get ip address
interface=$(ls /sys/class/net| grep -v "lo" | head -1)
ipaddr=$(ip route show | grep -v default | awk '{print $9}')
echo "$interface = $ipaddr"  

#configure ssh
ssh=$(cat /etc/ssh/sshd_config | grep "^UseDNS no")
echo $ssh
if [ $? -eq 0 ]; then
	echo "the parameters already exists"
else
	sed -i '/#VersionAddendum none/a\UseDNS no' /etc/ssh/sshd_config
fi
sed -i "s/GSSAPIAuthentication yes/GSSAPIAuthentication no/g" /etc/ssh/sshd_config
systemctl restart sshd

#modify system resource limits
cat /etc/security/limits.conf | grep '40960' || {
  echo '* soft noproc 40960' >> /etc/security/limits.conf
  echo '* hard noproc 40960' >> /etc/security/limits.conf
  echo '* soft nofile 40960' >> /etc/security/limits.conf
  echo '* hard nofile 40960' >> /etc/security/limits.conf
}
source /etc/security/limits.conf
# install jdk tools
cat /etc/profile | grep 'JAVA_HOME' || {
  echo '#####jdk parameter########' >>  /etc/profile
  echo "JAVA_HOME=/usr/java/jdk1.8.0_271-amd64" >> /etc/profile
  echo "CLASSPATH=%JAVA_HOME%/lib:%JAVA_HOME%/jre/lib" >> /etc/profile
  echo "PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin" >> /etc/profile
  echo "export PATH CLASSPATH JAVA_HOME" >> /etc/profile
  echo "#export LANG=zh_CN.UTF-8" >> /etc/profile
}
source /etc/profile

# start service
systemctl enable ntpd
systemctl enable mysqld
```

## 更新comp.xml文件

comps.xml文件包含所有与RPM相关的内容。 它检查“软件包”下的RPM软件包依赖性。 安装时如果依赖包丢失，它将提示哪个RPM包需要哪个依赖。

```sh
rm -rf /home/centos7/repodata/*.gz *.bz2 repomd.xml
# cp /media/cdrom/repodata/*-comps.xml /home/centos7/repodata/comps.xml
# 切换到/home/centos7/路径下生成comps.xml文件
cd /home/centos7/ && createrepo -g repodata/*-comps.xml ./
```

**注：确认repodata目录是新生成的目录**

## 制作iso文件

```shell
cd /home/centos7/ 
genisoimage -joliet-long -V CentOS7 -o /root/ISO/CentOS-7.7.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -R -J -v -cache-inodes -T -eltorito-alt-boot -e images/efiboot.img -no-emul-boot /home/centos7
```

mkisofs参数说明：

* -o /root/ISO/CentOS-7.7.iso，设置输出文件名，-output
* -b isolinux/isolinux.bin，指定开机映像文件
* -c isolinux/boot.cat，制作启动光盘时，mkisofs会将开机映像文件中的全-eltorito-catalog*文件的全部内容作成一个文件
* -no-emul-boot，非模拟模式启动
* -boot-load-size 4，
* -boot-info-table，
* -joliet-long，
* -R，使用Rock Ridge Extensions，是用于linux环境下的光盘，文件名区分大小写同时记录文件长度，-rock
* -J，使用Joliet格式的目录与文件名称，Jolient是windows下使用的光盘，-joliet
* -v，执行时显示详细的信息，-verbose
* -V "CentOS 7"，设置卷标Volume ID，-volid
* -T，建立文件名的转换表，适用于不支持Rock Ridge Extensions的系统，-translation-table
* /home/centos7，光盘源文件目录

其它操作:

* 嵌入md5校验码 （该命令由isomd5sum提供）   
  `implantisomd5 /tmp/CentOS7-v1.0.iso`

* 检验md5    
  `checkisomd5 /tmp/CentOS7-v1.0.iso`
