---
title: 陈明园是傻屌
date: 2020-12-15 10:40:38
categories: study
tags: [mycat]
---

##分库分表方案选型

####方案背景：
> 当项目随公司业务以及时间的发展，数据量相应就会增加，当单库的并发达到2000以上，或者单表的数据量达到千万级别，那么这个时候分库（分表）就不可避免，这篇文章主要调研市面上针对分库分表的方案，为后期分库分表做铺垫。

#####何为数据切分：
简单来说，就是指通过某种特定的条件，将我们存放在同一个数据库中的数据分散存
放到多个数据库（主机）上面，以达到分散单台设备负载的效果。
 数据切分分为两种：
 ######垂直切分， 水平切分
 
 #####垂直切分优缺点：
 优点：
 - 数据库的拆分简单明了，拆分规则明确；
 - 应用程序模块清晰明确，整合容易；
 - 数据维护方便易行，容易定位；

缺点：
- 部分表关联无法在数据库级别完成，需要在程序中完成，存在跨库join的问题，对于这类的
表，就需要去做平衡，是数据库让步业务，共用一个数据源，还是分成多个库，业务之间通
过接口来做调用；在系统初期，数据量比较少，或者资源有限的情况下，会选择共用数据源
，但是当数据发展到了一定的规模，负载很大的情况，就需要必须去做分割；
- 对于访问极其频繁且数据量超大的表仍然存在性能瓶颈，不一定能满足要求；
- 事务处理相对更为复杂；
- 切分达到一定程度之后，扩展性会遇到限制；
- 过多切分可能会带来系统过渡复杂而难以维护。

#####水平拆分：
相对于垂直拆分，水平拆分不是将表做分类，而是按照某个字段的某种规则来分散到多个库之中
，每个表中包含一部分数据。简单来说，我们可以将数据的水平切分理解为是按照数据行的切分
，就是将表中的某些行切分到一个数据库，而另外的某些行又切分到其他的数据库

优点：
-  表关联基本能够在数据库端全部完成；
-  不会存在某些超大型数据量和高负载的表遇到瓶颈的问题；
-  应用程序端整体架构改动相对较少；
-  事务处理相对简单；
-  只要切分规则能够定义好，基本上较难遇到扩展性限制；

缺点：
-  切分规则相对更为复杂，很难抽象出一个能够满足整个数据库的切分规则；
-  后期数据的维护难度有所增加，人为手工定位数据更困难；
-  应用系统各模块耦合度较高，可能会对后面数据的迁移拆分造成一定的困难。
-  跨节点合并排序分页问题
-  多数据源管理问题


#####几种典型的分片规则：
- 按照tenantId 求模，将数据分散到不同的数据库，具有相同数据用户的数
据都被分散到一个库中。
- 按照日期，将不同月甚至日的数据分散到不同的库中。
- 按照某个特定的字段求摸，或者根据特定范围段分散到不同的库中

#####数据切分参考：
第一原则：能不切分尽量不要切分。
第二原则：如果要切分一定要选择合适的切分规则，提前规划好。
第三原则：数据切分尽量通过数据冗余或表分组（Table Group）来降低跨库Join的可能。
第四原则：由于数据库中间件对数据Join实现的优劣难以把握，而且实现高性能难度极
大，业务读取尽量少使用多表Join。
>综上描述，数据切分带来的核心问题主要有三个：
-  引入分布式事务的问题；
-  跨节点Join 的问题；
-  跨节点合并排序分页问题；




##数据库代理中间件选型：

###Mycat
- 是一个数据库代理
- MySQL、SQL Server、Oracle、DB2等主流数据库，也支持MongoDB这种新型NoSQL方式的存储
- Mycat并不存储数据，只做数据路由

