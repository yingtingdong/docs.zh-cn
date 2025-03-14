# ALTER TABLE

## 功能

该语句用于修改已有表的结构。

> **注意**
>
> 该操作需要有对应表的 ALTER 权限。

## 语法

ALTER TABLE 语法格式如下：

```SQL
ALTER TABLE [database.]table
alter_clause1[, alter_clause2, ...]
```

其中 **alter_clause** 分为 partition、rollup、schema change、rename、index、swap、comment 操作，不同操作的应用场景为：

* partition: 修改分区属性，删除分区，增加分区。
* rollup: 创建或删除 rollup index。
* schema change: 增加列，删除列，调整列顺序，修改列类型。
* rename: 修改表名，rollup index 名称，修改 partition 名称，注意列名不支持修改。
* index: 修改索引(目前支持 bitmap 索引)。
* swap: 原子替换两张表。
* comment: 修改已有表的注释。**从 3.1 版本开始支持。**

> **说明**
>
> * partition、rollup 和 schema change 这三种操作不能同时出现在一条 `ALTER TABLE` 语句中。
> * rollup、schema change 是异步操作，命令提交成功后会立即返回一个成功消息，您可以使用 [SHOW ALTER TABLE](../data-manipulation/SHOW%20ALTER.md) 语句查看操作的进度。
> * partition、rename、swap 和 index 是同步操作，命令返回表示执行完毕。

### 操作 partition 相关语法

#### 增加分区 (ADD PARTITION)

语法：

```SQL
ALTER TABLE [database.]table 
ADD PARTITION [IF NOT EXISTS] partition_name
partition_desc ["key"="value"]
[DISTRIBUTED BY HASH (k1[,k2 ...]) [BUCKETS num]];
```

注意：

1. partition_desc 支持以下两种写法：
    * VALUES LESS THAN [MAXVALUE|("value1", ...)]
    * VALUES [("value1", ...), ("value1", ...))
2. 分区为左闭右开区间，如果用户仅指定右边界，系统会自动确定左边界。
3. 如果没有指定分桶方式，则自动使用建表使用的分桶方式。
4. 如指定分桶方式，只能修改分桶数，不可修改分桶方式或分桶列。
5. `["key" = "value"]` 部分可以设置分区的一些属性，具体说明见 [CREATE TABLE](../data-definition/CREATE%20TABLE.md)。

#### 删除分区 (DROP PARTITION)

语法：

```sql
-- 2.0之前版本
ALTER TABLE [database.]table
DROP PARTITION [IF EXISTS | FORCE] partition_name;
-- 2.0及之后版本
ALTER TABLE [database.]table
DROP PARTITION [IF EXISTS] partition_name [FORCE];
```

注意：

1. 使用分区方式的表至少要保留一个分区。
2. 执行 DROP PARTITION 一段时间内，可以通过 RECOVER 语句恢复被删除的分区。详见 [RECOVER](../data-definition/RECOVER.md) 语句。
3. 如果执行 DROP PARTITION FORCE，则系统不会检查该分区是否存在未完成的事务，分区将直接被删除并且不能被恢复，一般不建议执行此操作。

#### 增加临时分区 (ADD TEMPORARY PARTITION)

详细使用信息，请查阅[临时分区](../../../table_design/Temporary_partition.md)。

语法：

```SQL
ALTER TABLE [database.]table 
ADD TEMPORARY PARTITION [IF NOT EXISTS] partition_name
partition_desc ["key"="value"]
[DISTRIBUTED BY HASH (k1[,k2 ...]) [BUCKETS num]];
```

#### 使用临时分区替换原分区

语法：

```SQL
ALTER TABLE [database.]table 
REPLACE PARTITION partition_name 
partition_desc ["key"="value"]
WITH TEMPORARY PARTITION
partition_desc ["key"="value"]
[PROPERTIES ("key"="value", ...)]
```

#### 删除临时分区 (DROP TEMPORARY PARTITION)

语法：

```sql
ALTER TABLE [database.]table
DROP TEMPORARY PARTITION partition_name;
```

#### 修改分区属性 (MODIFY PARTITION)

语法：

```sql
ALTER TABLE [database.]table
MODIFY PARTITION p1|(p1[, p2, ...]) SET ("key" = "value", ...);
```

