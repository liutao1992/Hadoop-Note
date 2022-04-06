### Hadoop3.3.0--Linux编译安装

基础环境：Centos 7.7

编译环境软件安装目录

```
mkdir -p /export/server
```

----

#### 一、Hadoop编译安装（选做）

> ==可以直接使用课程提供已经编译好的安装包==。

- 安装编译相关的依赖

  ```shell
  yum install gcc gcc-c++ make autoconf automake libtool curl lzo-devel zlib-devel openssl openssl-devel ncurses-devel snappy snappy-devel bzip2 bzip2-devel lzo lzo-devel lzop libXtst zlib -y
  
  yum install -y doxygen cyrus-sasl* saslwrapper-devel*
  ```
  
- 手动安装cmake 

  ```shell
  #yum卸载已安装cmake 版本低
  yum erase cmake
  
  #解压
  tar zxvf CMake-3.19.4.tar.gz
  
  #编译安装
  cd /export/server/CMake-3.19.4
  
  ./configure
  
  make && make install
  
  #验证
  [root@node4 ~]# cmake -version
  cmake version 3.19.4
  
  #如果没有正确显示版本 请断开SSH连接 重写登录
  ```

- 手动安装snappy

  ```shell
  #卸载已经安装的
  
  rm -rf /usr/local/lib/libsnappy*
  rm -rf /lib64/libsnappy*
  
  #上传解压
  tar zxvf snappy-1.1.3.tar.gz 
  
  #编译安装
  cd /export/server/snappy-1.1.3
  ./configure
  make && make install
  
  #验证是否安装
  [root@node4 snappy-1.1.3]# ls -lh /usr/local/lib |grep snappy
  -rw-r--r-- 1 root root 511K Nov  4 17:13 libsnappy.a
  -rwxr-xr-x 1 root root  955 Nov  4 17:13 libsnappy.la
  lrwxrwxrwx 1 root root   18 Nov  4 17:13 libsnappy.so -> libsnappy.so.1.3.0
  lrwxrwxrwx 1 root root   18 Nov  4 17:13 libsnappy.so.1 -> libsnappy.so.1.3.0
  -rwxr-xr-x 1 root root 253K Nov  4 17:13 libsnappy.so.1.3.0
  ```

- 安装配置JDK 1.8

  ```shell
  #解压安装包
  tar zxvf jdk-8u65-linux-x64.tar.gz
  
  #配置环境变量
  vim /etc/profile
  
  export JAVA_HOME=/export/server/jdk1.8.0_241
  export PATH=$PATH:$JAVA_HOME/bin
  export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
  
  source /etc/profile
  
  #验证是否安装成功
  java -version
  
  java version "1.8.0_241"
  Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
  Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)
  ```
  
- 安装配置maven

  ```shell
  #解压安装包
  tar zxvf apache-maven-3.5.4-bin.tar.gz
  
  #配置环境变量
  vim /etc/profile
  
  export MAVEN_HOME=/export/server/apache-maven-3.5.4
  export MAVEN_OPTS="-Xms4096m -Xmx4096m"
  export PATH=:$MAVEN_HOME/bin:$PATH
  
  source /etc/profile
  
  #验证是否安装成功
  [root@node4 ~]# mvn -v
  Apache Maven 3.5.4
  
  #添加maven 阿里云仓库地址 加快国内编译速度
  vim /export/server/apache-maven-3.5.4/conf/settings.xml
  
  <mirrors>
       <mirror>
             <id>alimaven</id>
             <name>aliyun maven</name>
             <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
             <mirrorOf>central</mirrorOf>
        </mirror>
  </mirrors>
  ```

- 安装ProtocolBuffer 3.7.1

  ```shell
  #卸载之前版本的protobuf
  
  #解压
  tar zxvf protobuf-3.7.1.tar.gz
  
  #编译安装
  cd /export/server/protobuf-3.7.1
  ./autogen.sh
  ./configure
  make && make install
  
  #验证是否安装成功
  [root@node4 protobuf-3.7.1]# protoc --version
  libprotoc 3.7.1
  ```

