### HiveQL数据定义

#### Hive中的数据库
Hive 中数据库的概念本质上仅仅是表的一个目录或者命名空间。如果用户没有显示指定数据库，那么将会使用默认数据库default。

- 创建数据库

```
CREATE DATABASE financials
```

或者

```
CREATE DATABASE IF NOT EXISTS financials
```

- 查看hive中包含的数据库

```
SHOW DATABASES;
```

若数据库很多，则可以使用 like 进行过滤

```
SHOW DATABASES LIKE 'h.*'
```
> 该句将会过滤出h字母开头的数据库

Hive 会为每个数据库创建一个目录。数据库中的表将会以这个数据库的子目录形式存储。

> 默认数据库default除外。因为这个数据库本身没有自己的目录。


数据库所在的目录位于属性`hive.metastore.warehouse.dir`所指定的目录下。

我们可以通过以下命令来修改这个默认配置：
```
hive > CREATE DATABASE financials LOCATION '${path}'
```

也可以添加描述信息
```
hive > CREATE DATABASE financials COMMENT 'DESC'
```

查看数据库的描述信息
```
hive > DESCRIBE DATABASE financials
```

- 设置某个数据库为当前用户的工作数据库
```
USE financials
```

- 删除数据库

```
DROP DATABASE IF EXISTS financials
```
或
```
DROP DATABASE financials
```

默认情况下，Hive是不允许用户删除一个包含有表的数据库。用户要么需要先删除库中的表，然后再删除数据库；要么在删除命令的最后面加上关键字 CASCADE,这样可以使hive自行先删除数据库中的表。

```
DROP DATABASE IF EXISTS financials CASCADE
```

#### 创建表

- 创建表

CREATE TABLE 语句遵从SQL语法惯例，但hive这个语句具有显著的功能扩展，使其可以具有更广泛的灵活性。如定义表的数据文件存储在什么位置，使用什么样的格式存储等等。

```sql

CREATE TABLE IF NOT EXISTS financials.employees
(
    name    string comment "雇员姓名",
    salary  float  comment "雇员薪资",
    address string comment "雇员家庭住址"
)
-- 注释
COMMENT "表的描述信息"
--指定分隔符为制表符
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
```

- 拷贝表结构

```sql

CREATE TABLE financials.employees_bak like financials.employees;
```

- 查看表结构信息
```
DESCRIBE FORMATTED financials.employees_bak;
```

> 注意：我们目前所创建的表都是管理表，也被称为内部表。这种表，Hive会控制着数据的生命周期。正如我们所见，Hive默认情况下会将这些表的数据存储在由配置项`hive.metastore.warehouse.dir`所定义的目录的子目录下。
当我们删除一个管理表时，Hive也会删除这个表中数据。

有时候我们需要和其他工作共享同一份数据，我们在删除库时，不能一同也把数据删除，此时我们就需要外部表。


#### 外部表

- 创建外部表

```sql

CREATE EXTERNAL TABLE IF NOT EXISTS stocks (
    exchange string,
    symbol   string,
    ymd      string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/stocks';
```

关键字EXTERNAL告诉Hive这个是外部的，而后面的LOCATION... 子句则用于告诉Hive数据位于哪个路径下。

因为是外部表，所以Hive并非认为其完全拥有这份数据。因此，删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

- 查看表是管理表或外部表

```
DESCRIBE EXTENDED table_name
```

若该字段tableType显示
- MANAGED_TABLE 表示该表为管理表
- EXTERNAL_TABLE 表示该表为外部表


- 拷贝表结构

```sql

CREATE EXTERNAL TABLE IF NOT EXISTS db_msg.tb_msg_etl_share
    LIKE db_msg.tb_msg_etl
```

#### [分区表](https://juejin.cn/post/7040689938521653285)

数据分区的一般概念存在已久。其可以有多种形式，但通常使用分区来水平分散压力，将数据从物理上转移到和使用最频繁的用户更近的地方，以及实现其他的目的。

- 分区管理表

如：我们有以下数据：

dept_01.txt
```
10	ACCOUNTING	1700
20	RESEARCH	1800
```

dept_02.txt
```
30	SALES	1900
40	OPERATIONS	1700
```

dept_03.txt
```
50	TEST	2000
60	DEV	1900
```

##### 单节分区

- 创建分区

```sql
create table dept_partition(
    deptno int, 
    dname string, 
    loc string
)
partitioned by (day string)
row format delimited fields terminated by '\t';
```

> 注意：分区字段不能是表中已经存在的数据，可以将分区字段看作表的伪列。

- 查看分区表结构

```
show partitions dept_partition;
```

- 加载数据

```
load data local inpath '/opt/hivedata/dept_03.txt' into table db_msg.dept_partition
partition(day='20211205');


load data local inpath '/opt/hivedata/dept_03.txt' into table db_msg.dept_partition
partition(day='20211206');

load data local inpath '/opt/hivedata/dept_03.txt' into table db_msg.dept_partition
partition(day='20211207');
```

> 分区表加载数据时，必须指定分区，如果不指定会生成一个默认分区

#### 查看特定分区
```
show partitions db_msg.dept_partition partition (day=20211205);
```




















