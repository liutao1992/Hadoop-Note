### 向管理表中装载数据

```
LOAD DATA LOCAL INPATH '/PATH/' OVERWRITE INTO TABLE db_msg.dept_partition PARTITION (day='20211207');
```

 如果分区目录不存在的话，这个命令会先创建分区目录，然后在将数据拷贝到该目录下，如果目标表是非分区表，那么语句中省略PARTITION子句。

 如果使用了LOCAL关键字，那么这个路径应该为本地文件系统路径，数据将会被拷贝到目标位置。如果省略掉LOCAL关键字，那么这个路径应该是分布式文件系统中的路径。这种情况下，数据是从这个路径转移到目标位置的。

 > Hive要求源文件和目标文件以及目录应该在同一个文件系统中。如用户不可以使用LOCAL DATA语句将数据从一个集群的HDFS中转载到另一个集群的HDFS中。

 如果用户指定了`OVERWRITE`关键字，那么目标文件夹中之前的数据将会被先删除掉。如果没有这个关键字，仅仅会把新增的文件增加到目标文件夹中而不会删除之前的数据。然而，如果目标文件夹中已经存在和装载的文件同名的文件时，会保留之前的文件并且会重新命名新文件为“之前文件名_序列号”

 如果目标表是分区表，那么需要使用PARTITION子句，而且用户还必须为每个分区的键值指定一个值。如

 ```
LOAD DATA LOCAL INPATH '/PATH/' OVERWRITE INTO TABLE db_msg.dept_partition PARTITION (day='20211207');
```
这里我们指定分区的键值为20211207


#### 通过查询语句向表中插入数据

INSERT 语句允许用户通过查询语句向目标表中插入数据。

假设我们拥有这样一张表 , 该表中已经存放了相关数据。在这张表中我们使用不同的名字来表示国家和州，分别称作cnty和st。

```
INSERT OVERWRITE TABLE employees
PARTITION (country='US', state='OR') SELECT * FROM staged_employee se WHERE se.cnty='US' AND se.st = 'OR';
```

这里使用OVERWRITE，之前表中的内容将会被覆盖，若这里没有使用OVERWRITE关键字或者使用INTO关键字替换掉它的话，那么Hive将会以追加的方式写入数据而不会覆盖掉之前已经存在的内容。

#### 单个查询语句中创建表并加载数据

用户同样可以在一个语句中完成创建表并将查询结构载入这个表的操作。

```sql
CREATE TABLE ca_employees AS SELECT name, salary, address FROM employees WHERE se.state = 'CA'
```

这张表只包含有employees表中来自CA州的雇员的name、salary、address 3个字段的信息。新表的模式是根据SELECT语句来生成的。

使用这个功能的常见情况是从一个宽大的表中选取部分需要的数据集。这个功能不能用于外部表。



