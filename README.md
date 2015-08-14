
![](http://i.v2ex.co/27qp456L.png)

## 前言
提到数据同步就必然会谈到`rsync`，一般简单的服务器数据传输会使用`ftp/sftp`等方式，但是这样的方式效率不高，不支持差异化增量同步也不支持实时传输。针对数据实时同步需求大多数人会选择`rsync+inotify-tools`的解决方案，但是这样的方案也存在一些缺陷（文章中会具体指出），`sersync`是国人基于前两者开发的工具，不仅保留了优点同时还强化了实时监控，文件过滤，简化配置等功能，帮助用户提高运行效率，节省时间和网络资源。

>可靠高效的数据实时同步方式

---

## 更新历史

2015年08月14日 - 更新GitHub源码安装包
2015年08月13日 - 初稿

阅读原文 - http://wsgzao.github.io/post/sersync/

扩展阅读

基于rsync+sersync的服务器文件同步实战 - http://www.markdream.com/technologies/server/syncfile-by-rsync.shtml
通过 rsync sersync 实现高效的数据实时同步架构 - https://www.cnhzz.com/rsync_sersync/
rsync - https://rsync.samba.org/
inotify-tools - https://github.com/rvoicilas/inotify-tools
sersync - http://code.google.com/p/sersync/

---


## 原理

>Synchronize files and folders between servers -using inotiy and rsync with c++ 服务器实时同步文件，服务器镜像解决方案

`sersync`主要用于服务器同步，web镜像等功能。基于boost1.43.0,inotify api,rsync command.开发。目前使用的比较多的同步解决方案是`inotify-tools+rsync` ，另外一个是google开源项目`Openduckbill`（依赖于inotify- tools），这两个都是基于脚本语言编写的。相比较上面两个项目，本项目优点是：
1. sersync是使用c++编写，而且对linux系统文件系统产生的临时文件和重复的文件操作进行过滤（详细见附录，这个过滤脚本程序没有实现），所以在结合rsync同步的时候，节省了运行时耗和网络资源。因此更快。
2. 相比较上面两个项目，sersync配置起来很简单，其中bin目录下已经有基本上静态编译的2进制文件，配合bin目录下的xml配置文件直接使用即可。
3. 另外本项目相比较其他脚本开源项目，使用多线程进行同步，尤其在同步较大文件时，能够保证多个服务器实时保持同步状态。
4. 本项目有出错处理机制，通过失败队列对出错的文件重新同步，如果仍旧失败，则按设定时长对同步失败的文件重新同步。
5. 本项目自带crontab功能，只需在xml配置文件中开启，即可按您的要求，隔一段时间整体同步一次。无需再额外配置crontab功能。
6. 本项目socket与http插件扩展，满足您二次开发的需要。

>针对上图的设计架构，这里做几点说明，来帮助大家阅读和理解该图

1 ) `线程组线程`是等待线程队列的守护线程，当事件队列中有事件产生的时候，线程组守护线程就会逐个唤醒同步线程。当队列中 Inotify 事件较多的时候，同步线程就会被全部唤醒一起工作。这样设计的目的是为了能够同时处理多个 Inotify 事件，从而提升服务器的并发同步能力。同步线程的最佳数量=核数 x 2 + 2。
2 ) 那么之所以称之为`线程组线程`，是因为每个线程在工作的时候，会根据服务器上新写入文件的数量去建立子线程，子线程可以保证所有的文件与各个服务器同时同步。当要同步的文件较大的时候，这样的设计可以保证每个远程服务器都可以同时获得需要同步的文件。
3 ) 服务线程的作用有三个：
- 处理同步失败的文件，将这些文件再次同步，对于再次同步失败的文件会生成 rsync_fail_log.sh 脚本，记录失败的事件。
- 每隔10个小时执行 rsync_fail_log.sh 脚本一次，同时清空脚本。
- crontab功能，可以每隔一定时间，将所有路径整体同步一次。

