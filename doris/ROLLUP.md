## ROLLUP

ROLLUP 在多维分析中是“上卷”的意思，即将数据按某种指定的粒度进行进一步聚合。



### 基本概念

在 Doris 中，我们将用户通过建表语句创建出来的表称为 Base 表（Base Table）。Base 表中保存着按用户建表语句指定的方式存储的基础数据。

在 Base 表之上，我们可以创建任意多个 ROLLUP 表。这些 ROLLUP 的数据是基于 Base 表产生的，并且在物理上是**独立存储**的。

ROLLUP 表的基本作用，在于在 Base 表的基础上，获得更粗粒度的聚合数据。

下面我们用示例详细说明在不同数据模型中的 ROLLUP 表及其作用



### Aggregate 和 Uniq 模型中的 ROLLUP

因为 Uniq 只是 Aggregate 模型的一个特例，所以这里我们不加以区别。



#### 示例1：获得每个用户的总消费

接**Aggregate 模型**小节的**示例2**，Base 表结构如下：

> 实例2主要新增了timestamp字段，精确到秒。

| ColumnName      | Type        | AggregationType | Comment                |
| --------------- | ----------- | --------------- | ---------------------- |
| user_id         | LARGEINT    |                 | 用户id                 |
| date            | DATE        |                 | 数据灌入日期           |
| timestamp       | DATETIME    |                 | 数据灌入时间，精确到秒 |
| city            | VARCHAR(20) |                 | 用户所在城市           |
| age             | SMALLINT    |                 | 用户年龄               |
| sex             | TINYINT     |                 | 用户性别               |
| last_visit_date | DATETIME    | REPLACE         | 用户最后一次访问时间   |
| cost            | BIGINT      | SUM             | 用户总消费             |
| max_dwell_time  | INT         | MAX             | 用户最大停留时间       |
| min_dwell_time  | INT         | MIN             | 用户最小停留时间       |

存储的数据如下：

| user_id | date       | timestamp           | city | age  | sex  | last_visit_date     | cost | max_dwell_time | min_dwell_time |
| ------- | ---------- | ------------------- | ---- | ---- | ---- | ------------------- | ---- | -------------- | -------------- |
| 10000   | 2017-10-01 | 2017-10-01 08:00:05 | 北京 | 20   | 0    | 2017-10-01 06:00:00 | 20   | 10             | 10             |
| 10000   | 2017-10-01 | 2017-10-01 09:00:05 | 北京 | 20   | 0    | 2017-10-01 07:00:00 | 15   | 2              | 2              |
| 10001   | 2017-10-01 | 2017-10-01 18:12:10 | 北京 | 30   | 1    | 2017-10-01 17:05:45 | 2    | 22             | 22             |
| 10002   | 2017-10-02 | 2017-10-02 13:10:00 | 上海 | 20   | 1    | 2017-10-02 12:59:12 | 200  | 5              | 5              |
| 10003   | 2017-10-02 | 2017-10-02 13:15:00 | 广州 | 32   | 0    | 2017-10-02 11:20:00 | 30   | 11             | 11             |
| 10004   | 2017-10-01 | 2017-10-01 12:12:48 | 深圳 | 35   | 0    | 2017-10-01 10:00:15 | 100  | 3              | 3              |
| 10004   | 2017-10-03 | 2017-10-03 12:38:20 | 深圳 | 35   | 0    | 2017-10-03 10:20:22 | 11   | 6              | 6              |

在此基础上，我们创建一个 ROLLUP：

| ColumnName |
| ---------- |
| user_id    |
| cost       |

该 ROLLUP 只包含两列：user_id 和 cost。则创建完成后，该 ROLLUP 中存储的数据如下：

| user_id | cost |
| ------- | ---- |
| 10000   | 35   |
| 10001   | 2    |
| 10002   | 200  |
| 10003   | 30   |
| 10004   | 111  |

可以看到，ROLLUP 中仅保留了每个 user_id，在 cost 列上的 SUM 的结果。那么当我们进行如下查询时:

```
SELECT user_id, sum(cost) FROM table GROUP BY user_id;
```

Doris 会自动命中这个 ROLLUP 表，从而只需扫描极少的数据量，即可完成这次聚合查询。



#### 示例2：获得不同城市，不同年龄段用户的总消费、最长和最短页面驻留时间

紧接示例1。我们在 Base 表基础之上，再创建一个 ROLLUP：

| ColumnName     | Type        | AggregationType | Comment          |
| -------------- | ----------- | --------------- | ---------------- |
| city           | VARCHAR(20) |                 | 用户所在城市     |
| age            | SMALLINT    |                 | 用户年龄         |
| cost           | BIGINT      | SUM             | 用户总消费       |
| max_dwell_time | INT         | MAX             | 用户最大停留时间 |
| min_dwell_time | INT         | MIN             | 用户最小停留时间 |

则创建完成后，该 ROLLUP 中存储的数据如下：

| city | age  | cost | max_dwell_time | min_dwell_time |
| ---- | ---- | ---- | -------------- | -------------- |
| 北京 | 20   | 0    | 30             | 10             |
| 北京 | 30   | 1    | 2              | 22             |
| 上海 | 20   | 1    | 200            | 5              |
| 广州 | 32   | 0    | 30             | 11             |
| 深圳 | 35   | 0    | 111            | 6              |

当我们进行如下这些查询时:

- `SELECT city, age, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM table GROUP BY city, age;`
- `SELECT city, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM table GROUP BY city;`
- `SELECT city, age, sum(cost), min(min_dwell_time) FROM table GROUP BY city, age;`

Doris 会自动命中这个 ROLLUP 表。

### Duplicate 模型中的 ROLLUP

因为 Duplicate 模型没有聚合的语意。所以该模型中的 ROLLUP，已经失去了“上卷”这一层含义。而仅仅是作为调整列顺序，以命中前缀索引的作用。我们将在接下来的小节中，详细介绍前缀索引，以及如何使用ROLLUP改变前缀索引，以获得更好的查询效率。