- 编译hadoop

  ```shell
  #上传解压源码包
  tar zxvf hadoop-3.3.0-src.tar.gz
  
  #编译
  cd /root/hadoop-3.3.0-src
  
  mvn clean package -Pdist,native -DskipTests -Dtar -Dbundle.snappy -Dsnappy.lib=/usr/local/lib
  
  #参数说明：
  
  Pdist,native ：把重新编译生成的hadoop动态库；
  DskipTests ：跳过测试
  Dtar ：最后把文件以tar打包
  Dbundle.snappy ：添加snappy压缩支持【默认官网下载的是不支持的】
  Dsnappy.lib=/usr/local/lib ：指snappy在编译机器上安装后的库路径
  ```

- 编译之后的安装包路径

  ```
  /root/hadoop-3.3.0-src/hadoop-dist/target
  ```

------

#### 二、Hadoop集群分布式安装

- 集群规划

  | 主机  | 角色                 |
  | ----- | -------------------- |
  | node1 | NN    DN   RM  NM    |
  | node2 | SNN  DN           NM |
  | node3 | DN          NM       |

- 基础环境

  > 3台机器都需要操作

  ```shell
  # 主机名 
  cat /etc/hostname
  
  # hosts映射
  vim /etc/hosts
  
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  
  192.168.88.151 node1.itcast.cn node1
  192.168.88.152 node2.itcast.cn node2
  192.168.88.153 node3.itcast.cn node3
  
  # JDK 1.8安装  上传 jdk-8u241-linux-x64.tar.gz到/export/server/目录下
  cd /export/server/
  tar zxvf jdk-8u241-linux-x64.tar.gz
  
  	#配置环境变量
  	vim /etc/profile
  	
  	export JAVA_HOME=/export/server/jdk1.8.0_241
  	export PATH=$PATH:$JAVA_HOME/bin
  	export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
  	
  	#重新加载环境变量文件
  	source /etc/profile
  
  # 集群时间同步
  ntpdate ntp5.aliyun.com
  
  # 防火墙关闭
  firewall-cmd --state	#查看防火墙状态
  systemctl stop firewalld.service  #停止firewalld服务
  systemctl disable firewalld.service  #开机禁用firewalld服务
  
  # ssh免密登录（只需要配置node1至node1、node2、node3即可）
  
  	#node1生成公钥私钥 (一路回车)
  	ssh-keygen  
  	
  	#node1配置免密登录到node1 node2 node3
  	ssh-copy-id node1
  	ssh-copy-id node2
  	ssh-copy-id node3
  ```

- 上传Hadoop安装包到node1 /export/server

  ```shell
  hadoop-3.3.0-Centos7-64-with-snappy.tar.gz
  
  tar zxvf hadoop-3.3.0-Centos7-64-with-snappy.tar.gz
  ```
  
