---
title: mycat读写分离
date: 2020-12-15 10:40:38
categories: study
tags: [mycat]
---

## 读写分离方案选型

### Replication常用架构
#### 常规复制架构(Master --- Slaves)
<img src="/image/mycat读写分离1.png" width="80%">

缺点：
- master不能停机，停机就不能接收写请求
- slave过多会出现延迟

由于master需要进行常规维护停机了，那么必须要把一个slave提成master，选哪一个是一个问题？
某一个slave提成master了，就存在当前master和之前的master数据不一致的情况，并且之前master并没有保存当前master节点的binlog文件和pos位置。

#### Dual Master 复制架构(Master --- Master) Master)
<img src="/image/mycat读写分离2.png" width="80%">

可以配合一个第三方的工具，比如keepalived轻松做到IP的漂移，停机维护也不会影响写操作。

#### 级联复制架构(Master --- Slaves --- Slaves ...) ...)
<img src="/image/mycat读写分离3.png" width="80%">

如果读压力加大，就需要更多的slave来解决，但是如果slave的复制全部从master复制，势必会加大master的复制IO的压力，所以就出现了级联复制，减轻master压力。

#### Dual Master 与级联复制结合架构(Master -Master - Slaves)
<img src="/image/mycat读写分离4.png" width="80%">
这样解决了单点master的问题，解决了slave级联延迟的问题.

#### Replication机制的实现原理
<img src="/image/mycat读写分离5.png" width="80%">


### mysql主从复制

#### 基于docker
1.创建端口为3307的mysql容器：
docker run -p 3307:3306  --restart=always  --privileged=true --name mysql -v /home/mysql/docker-data/3307/conf:/etc/mysql/conf.d -v /home/mysql/docker-data/3307/data/:/var/lib/mysql -v /home/mysql/docker-data/3307/logs/:/var/log/mysql -e MYSQL_ROOT_PASSWORD="123456" -d mysql:5.7

master my.cnf配置：
```xml
// master配置
server-id=1283307
log-bin=mysql-bin //开启复制功能
auto_increment_increment=2
auto_increment_offset=1
lower_case_table_names=1
binlog-do-db=mstest //要同步的mstest数据库,要同步多个数据库
binlog-ignore-db=mysql //要忽略的数据库
```


2.创建端口为3308的mysql容器：
docker run -p 3308:3306  --restart=always  --privileged=true --name mysql -v /home/mysql/docker-data/3308/conf:/etc/mysql/conf.d -v /home/mysql/docker-data/3308/data/:/var/lib/mysql -v /home/mysql/docker-data/3308/logs/:/var/log/mysql -e MYSQL_ROOT_PASSWORD="123456" -d mysql:5.7

slave my.cnf配置
```xml
// slave配置
server-id=1293308
log-bin=mysql-bin
auto-increment-increment=2
auto-increment-offset=2
lower_case_table_names=1
replicate-do-db = wang #需要同步的数据库
binlog-ignore-db = mysql
binlog-ignore-db = information_schema
```

##### 1.在master mysql添加权限
GRANT REPLICATION SLAVE,FILE,REPLICATION CLIENT ON *.* TO 'repluser'@'%' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
##### 2.在master上查看master的二进制日志
<img src="/image/mycat12.png" width="80%">
##### 3 在slave中设置master的信息
change master to master_host='192.168.231.128',
master_port=3307,master_user='repluser',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=154;
##### 4 开启slave，启动SQL和IO线程
start slave;
##### 效果：
master:新增数据
<img src="/image/mycat读写分离6.png" width="80%">
slave:同步数据
<img src="/image/mycat读写分离7.png" width="80%">



#### 主从半同步复制：
##### 加载lib，所有主从节点都要配置
主库：install plugin rpl_semi_sync_master soname 'semisync_master.so';
从库：install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
可以一起装。建议一起装，因为会有主从切换的情景。


##### 启用半同步
先启用从库上的参数，最后启用主库的参数。
从库：set global rpl_semi_sync_slave_enabled = {0|1}; # 1：启用，0：禁止
主库：
set global rpl_semi_sync_master_enabled = {0|1}; # 1：启用，0：禁止