说明：

1. 当前支持修改分区的下列属性：
    * storage_medium
    * storage_cooldown_time
    * replication_num

2. 对于单分区表，partition_name 同表名。

### 操作 rollup 相关语法

#### 创建 rollup index (ADD ROLLUP)

**RollUp 表索引**: shortkey index 可加速数据查找，但 shortkey index 依赖维度列排列次序。如果使用非前缀的维度列构造查找谓词，用户可以为数据表创建若干 RollUp 表索引。 RollUp 表索引的数据组织和存储和数据表相同，但 RollUp 表拥有自身的 shortkey index。用户创建 RollUp 表索引时，可选择聚合的粒度，列的数量，维度列的次序。使频繁使用的查询条件能够命中相应的 RollUp 表索引。

语法：

```SQL
ALTER TABLE [database.]table 
ADD ROLLUP rollup_name (column_name1, column_name2, ...)
[FROM from_index_name]
[PROPERTIES ("key"="value", ...)];
```

properties: 支持设置超时时间，默认超时时间为 1 天。

例子：

```SQL
ALTER TABLE [database.]table 
ADD ROLLUP r1(col1,col2) from r0;
```

#### 批量创建 rollup index

语法：

```SQL
ALTER TABLE [database.]table
ADD ROLLUP [rollup_name (column_name1, column_name2, ...)
[FROM from_index_name]
[PROPERTIES ("key"="value", ...)],...];
```

例子：

```SQL
ALTER TABLE [database.]table
ADD ROLLUP r1(col1,col2) from r0, r2(col3,col4) from r0;
```

> 注意
>
1. 如果没有指定 from_index_name，则默认从 base index 创建。
2. rollup 表中的列必须是 from_index 中已有的列。
3. 在 properties 中，可以指定存储格式。具体请参阅 [CREATE TABLE](../data-definition/CREATE%20TABLE.md) 章节。

#### 删除 rollup index (DROP ROLLUP)

语法：

```SQL
ALTER TABLE [database.]table
DROP ROLLUP rollup_name [PROPERTIES ("key"="value", ...)];
```

例子：

```sql
ALTER TABLE [database.]table DROP ROLLUP r1;
```

#### 批量删除 rollup index

语法：

```sql
ALTER TABLE [database.]table
DROP ROLLUP [rollup_name [PROPERTIES ("key"="value", ...)],...];
```

例子：

```sql
ALTER TABLE [database.]table DROP ROLLUP r1, r2;
```

注意：

不能删除 base index。

### Schema change