Mycat的原理中最重要的一个动词是“拦截”，它拦截了用户发送过来的SQL语句，首先
对SQL语句做了一些特定的分析：如分片分析、路由分析、读写分离分析、缓存分析等，
然后将此SQL发往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户
![](https://www.showdoc.com.cn/server/api/attachment/visitfile/sign/84101da234dc4fe456286313909fa848?showdoc=.jpg)


####Mycat中间件配置：
- 1.逻辑库配置：
server.xml:
```xml
	<user name="user">
		<property name="password">******</property>
		<property name="schemas">avengers</property>
	</user>
```
schemas.xml:
```xml
	<schema name="avengers" checkSQLschema="true" dataNode="localdn1,localdn2">
		<!--mycat中的逻辑表-->
		<!--
		name:逻辑表的名称，名称必须唯一
		dataNode:值必须跟dataNode标签中的name对应，如果值过多可以用 dataNode="dn$0-99,cn$100-199"
		rule:分片规则配置，定义在rule.xml中，必须与tableRule中的name对应
		ruleRequired:该属性用于指定表是否绑定分片规则，如果配置为true，但没有配置具体rule的话 ，程序会报错
		primaryKey：该逻辑表对应真实表的主键，例如：分片的规则是使用非主键进行分片的，那么在使用主键查询的时候，就会发送查询语句到所有配置的DN上，如果使用该属性配置真实表的主键。难么MyCat会缓存主键与具体DN的信息，那么再次使用非主键进行查询的时候就不会进行广播式的查询，就会直接发送语句给具体的DN，但是尽管配置该属性，如果缓存并没有命中的话，还是会发送语句给具体的DN，来获得数据
		type：全局表：global  每一个dn都会保存一份全局表，普通表：不指定该值为globla的所有表
		autoIncrement:autoIncrement=“true”,默认是禁用的
		needAddLimit：默认是true
		-->
		<table name="t_order" dataNode="localdn" autoIncrement="true" subTables="t_order$1-3" primaryKey="id" rule="mod-long">
		</table>
<!--		<table name="mycat_sequence" dataNode="localdn" primaryKey="name"/>-->
	</schema>

	<!-- <dataNode name="dn1$0-743" dataHost="localhost1" database="db$0-743"
		/> -->
	<dataNode name="localdn1" dataHost="localhost1" database="consult" />
	<dataNode name="localdn2" dataHost="localhost2" database="consult" />
		<!--
	maxCon:指定每个读写实例连接池的最大连接。也就是说，标签内嵌套的writeHost、readHost标签都会使用这个属性的值来实例化出连接池的最大连接数
	minCon:指定每个读写实例连接池的最小连接，初始化连接池的大小
	balance:
 balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。
 balance="1"，全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
 balance="2"，所有读操作都随机的在writeHost、readhost上分发。
 balance="3"，所有读请求随机的分发到writeHost对应的readhost执行，writerHost不负担读压力，注意balance=3只在1.4及其以后版本有，1.3没有。
	writeType:
	负载均衡类型，目前的取值有3种：
 writeType="0", 所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后已切换后的为准，切换记录在配置文件中:dnindex.properties .
writeType="1"，所有写操作都随机的发送到配置的writeHost。
 writeType="2"，没实现
	switchType:
  - 表示不自动切换
    默认值，自动切换
    基于MySQL主从同步的状态决定是否切换
      心跳语句为 show slave status
    基于MySQL galary cluster的切换机制（适合集群）（1.4.1）
		-->
	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
			  writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"  slaveThreshold="100">
		<!--heartbeat标签
		MYSQL可以使用select user()
		Oracle可以使用select 1 from dual
		-->
		<heartbeat>select user()</heartbeat>
		<connectionInitSql></connectionInitSql>
		<!-- can have multi write hosts -->
		<!--
		如果writeHost指定的后端数据库宕机，那么这个writeHost绑定的所有readHost都将不可用。另一方面，由于这个writeHost宕机系统会自动的检测到，并切换到备用的writeHost上去
		-->
	<writeHost host="hostM1" url="jdbc:mysql://localhost:3306" user="root"
				password="123456">
	</writeHost>
	<dataHost name="localhost2" maxCon="1000" minCon="10" balance="0"
			  writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"  slaveThreshold="100">
		<!--heartbeat标签
		MYSQL可以使用select user()
		Oracle可以使用select 1 from dual
		-->
		<heartbeat>select user()</heartbeat>
		<connectionInitSql></connectionInitSql>
		<!-- can have multi write hosts -->
		<!--
		如果writeHost指定的后端数据库宕机，那么这个writeHost绑定的所有readHost都将不可用。另一方面，由于这个writeHost宕机系统会自动的检测到，并切换到备用的writeHost上去
		-->
	<writeHost host="hostM2" url="jdbc:mysql://localhost:3306" user="root"
				password="123456">
		<readHost host="hostM2" url="jdbc:mysql://localhost:3307" 			 password="123456" user="root"/>
	</writeHost>
	</dataHost>
```

### MyCat 分片规则

参考：http://www.mycat.io/document/Mycat_V1.6.0.pdf

####根据id取模分片规则：
rule.xml 
> PartitionByMod需要 分片id必须能够转换为数字
PartitionByHashMod  根据分片id hash,所以并不强制要求 column 的数据类型是数字，也可以是字符型

```xml
	<tableRule name="mod-long">
		<rule>
			<columns>tenant_id</columns>
			<algorithm>mod-long</algorithm>
		</rule>
	</tableRule>
	
	<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
		<!-- how many data nodes -->
		<property name="count">2</property>
	</function>
```

####固定hash分片：
rule.xml：
```xml
	<tableRule name="sharding-by-intfile">
		<rule>
			<columns>province</columns>
			<algorithm>hash-int</algorithm>
		</rule>
	</tableRule>
	
	<function name="hash-int"
			  class="io.mycat.route.function.PartitionByFileMap">
		<property name="mapFile">partition-hash-int.txt</property>
		<!--type默认值为0，0表示Integer，非零表示String-->
		<property name="type">1</property>
		<!--defaultNode 默认节点：小于0表示不设置默认节点，大于等于0表示设置默认节点,不能解析的枚举就存到默认节点-->
		<property name="defaultNode">0</property>
	</function>
```
partition-hash-int.txt：

> tenantId = 0

```html
234234234214231423=0
784656757853535643=1
345451423534645645=0
895784654345431245=1
563457651435234632=1
534534567587588567=1
```
####一致性hash算法分片
> 数据分布不均匀，
利用虚拟节点分布于hash环，通过hash算法将数据均匀分布在节点


```xml
	<tableRule name="sharding-by-murmur">
		<rule>
			<columns>orderId</columns>
			<algorithm>murmur</algorithm>
		</rule>
	</tableRule>

	<function name="murmur"
			  class="io.mycat.route.function.PartitionByMurmurHash">
		<property name="seed">0</property><!-- 默认是0 -->
		<property name="count">2</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片 -->
		<property name="virtualBucketTimes">160</property><!-- 一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍 -->
		<!-- <property name="weightMapFile">weightMapFile</property> 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写，以从0开始到count-1的整数值也就是节点索引为key，以节点权重值为值。所有权重值必须是正整数，否则以1代替 -->
		<property name="bucketMapPath">E:\idea\Mycat-Server-Mycat-server-1675-release\src\main\resources</property>
		<!-- 用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的murmur hash值与物理节点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 -->
	</function>
```

####调增一致性hash算法分片
> 数据分布均匀，效率高，消耗少，有分布式事务问题

```xml
	<tableRule name="jch">
		<rule>
			<columns>tenant_id</columns>
			<algorithm>jump-consistent-hash</algorithm>
		</rule>
	</tableRule>
	
		<function name="jump-consistent-hash" class="io.mycat.route.function.PartitionByJumpConsistentHash">
		<property name="totalBuckets">2</property>
	</function>
```

#### myCat 全局表
> 数据冗余

```xml
		<table name="sys_xxx" primaryKey="id" dataNode="dn140,dn141" type="global"/>
```

####MyCat 动态扩容

1.新增 newSchema.xml 和 newRule.xml 并配置成扩容后的节点
<img src="/image/2020-12-10_5fd1e973ebd30.png">

2.修改dataMigrate.sh文件
该文件类型是dos的，需要修改为unix类型
查看.sh文件类型：:set ff
修改.sh文件类型：:set ff=unix 回车

3.修改dataMigrate.sh中的配置
基本上修改一个即可：
> mysql bin路径
RUN_CMD="$RUN_CMD -mysqlBin=/usr/bin"
这个是mysqldump文件的路径，
找该文件：find / -name mysqldump

4.修改/conf/下migrateTables.properties配置文件，指定需要迁移的逻辑库和表

5. 执行 ./dataMigrate.sh

数据迁移后将newSchema.xml和 newRule.xml 替换schema.xml,rule.xml(整个过程可以在mycat 未开启的状态下迁移)

注意事项
1、jdk不能用openJdk，需要自己安装，配置环境变量
2、之前的老的schema.xml和rule.xml老配置不能动
3、新增迁移后的节点新配置newSchema.xml和newRule.xml
4、mycat bug migrateTables.properties中逻辑库的名称不能既有大些又有小写。
5、迁移类：DataMigrator



