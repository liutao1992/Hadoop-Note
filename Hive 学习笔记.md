
### 启动hadoop服务

```
$ /opt/hadoop-3.2.2/sbin/start-all.sh
```

### 格式化文件系统

若是采用默认配置，第一次启动或者是重启，则需要格式化文件系统。同时也要注意，一旦格式化，仓库里的数据也就不存在了。
```
$ /opt/hadoop-3.2.2/bin/hdfs namenode -format
```

### 访问NameNode节点

```
http://localhost:9870/
```

### 启动hive服务

- 创建仓库目录

如若存在，则可忽略

```
  $ /opt/hadoop-3.2.2/bin/hadoop fs -mkdir     /tmp
  $ /opt/hadoop-3.2.2/bin/hadoop fs -mkdir -p  /user/hive/warehouse
  $ /opt/hadoop-3.2.2/bin/hadoop fs -chmod g+w /tmp
  $ /opt/hadoop-3.2.2/bin/hadoop fs -chmod g+w /user/hive/warehouse
```

- 启动metastore服务

```
$ /opt/hive-3.1.2/bin/hive --service metastore &
```

- 启动hiveserver2服务
```
$ /opt/hive-3.1.2/bin/hive --service hiveserver2 &
```

- beeline客户端连接

```
$ /opt/hive-3.1.2/bin/beeline

beeline> ! connect jdbc:hive2://localhost:10000
beeline> root
beeline> 直接回车
```

### hive 加载数据

通过beeline客户端加载数据

- 创建表

```
create table t_archer(
id int comment "ID",
name string comment "英雄名称", hp_max int comment "最大生命", mp_max int comment "最大法力", attack_max int comment "最高物攻", defense_max int comment "最大物防", attack_range string comment "攻击范围", role_main string comment "主要定位", role_assist string comment "次要定位"
) comment "王者荣耀射手信息" row format delimited
fields terminated by "\t";
```



- 加载本地数据

`LOAD DATA LOCAL INPATH '$PATH' INTO TABLE t_archer`