4 ) `过滤队列`的建立是为了过滤短时间内产生的重复的inotify信息，例如在删除文件夹的时候，inotify就会同时产生删除文件夹里的文件与删除文件夹的事件，通过过滤队列，当删除文件夹事件产生的时候，会将之前加入队列的删除文件的事件全部过滤掉，这样只产生一条删除文件夹的事件，从而减轻了同步的负担。同时对于修改文件的操作的时候，会产生临时文件的重复操作。


## 角色

>注意主从配置的区别，记得调整SELinux和防火墙

iptables配置实践 - http://wsgzao.github.io/post/iptables/
LTMP手动编译安装以及全自动化部署实践 - http://wsgzao.github.io/post/ltmp/

1. 服务器A（主服务器）
2. 服务器B（从服务器/备份服务器）
3. rsync默认TCP端口为873

### 服务器B

``` bash
#在服务器B上安装rsync
cd /app/local
wget  http://rsync.samba.org/ftp/rsync/src/rsync-3.1.1.tar.gz
tar zxf rsync-3.1.1.tar.gz
cd rsync-3.1.1
./configure
make && make install


#设置rsync的配置文件
vi /etc/rsyncd.conf

#服务器B上的rsyncd.conf文件内容
uid=root
gid=root
#最大连接数
max connections=36000
#默认为true，修改为no，增加对目录文件软连接的备份 
use chroot=no
#定义日志存放位置
log file=/var/log/rsyncd.log
#忽略无关错误
ignore errors = yes
#设置rsync服务端文件为读写权限
read only = no 
#认证的用户名与系统帐户无关在认证文件做配置，如果没有这行则表明是匿名
auth users = rsync
#密码认证文件，格式(虚拟用户名:密码）
secrets file = /etc/rsync.pass
#这里是认证的模块名，在client端需要指定，可以设置多个模块和路径
[rsync]
#自定义注释
comment  = rsync
#同步到B服务器的文件存放的路径
path=/app/data/site/
[img]
comment  = img
path=/app/data/site/img

#创建rsync认证文件  可以设置多个，每行一个用户名:密码，注意中间以“:”分割
echo "rsync:rsync" > /etc/rsync.pass

#设置文件所有者读取、写入权限
chmod 600 /etc/rsyncd.conf  
chmod 600 /etc/rsync.pass  

#启动服务器B上的rsync服务
#rsync --daemon -v
rsync --daemon

#监听端口873
netstat -an | grep 873
lsof -i tcp:873

COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
rsync   31445 root    4u  IPv4 443872      0t0  TCP *:rsync (LISTEN)
rsync   31445 root    5u  IPv6 443873      0t0  TCP *:rsync (LISTEN)


#设置rsync为服务启动项（可选）
echo "/usr/local/bin/rsync --daemon" >> /etc/rc.local

#要 Kill rsync 进程，不要用 kill -HUP {PID} 的方式重启进程，以下3种方式任选
#ps -ef|grep rsync|grep -v grep|awk '{print $2}'|xargs kill -9
#cat /var/run/rsyncd.pid | xargs kill -9
pkill rsync
#再次启动
/usr/local/bin/rsync --daemon


```

### 服务器A

``` bash
#安装rsync
cd /app/local
wget  http://rsync.samba.org/ftp/rsync/src/rsync-3.1.1.tar.gz
tar zxf rsync-3.1.1.tar.gz
cd rsync-3.1.1
./configure
make && make install

#安装inotify-tools
cd /app/local
wget http://github.com/downloads/rvoicilas/inotify-tools/inotify-tools-3.14.tar.gz
tar zxf inotify-tools-3.14.tar.gz
cd inotify-tools-3.14
./configure --prefix=/app/local/inotify 
make && make install

#安装sersync
cd /app/local
wget https://sersync.googlecode.com/files/sersync2.5.4_64bit_binary_stable_final.tar.gz
tar zxf sersync2.5.4_64bit_binary_stable_final.tar.gz
mv /app/local/GNU-Linux-x86/ /app/local/sersync
cd /app/local/sersync
#配置下密码文件，因为这个密码是要访问服务器B需要的密码和上面服务器B的密码必须一致
echo "rsync" > /app/local/sersync/user.pass
#修改权限
chmod 600 /app/local/sersync/user.pass
#修改confxml.conf
vi /app/local/sersync/confxml.xml

```

