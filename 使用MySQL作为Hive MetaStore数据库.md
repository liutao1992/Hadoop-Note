### Hive中使用MySQL作为MetaStore数据库

-----

#### Mysql安装

- 卸载Centos7自带的mariadb

  ```shell
  [root@node3 ~]# rpm -qa|grep mariadb
  mariadb-libs-5.5.64-1.el7.x86_64
  
  [root@node3 ~]# rpm -e mariadb-libs-5.5.64-1.el7.x86_64 --nodeps
  [root@node3 ~]# rpm -qa|grep mariadb                            
  [root@node3 ~]# 
  ```

- 安装mysql

  ```shell
  mkdir /export/software/mysql
  
  #上传mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar 到上述文件夹下  解压
  tar xvf mysql-5.7.29-1.el7.x86_64.rpm-bundle.tar
  
  #执行安装
  yum -y install libaio
  
  [root@node3 mysql]# rpm -ivh mysql-community-common-5.7.29-1.el7.x86_64.rpm mysql-community-libs-5.7.29-1.el7.x86_64.rpm mysql-community-client-5.7.29-1.el7.x86_64.rpm mysql-community-server-5.7.29-1.el7.x86_64.rpm 
  
  warning: mysql-community-common-5.7.29-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
  Preparing...                          ################################# [100%]
  Updating / installing...
     1:mysql-community-common-5.7.29-1.e################################# [ 25%]
     2:mysql-community-libs-5.7.29-1.el7################################# [ 50%]
     3:mysql-community-client-5.7.29-1.e################################# [ 75%]
     4:mysql-community-server-5.7.29-1.e################                  ( 49%)
  ```

- mysql初始化设置

  ```shell
  #初始化
  mysqld --initialize
  
  #更改所属组
  chown mysql:mysql /var/lib/mysql -R
  
  #启动mysql
  systemctl start mysqld.service
  
  #查看生成的临时root密码
  cat  /var/log/mysqld.log
  
  [Note] A temporary password is generated for root@localhost: o+TU+KDOm004
  ```

- 修改root密码 授权远程访问 设置开机自启动

  ```shell
  [root@node2 ~]# mysql -u root -p
  Enter password:     #这里输入在日志中生成的临时密码
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 3
  Server version: 5.7.29
  
  Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> 
  
  
  #更新root密码  设置为hadoop
  mysql> alter user user() identified by "hadoop";
  Query OK, 0 rows affected (0.00 sec)
  
  
  #授权
  mysql> use mysql;
  
  mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'hadoop' WITH GRANT OPTION;
  
  mysql> FLUSH PRIVILEGES; 
  
  #mysql的启动和关闭 状态查看 （这几个命令必须记住）
  systemctl stop mysqld
  systemctl status mysqld
  systemctl start mysqld
  
  #建议设置为开机自启动服务
  [root@node2 ~]# systemctl enable  mysqld                             
  Created symlink from /etc/systemd/system/multi-user.target.wants/mysqld.service to /usr/lib/systemd/system/mysqld.service.
  
  #查看是否已经设置自启动成功
  [root@node2 ~]# systemctl list-unit-files | grep mysqld
  mysqld.service                                enabled 
  ```

- Centos7 干净卸载mysql 5.7

  ```shell
  #关闭mysql服务
  systemctl stop mysqld.service
  
  #查找安装mysql的rpm包
  [root@node3 ~]# rpm -qa | grep -i mysql      
  mysql-community-libs-5.7.29-1.el7.x86_64
  mysql-community-common-5.7.29-1.el7.x86_64
  mysql-community-client-5.7.29-1.el7.x86_64
  mysql-community-server-5.7.29-1.el7.x86_64
  
  #卸载
  [root@node3 ~]# yum remove mysql-community-libs-5.7.29-1.el7.x86_64 mysql-community-common-5.7.29-1.el7.x86_64 mysql-community-client-5.7.29-1.el7.x86_64 mysql-community-server-5.7.29-1.el7.x86_64
  
  #查看是否卸载干净
  rpm -qa | grep -i mysql
  
  #查找mysql相关目录 删除
  [root@node1 ~]# find / -name mysql
  /var/lib/mysql
  /var/lib/mysql/mysql
  /usr/share/mysql
  
  [root@node1 ~]# rm -rf /var/lib/mysql
  [root@node1 ~]# rm -rf /var/lib/mysql/mysql

  [root@node1 ~]# rm -rf /usr/share/mysql
  
  #删除默认配置 日志
  rm -rf /etc/my.cnf 
  rm -rf /var/log/mysqld.log
  ```
  

