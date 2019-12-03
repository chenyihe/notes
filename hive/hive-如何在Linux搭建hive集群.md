# hive-如何在Linux搭建hive集群



1、下载安装包。

​	下载地址：<https://mirrors.tuna.tsinghua.edu.cn/apache/hive/>

​	本人下载的是：apache-hive-2.3.6-bin.tar.gz

2、上传安装包至服务器，并解压至相应的目录。

​	解压命令：tar --zxvf apache-hive-2.3.6-bin.tar.gz -C /opt

3、添加hive环境变量

```sh
	vi /etc/profile
```

		export HIVE_HOME=/opt/hive
		export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HIVE/bin
4、安装配置mysql，并卸载centos自带的mysql

​	检查是否含有自带的mysql

​	rpm -qa | grep mysql

​	卸载mysql：rpm -e nodeps mysql

​	mysql下载地址：<https://downloads.mysql.com/archives/community/>

5、解压mysql安装包至/usr/bin/mysql目录，并安装mysql

​	1)    tar -zxvf  mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz -C /usr/local

​	2)    mkdir /opt/data      //新建数据仓库目录

​	3)    groupadd mysql   //添加mysql用户组

​	4）  useradd -r -s /sbin/nologin -g mysql mysql -d /usr/local/mysql     //添加mysql用户和登录地址并禁止shell登录

​	5）  chown -R mysql:mysql  /usr/bin/mysql

6、初始化mysql数据库，并记住生成的密码

​	bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql   
​	bin/mysql_ssl_rsa_setup  --datadir=/data/mysql
​	cd /usr/local/mysql/support-files	
​	cp my-default.cnf /etc/my.cnf
​	cp mysql.server /etc/init.d/mysql
​	vi /etc/init.d/mysql   //修改数据目录和安装目录
​	basedir=/usr/local/mysql
​	datadir=/opt/data/mysql

7、登录mysql

​	mysql -hlocalhost -uroot -p   //输入刚才生成的密码

​		--如果出现command not found 执行以下命令

​		`ln -s /usr/local/mysql/bin/mysql /usr/bin`

​	--设置root账户的host地址（修改了才可以远程连接）

​		mysql>grant all privileges on\*.* to 'root'@'%' identified by 'root';
​		mysql>flush privileges;

​	创建新账户  ：create user '用户名'@'%' identified by ' 密码'

​	数据库授权：grant all privileges on 数据库名.* to '用户名'@'%' identified by '密码';

8、解压hive压缩包，并将mysql-connector-java-5.1.39.jar放到hive/lib目录下。并修改配置文件

- cd /opt/hive/conf

- cp hive-env.sh.template hive-env.sh

- 编辑hive-env.sh     -- HADOOP_HOME=/opt/hadoop  export HIVE_CONF_DIR=/opt/hive/conf

- 创建配置文件中要用到的HDFS文件路径并赋予权限

  ​	`hadoop fs -mkdir -p /user/hive/warehouse`

  ​	`hadoop fs -mkdir -p /user/hive/logs`	

  ​	`hadoop fs -mkdir -p /user/hive/temp`

  ​	`hadoop fs -chmod 733 /user/hive/*`

- 配置hive-site.xml

  ```xml
  <property>
      <name>hive.exec.mode.local.auto</name>
      <!--value>false</value-->
      <value>true</value>
  	<!--改为true进行操作时优先使用本地模式（不使用MapReduce）-->
      <description>Let Hive determine whether to run in local mode automatically</description>
   </property>
  
  <property>
  	<name>javax.jdo.option.ConnectionURL</name> 
  	<!--value>jdbc:derby:;datavaseName=metastore_db;create=true</value-->
  	<value>jdbc:mysql://hadoop01:3306/hive?createDatabaseIfNotExist=true</value>
  	<!--替换为jdbc:mysql://sure（mysql所在的主机名）:3306/hive（随意创建的用户名）?createDatabaseIfNotExist=true-->
  </property>
  
  <property>
  	<name>javax.jdo.option.ConnectionDriverName</name>
  	<value>com.mysql.jdbc.Driver</value>
  </property>
  
  <property>
  	<name>javax.jdo.option.ConnectionUserName</name>
  	<value>hive</value>       
  	<!--//登录hive的用户-->
  </property>
  
  <property>
  	<name>javax.jdo.option.ConnectionPassword</name>
  	<value>hive</value>    
  	<!--//对应登录用户的密码-->
  </property>
  
  <property>
      <name>hive.metastore.warehouse.dir</name>  
  	<!--//HDFS创建的元数据目录-->
      <value>/hive/warehouse</value>
  </property>
  
  <property>
      <name>hive.exec.scratchdir</name>
      <value>/hive/temp</value>     
  	<!--//HDFS创建的临时目录路径-->
  </property>
  
  <property>
      <name>hive.querylog.location</name>
      <!--value>${system:java.io.tmpdir}/${system:user.name}</value-->
  	<value>/opt/hive/datas/logs</value>
      <description>Location of Hive run time structured log file</description>
  </property>
  
  <property>
      <name>hive.server2.logging.operation.log.location</name>
      <!--value>${system:java.io.tmpdir}/${system:user.name}/operation_logs</value-->
  	<value>/opt/hive/data/logs</value>
      <description>Top level directory where operation logs are stored if loggingfunctionality is enabled</description>
  </property>
  
  <property>
      <name>hive.exec.local.scratchdir</name>
  	<!--value>${system:java.io.tmpdir}/${system:user.name}</value-->
  	<value>/opt/hive/data/temp</value>
      <description>Local scratch space for Hive jobs</description>
  </property>
  
  <property>
      <name>hive.downloaded.resources.dir</name>
      <!--value>${system:java.io.tmpdir}/${hive.session.id}_resources</value-->
  	<value>/opt/hive/data/temp</value>
      <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
  
  ```

- 配置hive-site.xml
	``` cp hive-exec-log4j.properties.template  hive-exec-log4j.properties```
	``` cp hive-log4j.properties.template  hive-log4j.properties ```
	``` 修改property.hive.log.dir =/home/hadoop/hive-2.3.3/datas/logs```

