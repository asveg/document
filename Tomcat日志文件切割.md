Tomcat日志文件切割
```
我们都知道将一个项目部署到Tomcat之后，Tomcat服务启动后的标准输出(stdout)和标准出错(stderr)都会默认重定向到${TOMCAT_HOME}/logs/catalina.out这个文件中，如果不及时清理将会占用服务器磁盘大量空间从而影响到整个项目的正常运行； 
这样大日志文件对于我们进行错误排查以及日志分析不是很方便。为了解决这个问题编写了脚本来切割日志，即Tomcat运行的每天都按照日期命名新建一个日志文件。
```
1. 创建shell脚本进行catalina.out日志文件切割
```
编写一个tomcat-logrotate.sh文件并赋予文件执行全向最后放入$TOMCAT_HOME/bin目录下面，然后结合linux系统自带的定时器进行Tomcat日志切割。Shell脚本如下：

#!/bin/bash

cd  `dirname $0`                           ##进入执行脚本所在目录，我这里是$TOMCAT_HOME/bin
d=`date +%Y%m%d`                           ##获取当前日期
d7=`date -d'7 day ago' +%Y%m%d`            ##获取7天前的日期

cd  ../logs/                               ##进入日志所在目录
cp catalina.out   catalina.out.${d}        ##将当前日志的内容拷贝到以日期分割的新文件中，
echo "" > catalina.out                     ##并清空当前日志文件的内容
rm -rf catalina.out.${d7}                  ##删除七天前的日志

注意：执行这个脚本的定时任务的频率以及时间都要控制好，不然会有部分日志内容保存不下来的情况。
```
2. 使用log4j成功使catalina.out文件实现分割
```
在Tomcat根目录下建立 /webapps/项目名/WEB-INF/classes/log4j.properties，内容如下：

############################################################################ 
log4j.rootLogger=INFO, R 
log4j.appender.R=org.apache.log4j.RollingFileAppender 
log4j.appender.R.File=${TOMCAT_HOME}/logs/tomcat.newlog    #设定日志文件名
log4j.appender.R.MaxFileSize=100KB                          #设定文件到100kb即分割
log4j.appender.R.MaxBackupIndex=10                          #设定日志文件保留的序号数
log4j.appender.R.layout=org.apache.log4j.PatternLayout 
log4j.appender.R.layout.ConversionPattern=%p %t %c - %m%n
############################################################################

注意：以上配置需要在Tomcat根目录下的 /webapps/项目名/WEB-INF/lib目录下加入log4j.jar和commons-logging.jar，然后重新启动Tomcat服务即可生效。
```