---

#### Hive的安装

- 上传安装包 解压

  ```shell
  tar zxvf apache-hive-3.1.2-bin.tar.gz
  ```
  
- 解决Hive与Hadoop之间guava版本差异

  ```shell
  cd /export/server/apache-hive-3.1.2-bin/
  rm -rf lib/guava-19.0.jar
  cp /export/server/hadoop-3.3.0/share/hadoop/common/lib/guava-27.0-jre.jar ./lib/
  ```

- 修改配置文件

  - hive-env.sh

    ```shell
    cd /export/server/apache-hive-3.1.2-bin/conf
    mv hive-env.sh.template hive-env.sh
    
    vim hive-env.sh
    export HADOOP_HOME=/export/server/hadoop-3.3.0
    export HIVE_CONF_DIR=/export/server/apache-hive-3.1.2-bin/conf
    export HIVE_AUX_JARS_PATH=/export/server/apache-hive-3.1.2-bin/lib
    ```

  - hive-site.xml

    ```shell
    vim hive-site.xml
    ```

    ```xml
    <configuration>
    <!-- 存储元数据mysql相关配置 -->
    <property>
    	<name>javax.jdo.option.ConnectionURL</name>
    	<value>jdbc:mysql://node1:3306/hive3?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
    </property>
    
    <property>
    	<name>javax.jdo.option.ConnectionDriverName</name>
    	<value>com.mysql.jdbc.Driver</value>
    </property>
    
    <property>
    	<name>javax.jdo.option.ConnectionUserName</name>
    	<value>root</value>
    </property>
    
    <property>
    	<name>javax.jdo.option.ConnectionPassword</name>
    	<value>hadoop</value>
    </property>
    
    <!-- H2S运行绑定host -->
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>node1</value>
    </property>
    
    <!-- 远程模式部署metastore metastore地址 -->
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node1:9083</value>
    </property>
    
    <!-- 关闭元数据存储授权  --> 
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>
    </configuration>
    
    ```
  
- 上传mysql jdbc驱动到hive安装包lib下

  ```
  mysql-connector-java-5.1.32.jar
  ```

- 初始化元数据

  ```shell
  cd /export/server/apache-hive-3.1.2-bin/
  
  bin/schematool -initSchema -dbType mysql -verbos
  #初始化成功会在mysql中创建74张表
  ```

- 在hdfs创建hive存储目录（如存在则不用操作）

  ```shell
  hadoop fs -mkdir /tmp
  hadoop fs -mkdir -p /user/hive/warehouse
  hadoop fs -chmod g+w /tmp
  hadoop fs -chmod g+w /user/hive/warehouse
  ```

- ==启动hive==
- 1、启动metastore服务

    ```shell
    #前台启动  关闭ctrl+c
    /opt/hive-3.1.2/bin/hive --service metastore &

    #后台启动 进程挂起  关闭使用jps+ kill -9
    nohup /export/server/apache-hive-3.1.2-bin/bin/hive --service metastore &
    ```
  
- 2、启动hiveserver2服务
  
  ```shell
    nohup /export/server/apache-hive-3.1.2-bin/bin/hive --service hiveserver2 &
  
    #注意 启动hiveserver2需要一定的时间  不要启动之后立即beeline连接 可能连接不上
  ```
  
- 3、beeline客户端连接
    ```
    $ /opt/hive-3.1.2/bin/beeline

    beeline> ! connect jdbc:hive2://localhost:10000
    beeline> root
    beeline> 直接回车
    ```

[参考链接](https://cwiki.apache.org/confluence/display/Hive/Using+TiDB+as+the+Hive+Metastore+database)