//设置msater等待slave应答时间（默认10秒），无应答会从半同步复制变成异步复制
set global rpl_semi_sync_master_timeout = 10000; # 单位为ms 

##### 查看状态：
<img src="/image/mycat读写分离8.png" width="80%">
wait_point: AFTER_SYNC :增强版半同步复制

### mysql高可用
<img src="/image/mycat读写分离9.png" width="80%">
##### 配置haproxy

- 新建目录和用户 
mkdir /etc/haproxy mkdir /var/lib/haproxy useradd -r haproxy
- 配置日志
vim /etc/rsyslog.conf
Provides TCP syslog reception #去掉下面两行注释，开启TCP监听
$ModLoad imudp
$UDPServerRun 514
local2.* /var/log/haproxy.log #添加日志

- vi /etc/sysconfig/rsyslog
SYSLOGD_OPTIONS=""
改为SYSLOGD_OPTIONS="-r -m 2 -c 2"
- 创建日志文件
touch /var/log/haproxy.log
- 启动日志
systemctl restart rsyslog.service
- 启动haproxy cd /usr/sbin/haproxy
./haproxy -f /etc/haproxy/haproxy.cfg

- 在两台数据库添加远程访问权限
GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
- 测试haproxy服务器能否连接到数据库服务器
- 安装mysql客户端
yum install -y mysql
mysql -uroot -p123456 -h 192.168.231.128
mysql -uroot -p123456 -h 192.168.231.129
- 在非haproxy服务器测试通过访问haproxy访问到mysql服务
mysql -uroot -p123456 -h 192.168.231.128 -P 3300

- 访问页面http://192.168.231.128:1080/stats
<img src="/image/mycat读写分离10.png" width="80%">

- keepalived源码安装
wget http://www.keepalived.org/software/keepalived-1.3.5.tar.gz
tar -zxvf keepalived-1.3.5.tar.gz
- 安装openssl openssl-devel
yum -y install openssl openssl-devel
./configure --prefix=/usr/local/keepalived --sbindir=/usr/sbin/ --sysconfdir=/etc/ --
mandir=/usr/local/share/man/
make && make install

- 修改配置
vi /etc/keepalived/keepalived.conf
- 给执行权限 # chmod +x /etc/keepalived/chk.sh
- keepalived yum安装
- 预先安装好epel-release源
yum list installed|grep epel-release
- 查找可用安装的keepalived源
yum search keepalived
- 命令进行安装
yum install keepalived -y
- 启动keepalived服务
systemctl start keepalived
- 使用yum安装的会有一个默认配置文件模板
- 路径为/etc/keepalived/keepalived.conf
yum install ipvsadm -y

mysql -h 192.168.67.140 -u root -p123456 -P 3307 -e "show status;" >/dev/null 2>&1
> /dev/null 2>&1 输出”黑洞”
> $? 上一个指令是否执行成功
> 0 成功1 失

### MyCat配置：
#### 结构图如下：
<img src="/image/mycat读写分离11.png" width="80%">
##### 将MyCat配置到环境变量中
 vi /etc/profile
增加如下内容　　　　　　　　　
MYCAT_HOME=/usr/local/mycat
PATH=$MYCAT_HOME/bin:$PATH

配置schema.xml：

```
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!-- 定义虚拟数据库名称 -->
    <schema name="mycat_mdr" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1" >
        <!-- 这里配置分库分表，因只做读写分离所以这里暂不配置 -->
    </schema>

    <dataNode name="dn1" dataHost="192.168.231.128" database="marvel_test" /> 
    <!-- 上述这里三个参数分别是定义dataNode的别名、数据库的IP或局域网服务器的别名在hosts中配置、数据库名 -->
    <!-- 将balance设置为3表示开启读写分离 -->
    <dataHost name="localhost" maxCon="1000" minCon="10" balance="3"
              writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
        <!-- 定时执行SQL保持心跳 -->
        <heartbeat>select user()</heartbeat>
        <!-- 添加写入库配置 -->
        <writeHost host="hostM1" url="192.168.231.128:3307" user="root"
                   password="123456">
            <!-- 添加只读库配置-->
            <readHost host="hostS1" url="192.168.231.129:3308" user="root" password="123456" />
        </writeHost>

    </dataHost>
</mycat:schema>
```