下文中的 index 为物化索引。建表成功后表为 base 表 (base index)，基于 base 表可 [创建 rollup index](#创建-rollup-index-add-rollup)。

base index 和 rollup index 都是物化索引。下方语句在编写时如果没有指定 `rollup_index_name`，默认操作基表。

#### 向指定 index 的指定位置添加一列 (ADD COLUMN)

语法：

```sql
ALTER TABLE [database.]table
ADD COLUMN column_name column_type [KEY | agg_type] [DEFAULT "default_value"]
[AFTER column_name|FIRST]
[TO rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

```plain text
1. 聚合模型如果增加 value 列，需要指定 agg_type。
2. 非聚合模型（如 DUPLICATE KEY）如果增加 key 列，需要指定 KEY 关键字。
3. 不能在 rollup index 中增加 base index 中已经存在的列，如有需要，可以重新创建一个 rollup index。
```

#### 向指定 index 添加多列

语法：

```sql
ALTER TABLE [database.]table
ADD COLUMN (column_name1 column_type [KEY | agg_type] DEFAULT "default_value", ...)
[TO rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

1. 聚合模型如果增加 value 列，需要指定 agg_type。
2. 非聚合模型如果增加 key 列，需要指定 KEY 关键字。
3. 不能在 rollup index 中增加 base index 中已经存在的列，如有需要，可以重新创建一个 rollup index。

#### 从指定 index 中删除一列 (DROP COLUMN)

语法：

```sql
ALTER TABLE [database.]table
DROP COLUMN column_name
[FROM rollup_index_name];
```

注意：

1. 不能删除分区列。
2. 如果是从 base index 中删除列，则如果 rollup index 中包含该列，也会被删除。

#### 修改指定 index 的列类型以及列位置 (MODIFY COLUMN)

语法：

```sql
ALTER TABLE [database.]table
MODIFY COLUMN column_name column_type [KEY | agg_type] [NULL | NOT NULL] [DEFAULT "default_value"]
[AFTER column_name|FIRST]
[FROM rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

1. 聚合模型如果修改 value 列，需要指定 agg_type。
2. 非聚合类型如果修改 key 列，需要指定 KEY 关键字。
3. 只能修改列的类型，列的其他属性维持原样（即其他属性需在语句中按照原属性显式的写出，参见样例中 [Schema Change](#schema-change-1) 部分第 8 个例子）。
4. 分区列不能做任何修改。
5. 目前支持以下类型的转换（精度损失由用户保证）：

    * TINYINT/SMALLINT/INT/BIGINT 转换成 TINYINT/SMALLINT/INT/BIGINT/DOUBLE。
    * TINTINT/SMALLINT/INT/BIGINT/LARGEINT/FLOAT/DOUBLE/DECIMAL 转换成 VARCHAR。
    * VARCHAR 支持修改最大长度。
    * VARCHAR 转换成 TINTINT/SMALLINT/INT/BIGINT/LARGEINT/FLOAT/DOUBLE。
    * VARCHAR 转换成 DATE (目前支持 "%Y-%m-%d"，"%y-%m-%d"， "%Y%m%d"，"%y%m%d"，"%Y/%m/%d，"%y/%m/%d " 六种格式化格式)
    DATETIME 转换成 DATE(仅保留年-月-日信息，例如: `2019-12-09 21:47:05` <--> `2019-12-09`)
    DATE 转换成 DATETIME(时分秒自动补零，例如: `2019-12-09` <--> `2019-12-09 00:00:00`)
    * FLOAT 转换成 DOUBLE。
    * INT 转换成 DATE (如果 INT 类型数据不合法则转换失败，原始数据不变。)

6. 不支持从 NULL 转为 NOT NULL。

#### 对指定 index 的列进行重新排序

语法：

```sql
ALTER TABLE [database.]table
ORDER BY (column_name1, column_name2, ...)
[FROM rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

1. index 中的所有列都要写出来。
2. value 列在 key 列之后。

#### 增加生成列

语法：

```sql
ALTER TABLE [database.]table
ADD col_name data_type [NULL] AS generation_expr [COMMENT 'string']
```

增加生成列并且指定其使用的表达式。[生成列](../generated_columns.md)用于预先计算并存储表达式的结果，可以加速包含复杂表达式的查询。自 v3.1，StarRocks 支持该功能。

#### 修改表的属性

目前支持修改 `bloom_filter_columns`，`colocate_with`， `dynamic_partition` 属性，`enable_persistent_index` 属性，`replication_num` 和 `default.replication_num` 属性。

语法：

```sql
PROPERTIES ("key"="value")
```

注意：
也可以合并到上面的 schema change 操作中来修改，见下面例子。

### Rename 对名称进行修改

#### 修改表名

语法：

```sql
ALTER TABLE table_name RENAME new_table_name;
```

#### 修改 rollup index 名称 (RENAME ROLLUP)

语法：

```sql
ALTER TABLE [database.]table
RENAME ROLLUP old_rollup_name new_rollup_name;
```

#### 修改 partition 名称 (RENAME PARTITION)

语法：

```sql
ALTER TABLE [database.]table
RENAME PARTITION old_partition_name new_partition_name;
```

### Bitmap index 修改

#### 创建 Bitmap 索引 (ADD INDEX)

语法：

```sql
ALTER TABLE [database.]table
ADD INDEX index_name (column [, ...],) [USING BITMAP] [COMMENT 'balabala'];
```

注意：

```plain text
1. 目前仅支持bitmap 索引。
2. BITMAP 索引仅在单列上创建。
```

#### 删除索引 (DROP INDEX)

语法：

```sql
ALTER TABLE [database.]table
DROP INDEX index_name;
```

### Swap 将两个表原子替换

语法：

```sql
ALTER TABLE [database.]table
SWAP WITH table_name;
```

### 修改表的注释（3.1 版本起）

语法：

```sql
ALTER TABLE [database.]table COMMENT = "<new table comment>";
```

## 示例

### table

1. 修改表的默认副本数量，新建分区副本数量默认使用此值。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("default.replication_num" = "2");
    ```

2. 修改单分区表的实际副本数量(只限单分区表)。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("replication_num" = "3");
    ```

3. 修改数据在多副本间的写入和同步方式。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("replicated_storage" = "false");
    ```

    以上示例表示将多副本的写入和同步方式设置为 leaderless replication，即数据同时写入到多个副本，不区分主从副本。更多信息，参见 [CREATE TABLE](CREATE%20TABLE.md) 的 `replicated_storage` 参数描述。

### partition

1. 增加分区，现有分区 [MIN, 2013-01-01)，增加分区 [2013-01-01, 2014-01-01)，使用默认分桶方式。

    ```sql
    ALTER TABLE example_db.my_table
    ADD PARTITION p1 VALUES LESS THAN ("2014-01-01");
    ```

2. 增加分区，使用新的分桶数。

    ```sql
    ALTER TABLE example_db.my_table
    ADD PARTITION p1 VALUES LESS THAN ("2015-01-01")
    DISTRIBUTED BY HASH(k1) BUCKETS 20;
    ```

3. 增加分区，使用新的副本数。

    ```sql
    ALTER TABLE example_db.my_table
    ADD PARTITION p1 VALUES LESS THAN ("2015-01-01")
    ("replication_num"="1");
    ```

4. 修改分区副本数。

    ```sql
    ALTER TABLE example_db.my_table
    MODIFY PARTITION p1 SET("replication_num"="1");
    ```

5. 批量修改指定分区。

    ```sql
    ALTER TABLE example_db.my_table
    MODIFY PARTITION (p1, p2, p4) SET("replication_num"="1");
    ```

6. 批量修改所有分区。

    ```sql
    ALTER TABLE example_db.my_table
    MODIFY PARTITION (*) SET("storage_medium"="HDD");
    ```

7. 删除分区。

    ```sql
    ALTER TABLE example_db.my_table
    DROP PARTITION p1;
    ```

8. 增加一个指定上下界的分区。

    ```sql
    ALTER TABLE example_db.my_table
    ADD PARTITION p1 VALUES [("2014-01-01"), ("2014-02-01"));
    ```

### rollup

1. 创建 index `example_rollup_index`，基于 base index（k1, k2, k3, v1, v2），列式存储。

    ```sql
    ALTER TABLE example_db.my_table
    ADD ROLLUP example_rollup_index(k1, k3, v1, v2)
    PROPERTIES("storage_type"="column");
    ```

2. 创建 index `example_rollup_index2`，基于 example_rollup_index（k1, k3, v1, v2）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD ROLLUP example_rollup_index2 (k1, v1)
    FROM example_rollup_index;
    ```

3. 创建 index `example_rollup_index3`，基于 base index (k1, k2, k3, v1), 自定义 rollup 超时时间一小时。

    ```sql
    ALTER TABLE example_db.my_table
    ADD ROLLUP example_rollup_index3(k1, k3, v1)
    PROPERTIES("storage_type"="column", "timeout" = "3600");
    ```

4. 删除 index `example_rollup_index2`。

    ```sql
    ALTER TABLE example_db.my_table
    DROP ROLLUP example_rollup_index2;
    ```

### Schema Change

1. 向 `example_rollup_index` 的 `col1` 后添加一个 key 列 `new_col`（非聚合模型）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN new_col INT KEY DEFAULT "0" AFTER col1
    TO example_rollup_index;
    ```

2. 向 `example_rollup_index` 的 `col1` 后添加一个 value 列 `new_col`（非聚合模型）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN new_col INT DEFAULT "0" AFTER col1
    TO example_rollup_index;
    ```

3. 向 `example_rollup_index` 的 `col1` 后添加一个 key 列 `new_col`（聚合模型）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN new_col INT DEFAULT "0" AFTER col1
    TO example_rollup_index;
    ```

4. 向 `example_rollup_index` 的 `col1` 后添加一个 value 列 `new_col`（SUM 聚合类型）（聚合模型）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN new_col INT SUM DEFAULT "0" AFTER col1
    TO example_rollup_index;
    ```

5. 向 `example_rollup_index` 添加多列（聚合模型）。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN (col1 INT DEFAULT "1", col2 FLOAT SUM DEFAULT "2.3")
    TO example_rollup_index;
    ```

6. 从 `example_rollup_index` 删除一列。

    ```sql
    ALTER TABLE example_db.my_table
    DROP COLUMN col2
    FROM example_rollup_index;
    ```

7. 修改 base index 的 `col1` 列的类型为 BIGINT，并移动到 `col2` 列后面。

    ```sql
    ALTER TABLE example_db.my_table
    MODIFY COLUMN col1 BIGINT DEFAULT "1" AFTER col2;
    ```

8. 修改 base index 的 `val1` 列最大长度。原 `val1` 为 (`val1 VARCHAR(32) REPLACE DEFAULT "abc"`)。

    ```sql
    ALTER TABLE example_db.my_table
    MODIFY COLUMN val1 VARCHAR(64) REPLACE DEFAULT "abc";
    ```

9. 重新排序 `example_rollup_index` 中的列（设原列顺序为：k1, k2, k3, v1, v2）。

    ```sql
    ALTER TABLE example_db.my_table
    ORDER BY (k3,k1,k2,v2,v1)
    FROM example_rollup_index;
    ```

10. 同时执行两种操作。

    ```sql
    ALTER TABLE example_db.my_table
    ADD COLUMN v2 INT MAX DEFAULT "0" AFTER k2 TO example_rollup_index,
    ORDER BY (k3,k1,k2,v2,v1) FROM example_rollup_index;
    ```

11. 修改表的 bloom filter 列。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("bloom_filter_columns"="k1,k2,k3");
    ```

    也可以合并到上面的 schema change 操作中（注意多子句的语法有少许区别）

    ```sql
    ALTER TABLE example_db.my_table
    DROP COLUMN col2
    PROPERTIES ("bloom_filter_columns"="k1,k2,k3");
    ```

12. 修改表的 Colocate 属性。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("colocate_with" = "t1");
    ```

13. 将表的分桶方式由 Random Distribution 改为 Hash Distribution。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("distribution_type" = "hash");
    ```

14. 修改表的动态分区属性(支持未添加动态分区属性的表添加动态分区属性)。

    ```sql
    ALTER TABLE example_db.my_table
    SET ("dynamic_partition.enable" = "false");
    ```

    如果需要在未添加动态分区属性的表中添加动态分区属性，则需要指定所有的动态分区属性。

    ```sql
    ALTER TABLE example_db.my_table
    SET (
        "dynamic_partition.enable" = "true",
        "dynamic_partition.time_unit" = "DAY",
        "dynamic_partition.end" = "3",
        "dynamic_partition.prefix" = "p",
        "dynamic_partition.buckets" = "32"
        );
    ```

### rename

1. 将名为 `table1` 的表修改为 `table2`。

    ```sql
    ALTER TABLE table1 RENAME table2;
    ```

2. 将表 `example_table` 中名为 `rollup1` 的 rollup index 修改为 `rollup2`。

    ```sql
    ALTER TABLE example_table RENAME ROLLUP rollup1 rollup2;
    ```

3. 将表 `example_table` 中名为 `p1` 的 partition 修改为 `p2`。

    ```sql
    ALTER TABLE example_table RENAME PARTITION p1 p2;
    ```

### index

1. 在 `table1` 上为 `siteid` 创建 `bitmap` 索引。

    ```sql
    ALTER TABLE table1
    ADD INDEX index_name (siteid) [USING BITMAP] COMMENT 'balabala';
    ```

2. 删除 `table1` 上的 `siteid` 列的 bitmap 索引。

    ```sql
    ALTER TABLE table1 DROP INDEX index_name;
    ```

### swap

将 `table1` 与 `table2` 原子替换。

```sql
ALTER TABLE table1 SWAP WITH table2;
```

## 相关参考

* [CREATE TABLE](CREATE%20TABLE.md)
* [SHOW CREATE TABLE](../data-manipulation/SHOW%20CREATE%20TABLE.md)
* [SHOW TABLES](../data-manipulation/SHOW%20TABLES.md)
* [SHOW ALTER TABLE](../data-manipulation/SHOW%20ALTER.md)
* [DROP TABLE](DROP%20TABLE.md)