- 修改配置文件(配置文件路径 hadoop-3.3.0/etc/hadoop)

  - hadoop-env.sh

    ```shell
    #文件最后添加
    export JAVA_HOME=/export/server/jdk1.8.0_241
    
    export HDFS_NAMENODE_USER=root
    export HDFS_DATANODE_USER=root
    export HDFS_SECONDARYNAMENODE_USER=root
    export YARN_RESOURCEMANAGER_USER=root
    export YARN_NODEMANAGER_USER=root 
    ```

  - core-site.xml

    ```xml
    <!-- 设置默认使用的文件系统 Hadoop支持file、HDFS、GFS、ali|Amazon云等文件系统 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://node1:8020</value>
    </property>
    
    <!-- 设置Hadoop本地保存数据路径 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/export/data/hadoop-3.3.0</value>
    </property>
    
    <!-- 设置HDFS web UI用户身份 -->
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>root</value>
    </property>
    
    <!-- 整合hive 用户代理设置 -->
    <property>
        <name>hadoop.proxyuser.root.hosts</name>
        <value>*</value>
    </property>
    
    <property>
        <name>hadoop.proxyuser.root.groups</name>
        <value>*</value>
    </property>
    
    <!-- 文件系统垃圾桶保存时间 -->
    <property>
        <name>fs.trash.interval</name>
        <value>1440</value>
    </property>
    ```
    
  - hdfs-site.xml

    ```xml
    <!-- 设置SNN进程运行机器位置信息 -->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>node2:9868</value>
    </property>
    ```

  - mapred-site.xml

    ```xml
    <!-- 设置MR程序默认运行模式： yarn集群模式 local本地模式 -->
    <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
    </property>
    
    <!-- MR程序历史服务地址 -->
    <property>
      <name>mapreduce.jobhistory.address</name>
      <value>node1:10020</value>
    </property>
     
    <!-- MR程序历史服务器web端地址 -->
    <property>
      <name>mapreduce.jobhistory.webapp.address</name>
      <value>node1:19888</value>
    </property>
    
    <property>
      <name>yarn.app.mapreduce.am.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    
    <property>
      <name>mapreduce.map.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    
    <property>
      <name>mapreduce.reduce.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    ```
    
  - yarn-site.xml

    ```xml
    <!-- 设置YARN集群主角色运行机器位置 -->
    <property>
    	<name>yarn.resourcemanager.hostname</name>
    	<value>node1</value>
    </property>
    
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    
    <!-- 是否将对容器实施物理内存限制 -->
    <property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
    </property>
    
    <!-- 是否将对容器实施虚拟内存限制。 -->
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
    
    <!-- 开启日志聚集 -->
    <property>
      <name>yarn.log-aggregation-enable</name>
      <value>true</value>
    </property>
    
    <!-- 设置yarn历史服务器地址 -->
    <property>
        <name>yarn.log.server.url</name>
        <value>http://node1:19888/jobhistory/logs</value>
    </property>
    
    <!-- 历史日志保存的时间 7天 -->
    <property>
      <name>yarn.log-aggregation.retain-seconds</name>
      <value>604800</value>
    </property>
    ```

  - workers

    ```
    node1.itcast.cn
    node2.itcast.cn
    node3.itcast.cn
    ```

- 分发同步hadoop安装包

  ```shell
  cd /export/server
  
  scp -r hadoop-3.3.0 root@node2:$PWD
  scp -r hadoop-3.3.0 root@node3:$PWD
  ```

- 将hadoop添加到环境变量（3台机器）

  ```shell
  vim /etc/profile
  
  export HADOOP_HOME=/export/server/hadoop-3.3.0
  export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  
  source /etc/profile
  
  
  #别忘了scp给其他两台机器哦
  ```

- Hadoop集群启动

  - （==首次启动==）格式化namenode

    ```shell
    hdfs namenode -format
    ```

  - 脚本一键启动

    ```shell
    [root@node1 ~]# start-dfs.sh 
    Starting namenodes on [node1]
    Last login: Thu Nov  5 10:44:10 CST 2020 on pts/0
    Starting datanodes
    Last login: Thu Nov  5 10:45:02 CST 2020 on pts/0
    Starting secondary namenodes [node2]
    Last login: Thu Nov  5 10:45:04 CST 2020 on pts/0
    
    [root@node1 ~]# start-yarn.sh 
    Starting resourcemanager
    Last login: Thu Nov  5 10:45:08 CST 2020 on pts/0
    Starting nodemanagers
    Last login: Thu Nov  5 10:45:44 CST 2020 on pts/0
    ```

  - Web  UI页面

    - HDFS集群：http://node1:9870/
    - YARN集群：http://node1:8088/

  

-----

- 错误1:运行hadoop3官方自带mr示例出错。

  - 错误信息

    ```shell
    Error: Could not find or load main class org.apache.hadoop.mapreduce.v2.app.MRAppMaster
    
    Please check whether your etc/hadoop/mapred-site.xml contains the below configuration:
    <property>
      <name>yarn.app.mapreduce.am.env</name>
      <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
    </property>
    <property>
      <name>mapreduce.map.env</name>
      <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
    </property>
    <property>
      <name>mapreduce.reduce.env</name>
      <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
    </property>
    ```

  - 解决  mapred-site.xml,增加以下配置

    ```xml
    <property>
      <name>yarn.app.mapreduce.am.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
      <name>mapreduce.map.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
      <name>mapreduce.reduce.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    ```

    



