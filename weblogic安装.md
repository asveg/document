weblogic安装

一、环境
```
linux版本：RedHat 7.2
weblogic版本:WLS10.3.6  下载链接：https://pan.baidu.com/s/1nSTHA69lLVo55Uk5y_UcAw
JDK版本：1.8   下载链接：https://pan.baidu.com/s/1_p0KN_45cRstmrZc0kHO5w
```
二、安装JDK
```
将JDK包上传至根目录即可，
[root@bogon /]# tar -xvf  jdk-8u161-linux-x64.tar.gz
[root@bogon /]# mv jdk1.8.0_161/ /usr/java
[root@bogon /]# vim /etc/profile
export JAVA_HOME=/usr/java
export JRE_HOME=/usr/java/jre
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
export ORACLE_HOME=/weblogic/wls1212/ofmhome
[root@bogon /]# source /etc/profile
```
三、创建目录，用户，组
```
mkdir -pv /data/weblogic
groupadd weblogic
useradd -g weblogic weblogic
echo "123456" | passwd --stdin weblogic   
chown -R weblogic:weblogic /data
```
四、安装
```
将安装包wls1036_generic.jar上传至/home/weblogic下

su - weblogic
java -jar wls1036_generic.jar

执行上述命令后，进入命令行安装模式，出现如下提示：

---------oracle installer - weblogic 10.3.6.0---------

我们需要注意的是(除以下步骤需要选择，其他步骤均回车即可):

1. 在选择中间件主目录填写 /data/weblogic/
2. 在配置注册安全更新时，
	(1)输入要选择的索引号 3
	(2)输入No
	(3)输入Yes  #不接受更新
3.在选择安装类型时，选择定制安装 ，即选择索引号2

安装主目录是/data/weblogic
Weblogic server目录是 /data/weblogic/wlserver_10.3
Oracel coherence目录是/data/weblogic/coherence_3.7
```
五、配置weblogic server环境
```
5.1 配置域（它是一个管理的单元，其中包含了多个weblogic server）

使用配置助手配置，以下为各个操作系统上的不同方法汇总：

Scripts in <WEBLOGIC_HOME>/common/bin directory

Graphical mode:

   Windows start menu

 [windows]config.cmd

 [unix|linux]config.sh

Console mode:

   [windows]config.cmd -mode=console

   [unix|linux]config.sh -mode=console


本例中依然采用console模式配置域。
su - weblogic
cd /data/weblogic/wlserver_10.3/common/bin
./config.sh -mode=console

执行上述命令后，出现如下界面，按步骤配置即可：

---------Fusion Middleware配置向导---------

第一步：选择索引号1 创建新的weblogic域
第二部：选择索引号1 选择weblogic platform组件
第三步：按回车
第四步：输入域名  根据自己需要命名
第五步：按回车
第六步：按回车
第七步：选择对应的索引号 输入密码并确定密码
第八步：选择索引号 选择生产模式或开发模式，根据自己情况定
第九步：选择索引号1 选择自己安装jdk环境
第十步：选择索引号1 
第十一步：按回车
第十二步：选择索引号 修改监听的地址和端口，回车即可
第十三步：显示创建domain的进度条。
至此，weblogic域安装成功，安装主目录是/data/weblogic/user_projects/domains
```
六、打java反序列化补丁  weblogic用户
```
./bsu.sh -prod_dir=/data/weblogic/Oracle/Middleware/wlserver_10.3 -status=applied -verbose -view   先进行检查 会生成一个cache_dir目录

先打p20780171_1036_Generic

把文件传到cache_dir

./bsu.sh -install -patch_download_dir=/data/weblogic/Oracle/Middleware/utils/bsu/cache_dir -patchlist=EJUW -prod_dir=/data/weblogic/Oracle/Middleware/wlserver_10.3

./bsu.sh -install -patch_download_dir=/data/weblogic/Oracle/Middleware/utils/bsu/cache_dir -patchlist=ZLNA -prod_dir=/data/weblogic/Oracle/Middleware/wlserver_10.3

```
七、修改hosts文件
```
切换到root用户：
echo IP   hostname >> /etc/hosts
```
八、启动weblogic server
```
su - weblogic
cd  /data/weblogic/user_projects/domains/base_domain/bin
./startWeblogic.sh

输入用户名：weblogic
输入密码：12345678
```
九、打开控制台
```
http://IP:7001/console