``` xml

<?xml version="1.0" encoding="ISO-8859-1"?>
<head version="2.5">
 <host hostip="localhost" port="8008"></host>
 <debug start="true"/>
 <fileSystem xfs="false"/>
 <filter start="false">
 <exclude expression="(.*)\.php"></exclude>
 <exclude expression="^data/*"></exclude>
 </filter>
 <inotify>
 <delete start="true"/>
 <createFolder start="true"/>
 <createFile start="false"/>
 <closeWrite start="true"/>
 <moveFrom start="true"/>
 <moveTo start="true"/>
 <attrib start="false"/>
 <modify start="false"/>
 </inotify>
 
 <sersync>
 <localpath watch="/home/"> <!-- 这里填写服务器A要同步的文件夹路径-->
 <remote ip="8.8.8.8" name="rsync"/> <!-- 这里填写服务器B的IP地址和模块名-->
 <!--<remote ip="192.168.28.39" name="tongbu"/>-->
 <!--<remote ip="192.168.28.40" name="tongbu"/>-->
 </localpath>
 <rsync>
 <commonParams params="-artuz"/>
 <auth start="true" users="rsync" passwordfile="/app/local/sersync/user.pass"/> <!-- rsync+密码文件 这里填写服务器B的认证信息-->
 <userDefinedPort start="false" port="874"/><!-- port=874 -->
 <timeout start="false" time="100"/><!-- timeout=100 -->
 <ssh start="false"/>
 </rsync>
 <failLog path="/tmp/rsync_fail_log.sh" timeToExecute="60"/><!--default every 60mins execute once--><!-- 修改失败日志记录（可选）-->
 <crontab start="false" schedule="600"><!--600mins-->
 <crontabfilter start="false">
 <exclude expression="*.php"></exclude>
 <exclude expression="info/*"></exclude>
 </crontabfilter>
 </crontab>
 <plugin start="false" name="command"/>
 </sersync>
 
 <!-- 下面这些有关于插件你可以忽略了 -->
 <plugin name="command">
 <param prefix="/bin/sh" suffix="" ignoreError="true"/> <!--prefix /opt/tongbu/mmm.sh suffix-->
 <filter start="false">
 <include expression="(.*)\.php"/>
 <include expression="(.*)\.sh"/>
 </filter>
 </plugin>
 
 <plugin name="socket">
 <localpath watch="/home/demo">
 <deshost ip="210.36.158.xxx" port="8009"/>
 </localpath>
 </plugin>
 <plugin name="refreshCDN">
 <localpath watch="/data0/htdocs/cdn.markdream.com/site/">
 <cdninfo domainname="cdn.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>
 <sendurl base="http://cdn.markdream.com/cms"/>
 <regexurl regex="false" match="cdn.markdream.com/site([/a-zA-Z0-9]*).cdn.markdream.com/images"/>
 </localpath>
 </plugin>
</head>

```

``` bash
#运行sersync
nohup /app/local/sersync/sersync2 -r -d -o /app/local/sersync/confxml.xml >/app/local/sersync/rsync.log 2>&1 &
nohup /app/local/sersync/sersync2 -r -d -o /app/local/sersync/img.xml >/app/local/sersync/img.log 2>&1 &

-d:启用守护进程模式
-r:在监控前，将监控目录与远程主机用rsync命令推送一遍
-n: 指定开启守护线程的数量，默认为10个
-o:指定配置文件，默认使用confxml.xml文件

```

## GitHub源码仓库

``` bash
file://E:\sersync   (0 folders, 3 files, 1.88 MB, 1.88 MB in total.)
    inotify-tools-3.14.tar.gz 350.36 KB
    rsync-3.1.1.tar.gz    869.26 KB
    sersync2.5.4_64bit_binary_stable_final.tar.gz 710.24 KB
```

sersync - https://github.com/wsgzao/sersync

