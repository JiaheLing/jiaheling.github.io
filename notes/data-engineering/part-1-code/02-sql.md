---
title: "SQL基础"
---

## 目录

_本笔记默认掌握基础的SQL语法，仅讨论SQL常见问题的算法逻辑，如果需要详细学习SQL，请查看[Hive]()或者[Spark]()部分的笔记。_

- [1. SQL顺序](#1-sql顺序)
  - [语法顺序](#语法顺序)
  - [逻辑执行顺序](#逻辑执行顺序)
  - [物理执行顺序](#物理执行顺序)

- [2. TopN问题](#2-topn问题)
  - [TopN问题概述](#topn问题概述)
  - [窗口函数和窗口帧](#窗口函数和窗口帧)
  - [全局TopN问题](#全局topn问题)
  - [分组TopN问题](#分组topn问题)

- [3. 连续登陆问题](#3-连续登陆问题)
  - [连续登陆问题概述](#连续登陆问题概述)
  - [常用函数](#常用函数)
  - [常规解法](#常规解法)
  - [其他解法](#其他解法)
  - [间隔连续登陆（其他解法plus）](#间隔连续登陆其他解法plus)
  - [题型分类](#题型分类)

- [4. 行转列/列转行问题](#4-行转列列转行问题)
  - [行转列问题概述](#行转列问题概述)
  - [常用函数（列转行）](#常用函数列转行)
  - [常用函数（行转列）](#常用函数行转列)
  - [列转行解法](#列转行解法)
  - [行转列解法](#行转列解法)
  - [有序行转列解法](#有序行转列解法)

- [5. 同时在线问题](#5-同时在线问题)
  - [同时在线问题概述](#同时在线问题概述)
  - [同时在线问题解法](#同时在线问题解法)

- [6. 留存问题](#6-留存问题)
  - [留存问题核心参数](#留存问题核心参数)
  - [留存问题常用函数](#留存问题常用函数)
  - [留存问题解法](#留存问题解法)

- [7. 开窗/聚合函数（重点）](#7-开窗聚合函数重点)
  - [波峰波谷](#波峰波谷)
  - [掐头去尾](#掐头去尾)
  - [合并区间](#合并区间)
  - [空值填充](#空值填充)
  - [前后列转换](#前后列转换)
  - [相互关注](#相互关注)
  - [无效搜索](#无效搜索)
  - [相邻问题](#相邻问题)

- [8. 正则表达](#8-正则表达)
  - [正则表达式概述](#正则表达式概述)
  - [相关函数](#相关函数)

- [9. JSON字符串与解析](#9-json字符串与解析)

- [10. SQL高性能优化](#10-sql高性能优化)

---
## 1. SQL顺序

### 语法顺序
1. SELECT: 指定要查询的列
2. DISTINCT: 去除结果集中的重复行
3. FROM: 指定查询的表
4. JOIN, ON: 连接多个表并ON指定连接条件
5. WHERE: 过滤记录
6. GROUP BY: 对结果集进行分组
7. HAVING: 过滤分组后的结果集
8. ORDER BY: 对结果集进行排序
9. LIMIT: 限制结果集返回的行数

### 逻辑执行顺序

SQL查询语句在逻辑上被解析和执行的顺序，它反映了查询语句的结构和语义.

1. FROM
2. JOIN, ON
3. WHERE
4. GROUP BY
5. HAVING
6. SELECT
7. DISTINCT
8. ORDER BY
9. LIMIT

### 物理执行顺序

- **逻辑执行顺序**：是对查询语句的抽象表示，它描述了查询的语义和逻辑结构，如表连接、过滤条件、分组、排序等。逻辑执行计划并不涉及具体的执行细节，如数据的存储方式、数据的分区情况等。
- **物理执行顺序**：是根据逻辑执行计划和具体的执行环境（如数据的存储方式、集群的配置等）生成的，它包含了查询执行的具体细节，如数据的读取方式、数据的处理方式（如过滤、连接、排序等）以及数据的输出方式等。一般来说，先有逻辑执行计划，再生成具体的物理执行计划。
> 最常见物理执行顺序改变逻辑执行顺序的是**过滤条件下推**（将WHERE操作提前），即在物理执行过程中，过滤条件可能会被提前应用到数据源，以减少后续操作的数据量。

---
## 2. TopN问题

### TopN问题概述

TopN问题是指从数据表中查询出前N个最大或最小的数据记录（一旦题目中出现：按照指标X取前10名、前50%、后25%、第1名等等字眼，基本可以定位该问题为TopN问题）。

TopN问题可以分为以下2类：
1. 全局TopN：从整个数据表中查询前N个最大或最小的记录。
2. 分组TopN：按照某个字段分组后，在每个组内查询前N个最大或最小的记录。

### 窗口函数和窗口帧

#### 窗口函数

1. `ROW_NUMBER()`：为结果集中的每一行分配一个**唯一且连续的序号**。即使排序字段的值相同，不同行仍然会获得不同的序号。
    ```sql
   SELECT *, 
        ROW_NUMBER() OVER (
            PARTITION BY dept
            ORDER BY salary DESC
        ) AS rn
   FROM employee
   ```

    * `PARTITION BY dept`：每个部门分别排名。
    * `ORDER BY salary DESC`：部门内部按照工资从高到低排序
    * 适合严格取每组前 N 条记录，例如每个部门工资最高的 3 名员工。

2. `RANK()`：为结果集中的每一行分配排名。排序值相同的行会获得相同排名，后续排名会**跳号**。

   ```sql
   RANK() OVER (
       PARTITION BY dept
       ORDER BY salary DESC
   ) AS rk
   ```

3. `DENSE_RANK()`：为结果集中的每一行分配排名。排序值相同的行会获得相同排名，但后续排名**不会跳号**。

   ```sql
   DENSE_RANK() OVER (
       PARTITION BY dept
       ORDER BY salary DESC
   ) AS dr
   ```

4. `NTILE(n)`：将窗口内的数据按照 `ORDER BY` 后的结果尽可能平均地划分成 `n` 个桶，并为每行分配桶号 `1 ~ n`。例如，想取排名靠前的约 25%，可以将数据分成 4 个桶，取第 1 个桶：
   ```sql
   SELECT *,
        NTILE(4) OVER (
            ORDER BY score DESC
        ) AS bucket
    FROM student
   ```

   _`NTILE()` 是按照行数尽量均匀分桶，如果不能均分，较小编号的桶通常会多分一行。_

5. `PERCENT_RANK()`：计算当前行在窗口中的**百分比排名**，返回值范围为 `0 ~ 1`，表示当前行在整个排序结果中的相对位置。其计算思想可以理解为：`(rank - 1) / (总行数 - 1)`

   ```sql
   PERCENT_RANK() OVER (
       ORDER BY score DESC
   ) AS pct_rank
   ```


#### 窗口函数的基本结构

```sql
窗口函数() OVER (
    PARTITION BY ...
    ORDER BY ...
    ROWS / RANGE BETWEEN ... AND ...
)
```

1. `OVER()`：使用窗口计算（空`OVER()`表示把整个数据集作为一个窗口）。
    ```sql
    SUM(amount) OVER (
        PARTITION BY user_id
    )
    ```
    与普通 `GROUP BY` 不同，窗口函数**不会把多行合并成一行**。

    例如原数据：

    ```text
    user_id   amount
    A         100
    A         200
    A         300
    ```

    使用：

    ```sql
    SUM(amount) OVER (
        PARTITION BY user_id
    )
    ```

    得到：

    ```text
    user_id   amount   total_amount
    A         100      600
    A         200      600
    A         300      600
    ```

   但使用：

   ```sql
   SELECT user_id, SUM(amount) AS total_amount
   FROM table
   GROUP BY user_id
   ```  

   得到：

    ```text
    user_id   total_amount
    A         600
    ```

2. `PARTITION BY`：用于把数据划分成多个独立的分区，每个分区单独计算。

    例如，先按照部门分组，每个部门内部再按照工资从高到低排名。
    ```sql
    ROW_NUMBER() OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    )
    ```

    如果不写 `PARTITION BY`，则表示所有数据放在同一个窗口中进行全局排名。
    ```sql
    ROW_NUMBER() OVER (
        ORDER BY salary DESC
    )
    ```

3. `ORDER BY`：规定分区内部的数据顺序。
    ```sql
    ROW_NUMBER() OVER (
        PARTITION BY dept
        ORDER BY salary DESC
    )
    ```

    像下面这些函数通常都非常依赖 `ORDER BY`：

    ```sql
    ROW_NUMBER()
    RANK()
    DENSE_RANK()
    NTILE()
    PERCENT_RANK()
    LAG()
    LEAD()
    ```

#### 窗口帧（Window Frame）

##### 定义

窗口帧是：在已经通过 `PARTITION BY` 确定分区、通过 `ORDER BY` 确定顺序后，针对**当前这一行**，进一步规定这次计算具体使用哪些行。

例如：

```sql
SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY dt
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)
```

假设当前分区中有：

```text
第1行   100
第2行   200
第3行   300
第4行   400 <- 当前行
第5行   500
```

当前在第 4 行时：

```sql
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
```

表示只取：

```text
第2行   200
第3行   300
第4行   400  <- 当前行
```

因此：

```text
SUM = 200 + 300 + 400 = 900
```

所以可以理解为：
- PARTITION BY：定义分区，即“大范围”
- WINDOW FRAME：定义窗口帧，即“小范围”


##### 常见窗口帧关键字

* `ROWS` 按照**实际行的位置**计算：例如 `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` 表示当前行和前面的 1 行。
* `RANGE` 按照**排序值的范围**计算：例如 `RANGE BETWEEN 1 PRECEDING AND CURRENT ROW` 表示当前行和排序值相同或前面的行。

* `UNBOUNDED PRECEDING`：从当前分区的第一行开始。
* `n PRECEDING`：当前行之前的 n 行。
* `CURRENT ROW`：当前行。
* `n FOLLOWING`：当前行之后的 n 行。
* `UNBOUNDED FOLLOWING`：一直到当前分区最后一行。

```sql
SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) 
```



#### 窗口函数和窗口帧的关系

并不是所有窗口函数都需要显式指定窗口帧!

##### 不需要窗口帧

排名类函数以及前后行定位函数通常不需要显式指定 `ROWS` 或 `RANGE`，常见函数包括：

```sql
ROW_NUMBER()
RANK()
DENSE_RANK()
NTILE()
PERCENT_RANK()
LAG()
LEAD()
```

- `ROW_NUMBER()`、`RANK()`、`DENSE_RANK()`、`NTILE()`、`PERCENT_RANK()` 主要关心**当前行在整个分区**按照 `ORDER BY` 排序之后所处的位置，因此通常只需要 `PARTITION BY + ORDER BY`：
    ```sql
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY dt
    ) AS rn
    ```

- `LAG()` / `LEAD()` 用于直接获取当前行前面或后面的某一行，本身已经通过参数指定了偏移量，因此通常也不需要窗口帧。例如：
    ```sql
    LAG(price, 1) OVER (
        PARTITION BY sku_id
        ORDER BY dt
    ) AS last_price
    ```

> 可以简单理解为：`ROW_NUMBER()` / `RANK()` 等函数用于判断“当前行排第几”，`LAG()` / `LEAD()` 用于寻找“当前行前面或后面的某一行”，它们都不是对一段窗口数据进行聚合，因此通常不需要 `ROWS / RANGE`。

##### 需要窗口帧

窗口帧主要用于需要对“一段数据范围”进行计算的窗口函数，常见函数包括：

```sql
SUM()
AVG()
COUNT()
MIN()
MAX()
FIRST_VALUE()
LAST_VALUE()
```

`SUM()`、`AVG()`、`COUNT()`、`MIN()`、`MAX()` 这类聚合函数需要根据业务需求确定当前行计算时使用哪些行。例如，计算每个用户按照日期的累计消费金额（`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 表示从当前分区的第一行一直计算到当前行，因此可以实现累计求和）：

```sql
SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS total_amount
```

`FIRST_VALUE()` / `LAST_VALUE()` 也需要特别注意窗口帧，因为它们获取的是当前窗口帧中的第一个值或最后一个值，而不一定是整个 `PARTITION` 的第一个值或最后一个值。尤其是 `LAST_VALUE()`，如果希望获取整个分区真正的最后一个值，可以显式指定完整窗口帧：

```sql
LAST_VALUE(price) OVER (
    PARTITION BY sku_id
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS last_price
```

### 全局TopN问题

#### 定义
全局TopN问题是指在**所有数据**中找出排名前 N 的记录，通常用于需要获取整体排名靠前的数据场景。

#### 答案

`ORDER BY + LIMIT N` 是最常见的全局TopN问题解决方案：

```sql
SELECT student_id, student_name, math_score
FROM student_achievement_info
ORDER BY math_score DESC
LIMIT 10
```

#### 执行
底层执行引擎一般会尝试做优化，但优化效果会受到执行引擎版本、参数配置、limit 大小、SQL 复杂度和数据量的影响。

Hive、Spark 等执行引擎一般会尝试做 TopN 优化，例如先在局部阶段取每个分区的 TopN，再将局部候选结果汇总，最后计算全局 TopN。这样可以显著减少最终排序阶段的数据量。



### 分组TopN问题

#### 定义
分组TopN问题是指在**每个分组**中找出排名前 N 的记录，通常用于需要获取各个组内排名靠前的数据场景。

#### 解法

子查询 + 排序开窗函数 + where过滤

1. **TopN**：求每个班级中，数学成绩排名前10的学生信息：

    ```sql
    SELECT year, class_name, student_id, student_name, math_score
    FROM (
        SELECT year,  class_name, student_id, 
            student_name, 
            math_score, 
            ROW_NUMBER() OVER (
                    PARTITION BY year, class_name 
                    ORDER BY math_score DESC
            ) AS rk
        FROM student_achievement_info
    ) as t
    WHERE rk <= 10
    ```

2. **百分比**：求每个班级中，数学成绩排名前30%的学生信息
    ```sql
    SELECT year, class_name, student_id, student_name, math_score
    FROM (
        SELECT year, class_name, student_id,
            student_name, 
            math_score, 
            PERCENT_RANK() OVER (
                    PARTITION BY year, class_name 
                    ORDER BY math_score DESC
            ) AS prk
        FROM student_achievement_info
    ) as t
    WHERE prk <= 0.30
    ```

3. **分组与整体/个体**：空`OVER()`指定全数据整体直接取值，其他情况使用`GROUP BY`分组算出个体的值，然后结合窗口函数或子查询进行进一步处理

    - 求每个班级中，数学成绩排名前10的学生信息，以及这些学生和全校第一名的数学成绩差距
        > 使用窗口函数 `MAX() OVER ()` 获取全数据最高分，并结合 `ROW_NUMBER()` 过滤每个班级排名前10的学生

        ```sql
        SELECT student_id, student_name, math_score, class_name, (max_math_score - math_score) AS score_diff
        FROM (
            SELECT student_id, 
                student_name, 
                math_score, 
                class_name,
                ROW_NUMBER() OVER (
                        PARTITION BY class_name 
                        ORDER BY math_score DESC
                ) AS rk,
                MAX(math_score) OVER () AS max_math_score
            FROM student_achievement_info
        ) as t
        WHERE rk <= 10
        ```
    
    - 在学生每科成绩得分表中，选出每年，每个年级的总得分第一名和他的成绩
        > `GROUP BY` 求每个人总分，`ROW_NUMBER()` 结合 `rn = 1` 过滤

        ```sql
        SELECT year,
            class,
            name,
            total_score
        FROM (
            SELECT year,
                class,
                name,
                total_score,
                ROW_NUMBER() OVER (
                    PARTITION BY year, class
                    ORDER BY total_score DESC
                ) AS rn
            FROM (
                SELECT year,
                    class,
                    name,
                    SUM(score) AS total_score
                FROM ods_game_dev.topn_scores
                GROUP BY year, class, name
            ) t1
        ) t2
        WHERE rn = 1
        ```

4. **Top1**: 求平均分Top1的班级名称
    - `ROW_NUMBER()` 结合 `rn = 1` 过滤
        ```sql
        SELECT class
        FROM (
            SELECT class,
                AVG(math_score) AS avg_score,
                ROW_NUMBER() OVER (
                    ORDER BY AVG(math_score) DESC
                ) AS rn
            FROM score_list
            GROUP BY class
        ) as t
        WHERE rn = 1
        ```
    - `FIRST_VALUE()` 结合窗口函数
        ```sql
        SELECT class,
            AVG(math_score) AS avg_score,
            FIRST_VALUE(class) OVER (
                ORDER BY AVG(math_score) DESC
            ) AS top_class
        FROM score_list
        GROUP BY class
        ```

5. **最高最低**：在学生成绩得分表中，查询每一科目成绩最高和最低分数的学生

    ```sql
    SELECT subject, name, score
    FROM (
        SELECT subject,
            name,
            score,
            score_rk_desc,
            score_rk_asc
        FROM (
            SELECT subject,
                name,
                score,
                ROW_NUMBER() OVER (
                    PARTITION BY subject
                    ORDER BY score DESC
                ) AS score_rk_desc,
                ROW_NUMBER() OVER (
                    PARTITION BY subject
                    ORDER BY score ASC
                ) AS score_rk_asc
            FROM ods_game_dev.topn_scores
        ) t1
        WHERE score_rk_desc = 1
        OR score_rk_asc = 1
    ) t2;
    ```

---
## 3. 连续登陆问题

### 连续登陆问题概述

连续登录问题的核心在于日期连续，一般题目中出现 "求XXX连续N天登录" 这种字眼时，往往就是一道连续登陆日期的题目。

通常需要注意以下三点：

- 日期必须连续（比如12.5、12.6、12.7 登录，就是连续 3 天）

- 每天只保留一条登录记录（比如 12.5上午、12.5 下午、12.6 登录，只能算连续 2 天）

- 不同连续区间需要分组
   * 如：12.1、12.2、12.3、12.5、12.6、12.7
   * 应拆成两组：`12.1~12.3`、`12.5~12.7`
   * 即 **两次连续 3 天**，而不是连续 6 天。


### 常用函数

#### `lag()`

获取当前行的**上一条记录**。

```sql
lag(dt, 1) over (
    partition by user_id
    order by dt
)
```

含义：

* `partition by user_id`：每个用户单独计算
* `order by dt`：按登录日期排序
* `lag(dt, 1)`：获取上一次登录日期

#### `datediff()`

计算两个日期之间相差的天数。例如：`datediff('2024-12-06', '2024-12-05') = 1`

```sql
datediff(dt, last_dt)
```

因此：

```text
datediff = 1 -> 日期连续
datediff > 1 -> 日期中断
```

#### `date_sub()`

将日期向前减指定天数。例如：`12-01 - 1天 = 11-30`

```sql
date_sub(dt, rn)
```

连续日期的 `dt` 和 `rn` 都同时 `+1`，所以：**连续的一组日期，其 `date_sub(dt, rn)` 结果相同。**

因此可以用它构造连续区间的分组标识 `sub_dt`。

### 常规解法

```sql
SELECT
    user_id,
    sub_dt,
    COUNT(*) AS continuous_days
FROM (
    SELECT 
        user_id,
        dt,
        DATE_SUB(
            dt,
            ROW_NUMBER() OVER (
                PARTITION BY user_id
                ORDER BY dt
            )
        ) AS sub_dt
    FROM (
        SELECT DISTINCT
            user_id,
            dt
        FROM user_login_table
    ) t0
) t1
GROUP BY user_id, sub_dt
HAVING COUNT(*) >= 3;
```

1. **去重**：连续登录统计的是**天数**，同一天登录多次只能算一天。因此先保证：一个 user_id + dt 只有一条记录

```sql
SELECT DISTINCT
    user_id,
    dt
FROM user_login_table
```

2. `ROW_NUMBER()` **开窗排序**：对每个用户的登录日期排序：

```sql
ROW_NUMBER() OVER (
    PARTITION BY user_id
    ORDER BY dt
) AS rn
```

例如：

| user_id  |   dt    | rn |
| ---------| ------- | -: |
| 1 | 12-01 |  1 |
| 1 | 12-02 |  2 |
| 1 | 12-03 |  3 |
| 1 | 12-05 |  4 |
| 1 | 12-06 |  5 |


3. 构造日期差标识

```sql
DATE_SUB(dt, rn) AS sub_dt
```

得到：

| user_id | dt    | rn | sub_dt |
|---------|-------|----|--------|
| 1 | 12-01 |  1 | 11-30  |
| 1 | 12-02 |  2 | 11-30  |
| 1 | 12-03 |  3 | 11-30  |
| 1 | 12-05 |  4 | 12-01  |
| 1 | 12-06 |  5 | 12-01  |


4. `GROUP BY` 聚合统计

```sql
GROUP BY user_id, sub_dt
```

分组：

```sql
SELECT
    user_id,
    sub_dt,
    count(*) as continuous_days
FROM t
GROUP BY
    user_id,
    sub_dt;
```

每个分组就是一段连续登录区间，`count(*)` 即该段的**连续登录天数**。


### 其他解法

```sql
SELECT
    user_id,
    group_id,
    COUNT(*) AS continuous_days
FROM (
    SELECT
        user_id,
        dt,
        SUM(tmp_lab) OVER (
            PARTITION BY user_id
            ORDER BY dt
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS group_id
    FROM (
        SELECT
            user_id,
            dt,
            CASE
                WHEN login_date_diff = 1 THEN 0
                ELSE 1
            END AS tmp_lab
        FROM (
            SELECT
                user_id,
                dt,
                DATEDIFF(
                    dt,
                    LAG(dt, 1) OVER (
                        PARTITION BY user_id
                        ORDER BY dt
                    )
                ) AS login_date_diff
            FROM (
                SELECT DISTINCT
                    user_id,
                    dt
                FROM user_login_table
            ) t0
        ) t1
    ) t2
) t3
GROUP BY user_id, group_id
HAVING COUNT(*) >= 3;
```

1. **去重**：同一天登录多次只能算一天，因此先保证一个 `user_id + dt` 只有一条记录。

```sql
SELECT DISTINCT
    user_id,
    dt
FROM user_login_table
```

2. `LAG()` + `DATEDIFF()` **计算相邻登录日期差**：先通过 `LAG()` 获取用户上一次登录日期，再通过 `DATEDIFF()` 计算当前日期与上一次登录日期相差多少天。

```sql
DATEDIFF(
    dt,
    LAG(dt, 1) OVER (
        PARTITION BY user_id
        ORDER BY dt
    )
) AS login_date_diff
```

例如：

| user_id | dt    | last_dt | login_date_diff |
| ------- | ----- | ------- | --------------: |
| 1       | 12-01 | NULL    |            NULL |
| 1       | 12-02 | 12-01   |               1 |
| 1       | 12-03 | 12-02   |               1 |
| 1       | 12-05 | 12-03   |               2 |
| 1       | 12-06 | 12-05   |               1 |

其中：

```text
login_date_diff = 1 -> 和上一条记录连续
login_date_diff != 1 -> 连续中断
```

3. **构造中断标识**：通过 `CASE WHEN` 判断当前记录是否开启了一段新的连续登录区间。

```sql
CASE
    WHEN login_date_diff = 1 THEN 0
    ELSE 1
END AS tmp_lab
```

得到：

| user_id | dt    | login_date_diff | tmp_lab |
| ------- | ----- | --------------: | ------: |
| 1       | 12-01 |            NULL |       1 |
| 1       | 12-02 |               1 |       0 |
| 1       | 12-03 |               1 |       0 |
| 1       | 12-05 |               2 |       1 |
| 1       | 12-06 |               1 |       0 |

其中：

```text
tmp_lab = 0 -> 延续上一段
tmp_lab = 1 -> 开启新的一段
```

4. `SUM()` **累计构造分组标识**：对 `tmp_lab` 进行累计求和，每遇到一次中断，分组编号就 `+1`。

```sql
SUM(tmp_lab) OVER (
    PARTITION BY user_id
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS group_id
```

得到：

| user_id | dt    | tmp_lab | group_id |
| ------- | ----- | ------: | -------: |
| 1       | 12-01 |       1 |        1 |
| 1       | 12-02 |       0 |        1 |
| 1       | 12-03 |       0 |        1 |
| 1       | 12-05 |       1 |        2 |
| 1       | 12-06 |       0 |        2 |

因此：

```text
group_id = 1 -> 12-01 ~ 12-03
group_id = 2 -> 12-05 ~ 12-06
```

5. `GROUP BY` **聚合统计**

```sql
GROUP BY user_id, group_id
HAVING COUNT(*) >= 3
```

每个 `user_id + group_id` 就是一段连续登录区间，`COUNT(*)` 即该段的**连续登录天数**。

### 间隔连续登陆（其他解法plus）

#### 问题描述

设置一个间隔天数 `k`，如果用户在 `k` 天内登录，则视为连续登录（例子：假设用户在上次登录后2天内登录就视作连续登陆，一个用户在 12.1，12.3，12.5，12.6 共4天都登录了游戏，则视为连续 6 天登录）。

#### 解决方案

```sql
SELECT
    user_id,
    sum_tmp_lab,
    DATEDIFF(MAX(dt), MIN(dt)) + 1 AS continuous_days
FROM (
    SELECT
        user_id,
        dt,
        SUM(tmp_lab) OVER (
            PARTITION BY user_id
            ORDER BY dt
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS sum_tmp_lab
    FROM (
        SELECT
            user_id,
            dt,
            CASE
                WHEN login_date_diff <= 2 THEN 0
                ELSE 1
            END AS tmp_lab
        FROM (
            SELECT
                user_id,
                dt,
                DATEDIFF(
                    dt,
                    LAG(dt, 1) OVER (
                        PARTITION BY user_id
                        ORDER BY dt
                    )
                ) AS login_date_diff
            FROM (
                SELECT DISTINCT
                    user_id,
                    dt
                FROM user_login_table
            ) t0
        ) t1
    ) t2
) t3
GROUP BY user_id, sum_tmp_lab
HAVING DATEDIFF(MAX(dt), MIN(dt)) + 1 >= 6;
```

1. **去重**：保证一个用户一天只有一条登录记录。

```sql
SELECT DISTINCT
    user_id,
    dt
FROM user_login_table
```

2. `LAG() + DATEDIFF()` **计算相邻两次登录的日期差**。

```sql
DATEDIFF(
    dt,
    LAG(dt, 1) OVER (
        PARTITION BY user_id
        ORDER BY dt
    )
) AS login_date_diff
```

例如：

| dt    | 上次登录  | login_date_diff |
| ----- | ----- | --------------: |
| 12-01 | NULL  |            NULL |
| 12-03 | 12-01 |               2 |
| 12-05 | 12-03 |               2 |
| 12-06 | 12-05 |               1 |

3. `CASE WHEN` **判断是否超过允许的间隔**。

普通连续登录：

```sql
CASE
    WHEN login_date_diff = 1 THEN 0
    ELSE 1
END
```

允许间隔一天：

```sql
CASE
    WHEN login_date_diff <= 2 THEN 0
    ELSE 1
END AS tmp_lab
```

也就是：

```text
日期差 <= 2 -> 仍然连续 -> 0
日期差 > 2  -> 连续中断 -> 1
```

4. `SUM()` **累计生成连续区间编号**。

```sql
SUM(tmp_lab) OVER (
    PARTITION BY user_id
    ORDER BY dt
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS sum_tmp_lab
```

例如：

| dt    | login_date_diff | tmp_lab | sum_tmp_lab |
| ----- | --------------: | ------: | ----------: |
| 12-01 |            NULL |       1 |           1 |
| 12-03 |               2 |       0 |           1 |
| 12-05 |               2 |       0 |           1 |
| 12-06 |               1 |       0 |           1 |

因此这 4 条记录属于同一个连续区间。

5. `GROUP BY` **聚合计算连续天数**。

```sql
GROUP BY user_id, sum_tmp_lab
```

普通连续登录可以直接：

```sql
COUNT(*)
```

但是间隔连续登录**不能使用 `COUNT(*)`**。

因为：

```text
登录日期：12-01、12-03、12-05、12-06

COUNT(*) = 4
```

但题目认为这一段其实算作连续登陆6天；所以需要计算日期跨度（12-06 - 12-01 + 1 = 6天）：

```sql
DATEDIFF(MAX(dt), MIN(dt)) + 1
```

因此间隔连续登录的核心区别是：

```text
普通连续登录：
login_date_diff = 1
COUNT(*)

间隔一天也连续：
login_date_diff <= 2
DATEDIFF(MAX(dt), MIN(dt)) + 1
```

### 题型分类

TBD



## 4. 行转列/列转行问题

### 行转列问题概述

行转列和列转行的核心都是改变数据的组织形式 / 粒度。 **行转列**通常是把原来分散在多行中的同一主体数据进行**聚合**，整理到更少的行、更多的列或一个聚合字段中；**列转行**则相反，把原来保存在一行中的一个或多个字段**拆开**，生成多条明细记录。

> 行转列通常涉及 GROUP BY + 聚合操作，而列转行通常涉及把字段“炸开”的操作。

### 常用函数（列转行）

#### 1. `explode()`

`explode()` 是最常用的**炸裂函数 / UDTF（User-Defined Table-Generating Function，表生成函数）**之一。UDTF 的特点是可以让原表中的**一行数据生成多行数据**。对于 `ARRAY`，`explode()` 会把数组中的每个元素展开成一行；对于 `MAP`，会把每个键值对展开成 `key + value` 两列。`posexplode()`和 `explode(array)` 类似，但会额外返回数组元素的位置索引，索引从 `0` 开始。

**例子**

| id | name | hobbies                    |
| -: | ---- | -------------------------- |
|  1 | Amy  | `['game','swim']`          |
|  2 | Bob  | `['run']`                  |
|  3 | Jack | `['movie','game','music']` |

- `explode(ARRAY)`：如果希望把“一个人的多个兴趣”拆成真正的明细记录，可以对 `hobbies` 使用 `explode()`：

    ```sql
    SELECT
        id,
        name,
        hobby
    FROM person
    LATERAL VIEW explode(hobbies) tmp AS hobby;
    ```

    结果：

    | id | name | hobby |
    | -: | ---- | ----- |
    |  1 | Amy  | game  |
    |  1 | Amy  | swim  |
    |  2 | Bob  | run   |
    |  3 | Jack | movie |
    |  3 | Jack | game  |
    |  3 | Jack | music |

    这里真正负责产生多行的是 `explode(hobbies)`：Amy 原本只有 **1 行**，但 `hobbies` 中有 2 个元素，因此炸裂后变成 **2 行**；Jack 的数组有 3 个元素，因此变成 **3 行**。`LATERAL VIEW` 只是把这些炸出来的 `hobby` 重新和当前 `person` 行的 `id / name` 组合起来，具体作用后面单独讲。

     `explode()` 并不是为了“做一道列转行题”，而是为了**降低数据粒度，把集合中的元素变成可以继续 `WHERE / JOIN / GROUP BY` 的明细记录**。用户标签、商品列表、日志事件数组、角色列表、设备列表等都属于同一种处理思想。

- `explode(MAP)`：MAP（数据类型）中的一个元素本身包含 `key + value`，所以会一次生成两列。(`SELECT EXPLODE(MAP("a", 1, "b", 2, "c", 3)) as (key, value);`)

    | user_id | name | attrs                  |
    | ------: | ---- | ---------------------- |
    |       1 | Amy  | `{'level':10,'vip':1}` |
    |       2 | Bob  | `{'level':6,'vip':0}`  |

    执行：

    ```sql
    SELECT
        user_id,
        name,
        attr_name,
        attr_value
    FROM user_info
    LATERAL VIEW explode(attrs) tmp AS attr_name, attr_value;
    ```

    结果：

    | user_id | name | attr_name | attr_value |
    | ------: | ---- | --------- | ---------: |
    |       1 | Amy  | level     |         10 |
    |       1 | Amy  | vip       |          1 |
    |       2 | Bob  | level     |          6 |
    |       2 | Bob  | vip       |          0 |

    所以数组和 MAP 的区别可以直接记成：**`explode(ARRAY)` 每个元素产生 1 个输出字段；`explode(MAP)` 每个键值对产生 2 个输出字段，即 `key + value`。** 原笔记中的示例也是 `explode(array())` 返回一列，而 `explode(map())` 返回两列。

- `posexplode()`：`explode()` + position，除了把数组元素拆成多行，还保留元素原本在数组中的索引，索引从 `0` 开始。

    | user_id | events                    |
    | ------: | ------------------------- |
    |       1 | `['login','click','pay']` |
    |       2 | `['login','logout']`      |

    执行：

    ```sql
    SELECT
        user_id,
        pos,
        event
    FROM user_event
    LATERAL VIEW posexplode(events) tmp AS pos, event;
    ```

    结果：

    | user_id | pos | event  |
    | ------: | --: | ------ |
    |       1 |   0 | login  |
    |       1 |   1 | click  |
    |       1 |   2 | pay    |
    |       2 |   0 | login  |
    |       2 |   1 | logout |

    如果只用 `explode(events)`，最终只能知道用户 1 有 `login / click / pay` 三个元素；使用 `posexplode(events)` 后，还可以知道 `login` 原来是第 0 个元素、`click` 是第 1 个、`pay` 是第 2 个。因此只关心“元素是什么”时用 `explode()`，还需要保留**数组原始位置**时使用 `posexplode()`。这与原笔记中 `posexplode(array())` 同时返回 `pos + item` 的定义一致。

> 需要特别注意**数据膨胀**：如果原表有 100 万行，每行数组平均有 20 个元素，炸裂后理论上可能产生约 2000 万行，因此炸裂之后再做 `JOIN / GROUP BY / DISTINCT` 可能明显增加计算量，实际写 SQL 时通常应该先完成可以提前做的过滤。

> 另一个注意点是空数组或 `NULL`：普通炸裂没有元素可输出时，原记录可能不会保留；如果业务要求即使数组为空也保留原记录，Hive / Spark SQL 中通常使用 `LATERAL VIEW OUTER explode(...)`。

```sql
SELECT
    id,
    name,
    hobby
FROM person
LATERAL VIEW OUTER EXPLODE(hobbies) tmp AS hobby;
```

#### `LATERAL VIEW`

`LATERAL VIEW` **不是函数，而是配合 UDTF 使用的一种 SQL 语法**。它会把 UDTF 应用到源表的每一行，让当前行生成一行或多行结果，然后把这些结果重新和当前源表行组合起来形成一个虚表。原笔记中的表述就是：将 UDTF 应用到源表的**每行数据**，把每行转换成一行或多行，并将输出结果与该源行连接起来。 因此 `explode()` 和 `LATERAL VIEW` 要分开理解：**`explode()` 负责把什么炸出来，`LATERAL VIEW` 负责把炸出的结果和原表哪一行对应起来。**

- 基本结构：

    ```sql
    FROM 原表
    LATERAL VIEW explode(数组字段) 虚表别名 AS 输出列名
    ```

    假设原表 `person`：

    | id | name | hobbies           |
    | -: | ---- | ----------------- |
    |  1 | Amy  | `['game','swim']` |
    |  2 | Bob  | `['run']`         |

    执行：

    ```sql
    SELECT
        id,
        name,
        hobby
    FROM person
    LATERAL VIEW EXPLODE(hobbies) tmp AS hobby;
    ```

    得到：

    | id | name | hobby |
    | -: | ---- | ----- |
    |  1 | Amy  | game  |
    |  1 | Amy  | swim  |
    |  2 | Bob  | run   |

    这里 `explode(hobbies)` 产生炸裂结果，`tmp` 是炸裂结果形成的**虚表别名**，`hobby` 是这个虚表输出字段的**列名**。这和原笔记中的解释一致。 分析时最好把 `FROM person LATERAL VIEW explode(hobbies) tmp AS hobby` 当成一个整体理解：对于 `person` 的当前一行，把 `hobbies` 炸开，然后将炸出来的每一个元素都复制并关联回当前的 `person` 行，所以 Amy 原本的一行最终变成两行，但 `id`、`name` 等原始信息仍然保留。

- 如果输入是 `MAP`，因为 `explode(map)` 本身产生两列，因此 `AS` 后也要给两个输出字段命名：

    ```sql
    SELECT
        id,
        attr_name,
        attr_value
    FROM user_info
    LATERAL VIEW explode(attrs) tmp AS attr_name, attr_value;
    ```

    假设原表：

    | id | attrs                  |
    | -: | ---------------------- |
    |  1 | `{'level':10,'vip':1}` |

    结果：

    | id | attr_name | attr_value |
    | -: | --------- | ---------: |
    |  1 | level     |         10 |
    |  1 | vip       |          1 |


- 还需要特别注意多个 `LATERAL VIEW` 的**乘法效应**。

    | id | arr1            | arr2    |
    | -: | --------------- | ------- |
    |  1 | `['A','B','C']` | `[1,2]` |

    如果连续独立炸裂两个数组：

    ```sql
    FROM t
    LATERAL VIEW EXPLODE(arr1) a AS x
    LATERAL VIEW EXPLODE(arr2) b AS y
    ```

    结果可能是：

    | id | x |  y |
    | -: | - | -: |
    |  1 | A |  1 |
    |  1 | A |  2 |
    |  1 | B |  1 |
    |  1 | B |  2 |
    |  1 | C |  1 |
    |  1 | C |  2 |

    也就是 `3 × 2 = 6` 行。因此多个数组需要先判断业务关系是“所有组合”还是“按照位置一一对应”，不能看到两个数组就直接连续炸裂。

#### 2. `SPLIT()`

`SPLIT()` 用于**按照指定分隔规则拆分字符串**，拆分后的返回结果是一个 `ARRAY` 数组。在列转行问题中，它经常负责先完成 `STRING → ARRAY`，再把数组交给 `explode()` 展开成多行。

##### 语法

```sql
SPLIT(str, pattern)
```

* `str`：需要拆分的字符串。
* `pattern`：分隔规则（在 Hive 中这里是正则表达式 pattern，可以是简单字符，也可以是更复杂的正则表达式）。
* **返回值**：`ARRAY<STRING>`，即字符串数组。

例如原表：

| id | category              |
| -: | --------------------- |
|  1 | `Action,Comedy,Drama` |
|  2 | `Comedy,Romance`      |

执行：

```sql
SELECT
    id,
    split(category, ',') AS category_array
FROM movie;
```

结果可以理解为：

| id | category_array                |
| -: | ----------------------------- |
|  1 | `['Action','Comedy','Drama']` |
|  2 | `['Comedy','Romance']`        |


### 常用函数（行转列）

#### 1. `collect_list()`

`collect_list(expr)` 是一个**聚合函数**（多行聚合），作用是把一个分组中的多行 `expr` 收集起来，返回一个 `ARRAY`，而且**保留重复元素**。

例如订单明细表：

| user_id | product |
| ------: | ------- |
|       1 | A       |
|       1 | B       |
|       1 | A       |
|       2 | C       |
|       2 | D       |

执行：

```sql
SELECT
    user_id,
    collect_list(product) AS products
FROM orders
GROUP BY user_id;
```

逻辑结果：

| user_id | products        |
| ------: | --------------- |
|       1 | `['A','B','A']` |
|       2 | `['C','D']`     |

可以看到，用户 1 的商品 A 出现两次，`collect_list()` 会把这两次都保留下来。因此如果重复次数具有业务含义，比如一个用户真实发生了两次 `click`、一个商品被购买了三次、一个订单出现了多条同 SKU 记录，就应该使用 `collect_list()`，而不是自动去重。

> 这类数组之后还可以继续交给 `size()` 统计长度、`array_contains()` 判断是否出现某个元素、`transform()` 做元素转换、`explode()` 再拆回明细，因此 `collect_list()` 更准确的理解是：**把一个分组内的明细保存为数组结构，供后续复杂数据处理。**

> 不要依赖 `collect_list()` 默认的元素顺序。该函数是非确定性的，因为结果顺序取决于输入行顺序，而经过 shuffle 后行顺序可能发生变化。 可以进一步写 `sort_array(collect_list(score))`来得到稳定的顺序。



#### 2. `collect_set()`

`collect_set(expr)` 同样是**聚合函数**，也是把一个分组中的多行数据收集成一个 `ARRAY`，但它会**自动去重，只保留唯一元素**。原笔记中的定义是“收集并返回一个唯一元素的集合”。 因此选择 `collect_list()` 还是 `collect_set()`，关键不在于哪个函数更好，而在于**重复值有没有业务意义**。

例如：

| user_id | product |
| ------: | ------- |
|       1 | A       |
|       1 | B       |
|       1 | A       |
|       1 | C       |

分别使用：

```sql
SELECT
    user_id,
    collect_list(product) AS product_list,
    collect_set(product) AS product_set
FROM orders
GROUP BY user_id;
```

逻辑上可以理解为：

| user_id | product_list        | product_set     |
| ------: | ------------------- | --------------- |
|       1 | `['A','B','A','C']` | `['A','B','C']` |

所以如果问题是“用户所有购买记录”，两次 A 都应该保留，适合 `collect_list()`；如果问题是“用户买过哪些不同商品”，只关心 A 是否出现过，适合 `collect_set()`。同样地，用户行为 `login、click、click、pay` 中，“完整行为明细”使用 `collect_list(event)`，“出现过哪些不同事件类型”使用 `collect_set(event)`。

`collect_set()` 同样不能依赖结果顺序。如果需要按照数组元素自身稳定排序，一般继续使用`sort_array(collect_set(product))`。



#### 3. `sort_array()`

`sort_array(array[, ascendingOrder])` 是**数组排序函数**：输入一个数组，根据数组元素本身的自然顺序排序，最后仍然返回一个数组。原笔记基于 Spark 3.4.4 的说明是：该函数既支持升序也支持降序；对于 `double / float`，`NaN` 大于任何非 `NaN` 元素；升序时 `NULL` 放在数组开头，降序时 `NULL` 放在数组末尾。

例如原表：

| user_id | scores        |
| ------: | ------------- |
|       1 | `[90,70,85]`  |
|       2 | `[60,100,80]` |

执行：

```sql
SELECT
    user_id,
    sort_array(scores, true) AS score_asc,
    sort_array(scores, false) AS score_desc
FROM user_score;
```

结果：

| user_id | score_asc     | score_desc    |
| ------: | ------------- | ------------- |
|       1 | `[70,85,90]`  | `[90,85,70]`  |
|       2 | `[60,80,100]` | `[100,80,60]` |

所以 `sort_array()` 的数据粒度完全没有变化：仍然是一行、仍然是一个数组，只是数组内部元素的排列顺序发生变化。它完全不局限于行转列，任何已经存在的数组字段都可以直接排序。

最常见的组合之一是配合 `collect_list()` / `collect_set()`。

> `sort_array()` 只能根据数组元素自己排序，而不能像普通 `ORDER BY` 一样任意指定另一个字段。


#### 4. `concat_ws()`

`concat_ws(sep[, str | array(str)]+)` 是**字符串拼接函数**，作用是使用指定的 `sep` 作为分隔符，把多个字符串或一个字符串数组拼成最终的字符串；原笔记还特别指出，它在拼接时会跳过 `NULL`。 `ws` 可以直接理解为 **with separator**，所以它和 `concat()` 最大区别就是可以指定分隔符。

- 普通字符串字段：

    | year | month | day |
    | ---: | ----: | --: |
    | 2026 |    08 |  30 |

    执行：

    ```sql
    SELECT concat_ws('-', year, month, day) AS dt
    FROM t
    ```

    结果：

    | dt           |
    | ------------ |
    | `2026-08-30` |

- 处理字符串数组：

    | user_id | tags                      |
    | ------: | ------------------------- |
    |       1 | `['vip','game','active']` |
    |       2 | `['new','mobile']`        |

    执行：

    ```sql
    SELECT
        user_id,
        concat_ws(',', tags) AS tag_string
    FROM user_tag;
    ```

    结果：

    | user_id | tag_string        |
    | ------: | ----------------- |
    |       1 | `vip,game,active` |
    |       2 | `new,mobile`      |

#### 5. `CASE`

`CASE` 是 SQL 中的**条件表达式**，作用是根据不同条件返回不同的值，可以理解成 `if / else if / else`。它本身不是聚合函数，也不只用于行转列；整个 `CASE ... END` 最终会计算成**一个值**，因此可以和 `SELECT`、聚合函数、`ORDER BY` 等组合使用。

##### 语法

`CASE` 主要有两种写法。

* **搜索型 `CASE WHEN condition`**：适合范围判断、多字段判断、复杂条件。

    ```sql
    CASE
        WHEN condition1 THEN result1
        WHEN condition2 THEN result2
        ...
        ELSE default_result
    END
    ```

    例如：

    ```sql
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 60 THEN 'C'
        ELSE 'D'
    END
    ```

    `WHEN` 会**从上到下依次判断**，第一个满足条件的分支就是最终结果，所以条件顺序很重要。`WHEN` 后可以使用 `=、>、<、BETWEEN、IN、IS NULL、AND、OR` 等条件。

* **简单型 `CASE col WHEN value`**：适合同一个字段和多个固定值做**等值匹配**。

    ```sql
    CASE col
        WHEN value1 THEN result1
        WHEN value2 THEN result2
        ...
        ELSE default_result
    END
    ```

    例如：

    ```sql
    CASE status
        WHEN 1 THEN '待支付'
        WHEN 2 THEN '已支付'
        WHEN 3 THEN '已完成'
        ELSE '未知状态'
    END
    ```

    它等价于：

    ```sql
    CASE
        WHEN status = 1 THEN '待支付'
        WHEN status = 2 THEN '已支付'
        WHEN status = 3 THEN '已完成'
        ELSE '未知状态'
    END
    ```

- **`ELSE` 可以省略**：如果所有 `WHEN` 都不满足，并且没有写 `ELSE`，默认返回 `NULL`。

    ```sql
    CASE
        WHEN score >= 60 THEN 'PASS'
    END
    ```

    等价于：

    ```sql
    CASE
        WHEN score >= 60 THEN 'PASS'
        ELSE NULL
    END
    ```

-  **返回值规则**：每个 `THEN` 和 `ELSE` 都返回一个值，整个 `CASE ... END` 最终也只返回一个值。各分支最好返回相同或兼容的数据类型，例如都返回数字或都返回字符串，避免隐式类型转换。

> NULL 判断：判断空值必须使用 `IS NULL / IS NOT NULL`，不能使用 `= NULL / != NULL`。简单型 `CASE col WHEN NULL` 也不能可靠匹配 NULL，因为本质仍类似 `col = NULL`。


##### 常见用法

* **分类 / 打标签**：根据字段值生成新的类别字段。

    ```sql
    CASE
        WHEN amount >= 10000 THEN 'S'
        WHEN amount >= 3000 THEN 'A'
        WHEN amount >= 500 THEN 'B'
        ELSE 'C'
    END AS user_level
    ```

* **固定值映射**：当一个字段有若干固定编码时，使用简单型 `CASE col WHEN value` 更简洁。

    ```sql
    CASE status
        WHEN 1 THEN '待支付'
        WHEN 2 THEN '已支付'
        WHEN 3 THEN '已完成'
        ELSE '未知'
    END
    ```

* **条件求和**：满足条件时保留数值，否则返回 `0`，再交给 `SUM()` 聚合。(可以理解为：`CASE WHEN` 负责决定**哪些值参与计算**，`SUM()` 负责真正求和)

    ```sql
    SUM(
        CASE
            WHEN pay_type = 'game' THEN amount
            ELSE 0
        END
    ) AS game_amount
    ```

* **条件计数**：满足条件记 `1`，否则记 `0`，最后求和。

    ```sql
    SUM(
        CASE
            WHEN status = 'paid' THEN 1
            ELSE 0
        END
    ) AS paid_cnt
    ```

    也可以利用 `COUNT(expr)` 不统计 `NULL` 的特性：

    ```sql
    COUNT(
        CASE
            WHEN status = 'paid' THEN 1
        END
    )
    ```

* **行转列 / 条件聚合**：用 `CASE WHEN` 把不同类别的数据放进不同列，再通过 `GROUP BY + SUM()/MAX()` 合并多行。

    ```sql
    SELECT
        product_id,
        SUM(CASE WHEN store = 'store1' THEN price END) AS store1,
        SUM(CASE WHEN store = 'store2' THEN price END) AS store2,
        SUM(CASE WHEN store = 'store3' THEN price END) AS store3
    FROM Products
    GROUP BY product_id;
    ```

    **`CASE WHEN` 负责“分类放位置 / 造列”，聚合函数负责“多行合成一行”。**

* **自定义排序**：当业务顺序和默认字典顺序不同，可以通过 `CASE` 人工生成排序权重。

    ```sql
    ORDER BY
        CASE status
            WHEN 'urgent' THEN 1
            WHEN 'processing' THEN 2
            WHEN 'completed' THEN 3
            ELSE 4
        END
    ```

* **复杂条件判断**：搜索型 `CASE WHEN` 可以组合多个条件。

    ```sql
    CASE
        WHEN age < 18 THEN '未成年'
        WHEN age BETWEEN 18 AND 59 THEN '成年'
        ELSE '其他'
    END
    ```

    也可以使用：

    ```sql
    WHEN city IN ('A', 'B')
    WHEN score >= 60 AND attendance >= 80
    WHEN value IS NULL
    ```

> 可出现的位置**：因为 `CASE ... END` 本质是一个表达式，所以可以出现在很多需要“值”的位置，例如 `SELECT`、`SUM/MAX/COUNT` 内部、`ORDER BY` 等。

```sql
SELECT CASE WHEN ... THEN ... END AS new_col

SUM(CASE WHEN ... THEN ... END)

ORDER BY CASE WHEN ... THEN ... END
```

#### 6. `TRANSFORM()`

`TRANSFORM()` 是一个**数组高阶函数**，用于遍历 `ARRAY` 中的每个元素，对每个元素执行指定的转换逻辑，最后返回一个**新的数组**。转换后的数组长度与原数组相同，只是数组中的元素被重新处理。

```sql
transform(array, element -> expression)
```

* `array`：需要处理的数组
* `element`：数组中的当前元素，可以自定义变量名，如 `x`
* `expression`：对当前元素执行的转换逻辑
* **返回值：新的 `ARRAY`**


##### 语法

- 基础用法

    例如原表：

    | id | scores       |
    | -: | ------------ |
    |  1 | `[10,20,30]` |
    |  2 | `[40,50]`    |

    ```sql
    SELECT
        id,
        transform(scores, x -> x + 1) AS new_scores
    FROM student;
    ```

    结果：

    | id | new_scores   |
    | -: | ------------ |
    |  1 | `[11,21,31]` |
    |  2 | `[41,51]`    |


- `transform()` 还可以同时获取**元素和下标**：

    ```sql
    transform(array, (element, index) -> expression)
    ```

    资料中的示例：

    ```sql
    transform(array(1, 2, 3), (x, i) -> x + i)
    ```

    结果：

    ```text
    [1,3,5]
    ```

    其中数组下标从 `0` 开始，因此实际计算为 `1+0、2+1、3+2`。

##### 常见用法

* **对数组元素进行计算**

    ```sql
    transform(scores, x -> x * 10)
    ```

* **对字符串数组中的元素使用字符串函数**

    ```sql
    transform(tags, x -> upper(x))
    ```

* **结合 `CASE WHEN` 进行复杂转换**

    ```sql
    transform(
        nums,
        x -> CASE
                WHEN x % 2 = 0 THEN x * 10
                ELSE x
            END
    )
    ```

* **用于有序行转列**
    
    `collect_list()` / `collect_set()` 聚合后的结果本身就是 `ARRAY`，因此可以继续交给 `transform()` 处理。资料中 `transform()` 主要用于**有序行转列**：先把排序字段与目标字段组合起来并排序，再使用 `transform()` 对排好序的数组逐个处理，去掉辅助排序字段，只留下真正需要输出的数据。


> `transform()` **只能处理 ARRAY 类型**，不能直接处理普通字符串、数字等非数组字段；`transform()` 处理后仍然返回数组，而且通常与原数组**元素数量相同**。


### 列转行解法

列转行就是把一行中的多个值拆成多行，核心思路是：**`split()` + 炸裂函数 `explode()`**。

* **如果字段是 STRING**：先用 `split()` 拆成 `ARRAY`（如果字段本身就是 ARRAY，直接 `explode()`）
* **`explode()`**：把数组中的每个元素炸成一行
* **`LATERAL VIEW`**：把炸裂后的结果和原表其他字段组合起来


常用模板：

```sql
SELECT
    原表字段,
    new_col
FROM table_name
LATERAL VIEW explode(
    split(待拆字段, '分隔符')
) tmp AS new_col;
```

例如原表：

| title   | category              |
| ------- | --------------------- |
| Movie A | `Action,Comedy,Drama` |

SQL：

```sql
SELECT
    title,
    category_new
FROM movie
LATERAL VIEW explode(split(category, ',')) tmp AS category_new;
```

结果：

| title   | category_new |
| ------- | ------------ |
| Movie A | Action       |
| Movie A | Comedy       |
| Movie A | Drama        |


> 如果原字段（假设 `hobbies = ['game','swim','movie']`）已经是数组，则不需要 `split()`:

```sql
SELECT id, name, hobby
FROM person LATERAL VIEW explode(hobbies) tmp AS hobby;
```


### 行转列解法

行转列本质上是**对多行数据进行聚合**：原来同一个主体的信息分散在多行，最终需要整理到更少的行或一行中。因此通常首先要确定**最终一行代表什么**，也就是找到 `GROUP BY` 的分组字段，再根据目标结果选择聚合方式。原资料把常见聚合方式归纳为两类：
1. 常规聚合函数 `MAX()/SUM()` + `CASE WHEN`
2. 数组聚合函数 `collect_list()/collect_set()`（通常再配合 `concat_ws()`）

行转列常见的三种解决思路：

- **`GROUP BY + collect_list()/collect_set() + concat_ws()`**：适合把同组多行数据收集到一个字段
- **`GROUP BY + MAX()/SUM() + CASE WHEN`**：适合把某个字段的不同取值真正变成多个列
- **没有现成字段可以直接聚合**：先“人工造一列”用于聚合；部分题目也可能使用 `UNION ALL` 直接拼接最终结果

做题时可以按照下面的顺序判断：

1. 先看最终一行代表什么，确定 `GROUP BY` 字段
2. 再看原来的多行最终怎么保存：
   - 多个值收进一个字段：`collect_list/set + concat_ws`
   - 不同类型变成不同列：`CASE WHEN + MAX/SUM`
3. (optional) 如果没有字段可以 `GROUP BY`，先人工构造一个用于聚合

#### 例题 1：`GROUP BY + collect_set() + concat_ws()`

把星座和血型相同的人归到一起，姓名之间使用 `|` 分隔。

例如原表 `constellation`：

| name | constellation_name | blood_type |
| ---- | ------------------ | ---------- |
| 孙悟空  | 白羊座            | A          |
| 猪八戒  | 白羊座            | A          |
| 宋宋   | 白羊座             | B          |
| 大海   | 射手座             | A          |
| 凤姐   | 射手座             | A          |

目标结果：

| constellation_name | blood_type | name       |
| ------------------ | ---------- | ---------- |
| 白羊座                | A          | `孙悟空\|猪八戒` |
| 白羊座                | B          | `宋宋`       |
| 射手座                | A          | `大海\|凤姐`   |

1. 最终一行代表什么（确定分组字段）：一个“星座 + 血型”组合

    ```sql
    GROUP BY constellation_name, blood_type
    ```

2. 多行保存形式（多个值收进一个字段）：同一个组合中可能有多个人，需要先把组内姓名聚合起来，再用 `|` 拼接。资料给出的 SQL 是：

    ```sql
    SELECT
        constellation_name,
        blood_type,
        concat_ws('|', collect_set(name)) AS name
    FROM constellation
    GROUP BY constellation_name, blood_type;
    ```

这里三个操作职责不同：`GROUP BY` 决定哪些行属于同一组，`collect_set()` 把组内多行姓名收集成数组，`concat_ws()` 再把数组拼成字符串（`concat_ws()` 本身不是聚合函数，因此不能直接聚合多行，必须先由 `collect_set()` 等函数完成组内聚合）。

#### 例题 2：`GROUP BY + SUM() + CASE WHEN`

把 `Products` 表转换成 `Result` 表，将不同 `store` 从行变成不同列。

原表：

| product_id | store  | price |
| ---------: | ------ | ----: |
|          0 | store1 |    95 |
|          0 | store2 |   100 |
|          0 | store3 |   105 |
|          1 | store1 |    70 |
|          1 | store3 |    80 |

目标：

| product_id | store1 | store2 | store3 |
| ---------: | -----: | -----: | -----: |
|          0 |     95 |    100 |    105 |
|          1 |     70 |   NULL |     80 |

1. 最终一行代表什么（确定分组字段）：最终是一种商品一行

    ```sql
    GROUP BY product_id
    ```

2. 多行保存形式（不同类型变成不同列）：在聚合之前，需要先解决“95 应该进入 `store1` 列、100 应该进入 `store2` 列”的问题，因此使用 `CASE WHEN`：

    ```sql
    CASE WHEN store = 'store1' THEN price ELSE NULL END
    ```

    三列一起处理后，可以把原数据理解成：

    | product_id | store1 | store2 | store3 |
    | ---------: | -----: | -----: | -----: |
    |          0 |     95 |   NULL |   NULL |
    |          0 |   NULL |    100 |   NULL |
    |          0 |   NULL |   NULL |    105 |
    |          1 |     70 |   NULL |   NULL |
    |          1 |   NULL |   NULL |     80 |

3. 最后再按 `product_id` 聚合，就可以把这些行压成一行。资料使用的 SQL 是：

    ```sql
    SELECT
        product_id,
        SUM(CASE WHEN store = 'store1' THEN price ELSE NULL END) AS store1,
        SUM(CASE WHEN store = 'store2' THEN price ELSE NULL END) AS store2,
        SUM(CASE WHEN store = 'store3' THEN price ELSE NULL END) AS store3
    FROM Products
    GROUP BY product_id;
    ```


`CASE WHEN` 负责“分类放位置 / 造列”，`GROUP BY + SUM()` 负责“把同一个 product 的多行合并成一行”（`SUM` 只会把非 `NULL` 的值加起来，但这里的 `SUM()` 主要充当**聚合桥梁**，并不是重点在把多个 store 的价格加在一起）。

#### 例题 3：没有聚合字段

输出每个大洲的姓名，每个大洲一列，并且姓名按照字典顺序排列。

例如原表 `student`：

| name   | continent |
| ------ | --------- |
| Jack   | America   |
| Jane   | America   |
| Xi     | Asia      |
| Pascal | Europe    |

目标类似：

| America | Europe | Asia |
| ------- | ------ | ---- |
| Jack    | Pascal | Xi   |
| Jane    | NULL   | NULL |

1. 人工造 `id`： 原表没有一个类似 `product_id` 的字段告诉我们哪些记录应该出现在目标结果的同一行。

    ```sql
    WITH a AS (
        SELECT
            ROW_NUMBER() OVER (
                PARTITION BY continent
                ORDER BY name
            ) AS id,
            name,
            continent
        FROM student
    )
    ```

    中间结果可以理解成：

    | id | name   | continent |
    | -: | ------ | --------- |
    |  1 | Jack   | America   |
    |  2 | Jane   | America   |
    |  1 | Pascal | Europe    |
    |  1 | Xi     | Asia      |

    这样就人为建立了对应关系：各大洲的第 1 个姓名都是 `id = 1`，各大洲的第 2 个姓名都是 `id = 2`。之后就可以按照这个人工 `id` 做条件聚合。

2. 多行保存形式（不同类型变成不同列）：使用 `MAX(CASE WHEN ...)` 把不同大洲的姓名放到对应列中（聚合成列）。

    ```sql
    SELECT
        MAX(CASE continent WHEN 'America' THEN name ELSE NULL END) AS America,
        MAX(CASE continent WHEN 'Europe'  THEN name ELSE NULL END) AS Europe,
        MAX(CASE continent WHEN 'Asia'    THEN name ELSE NULL END) AS Asia
    FROM a
    GROUP BY id;
    ```

先判断目标结果中哪些数据应该处于同一行；如果原表没有这样的对应关系，就先人工构造出来，再使用普通的行转列方法。


### 有序行转列解法


有序行转列是把多行聚合成数组/字符串的同时，还要求结果按照指定字段排序。

正常行转列用`collect_list(value)`不能保证最终顺序，因为 `collect_list()` / `collect_set()` 的结果顺序在发生 shuffle 后可能不确定。

所以核心思路是：**不要直接对待输出字段排序，而是把“排序字段”和“待输出字段”绑定在一起 -> 排序 -> 再取出真正需要的字段。**

#### 方法 1：拼接排序字段 + `sort_array()` + `transform()`

先用`concat(delivery_time, customer_id)` 把排序字段放在待输出字段前面，然后`collect_list()` 聚合成数组，再用 `sort_array()` 排序，最后用 `transform()` 去掉前面的排序字段，只保留真正的目标字段。

完整写法：

```sql
SELECT
    rider_id,
    concat_ws(
        ',',
        transform(
            sort_array(collect_list(time_customer)),
            x -> substr(x, 20)
        )
    ) AS customer_list
FROM (
    SELECT
        rider_id,
        concat(delivery_time, customer_id) AS time_customer
    FROM ods_game_dev.t_delivery_orders
) t
GROUP BY rider_id;
```

其中：

* `concat()`：把**排序字段 + 目标字段**绑定起来
* `collect_list()`：聚合成数组
* `sort_array()`：对数组排序
* `transform()`：去掉前面的排序字段，只保留目标字段（资料这里用 `substr(x, 20)`，是因为前面的日期时间字符串长度固定为 19 位，因此从第 20 位开始取真正的 `customer_id`）
* `concat_ws()`：最终拼成字符串


#### 方法 2：`STRUCT` 绑定字段 + `array_sort()`

比字符串拼接更直接的方法，是用`struct(delivery_time, customer_id)`把排序字段和目标字段组成一个 `STRUCT`，并把排序字段放在前面。接着，用 `array_sort()` 对数组进行排序。

```sql
SELECT
    rider_id,
    array_sort(
        collect_list(
            struct(delivery_time, customer_id)
        )
    ).customer_id AS customer_id_list
FROM ods_game_dev.t_delivery_orders
GROUP BY rider_id;
```

用排序字段和最终字段组成 `STRUCT`，排序字段放前面，排序完成后再通过 `.字段名` 提取真正需要的数据。


---
## 5. 同时在线问题

### 同时在线问题概述

同时在线问题可以定义为：给定每个用户的进入时间和离开时间，统计某个时间点或某个时间段内，有多少用户同时处于某些状态（比如，"在线/活跃/等待"），以及一些衍生问题（比如最多同时在线用户数等问题）。

### 同时在线问题解法

#### 题目描述

用户行为日志表 `tb_user_log` 记录了用户浏览文章的行为，每条记录包含用户进入文章的时间 `in_time` 和离开文章的时间 `out_time`。

表中主要字段如下：

| 字段 | 含义 |
|---|---|
| `id` | 记录 ID |
| `uid` | 用户 ID |
| `article_id` | 文章 ID |
| `in_time` | 用户进入文章的时间 |
| `out_time` | 用户离开文章的时间 |
| `sign_in` | 是否签到 |

其中，`article_id = 0` 表示用户处于非文章内容页面，例如 App 首页、活动页等，因此统计文章在线人数时需要排除这些记录。

#### 问题要求

统计每篇文章在任意时刻的**最大同时在线人数**，如果同一时刻有进入也有离开，先记录用户增加再记录减少，结果按最大人数降序排列。

| 输出字段 | 含义 |
|---|---|
| `article_id` | 文章 ID |
| `max_uv` | 最大同时在线人数 |

#### 解决办法

1. **标记时间**：开始阅读文章的时间点记为1，结束阅读文章的时间点记为-1，然后将两部分数据 union all 起来

    ```sql
    WITH t1 AS (
        SELECT article_id, in_time AS dt, 1 AS num
        FROM tb_user_log
        WHERE article_id != 0 -- 只选择正在观看的记录

        UNION ALL

        SELECT article_id, out_time AS dt, -1 AS num
        FROM tb_user_log
        WHERE article_id != 0 -- 只选择正在观看的记录
    )
    ```

2. **使用开窗函数对人数进行累加**：这里需要按照事件时间来对数据进行排序，然后对在线人数进行累计计数
    > 当同一时刻即有人开始阅读，又有人结束阅读时，是先统计进入的还是退出的 （这个属于边界情况，写之前一定要确认好）。

    ```sql
    WITH t2 AS (
        SELECT 
            article_id,
            sum(num) OVER (
                PARTITION BY article_id
                ORDER BY dt ASC, num DESC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS cnt
        FROM t1
    )
    ```
3. **找出最终的结果**：根据题目要求，筛选出最终结果即可，这里直接求出来最大的在线人数即可。

    ```sql
    SELECT article_id, MAX(cnt) AS max_uv
    FROM t2
    GROUP BY article_id
    ORDER BY max_uv DESC;
    ```


**示例：假设文章 `9001` 有两位用户访问**

原始数据：

| article_id | in_time | out_time |
|---|---|---|
| 9001 | 10:00 | 10:10 |
| 9001 | 10:05 | 10:15 |

经过第 1 步拆分事件后：

| article_id | dt | num |
|---|---|---:|
| 9001 | 10:00 | +1 |
| 9001 | 10:05 | +1 |
| 9001 | 10:10 | -1 |
| 9001 | 10:15 | -1 |

经过第 2 步按时间累计后：

| article_id | dt | num | cnt |
|---|---|---:|---:|
| 9001 | 10:00 | +1 | 1 |
| 9001 | 10:05 | +1 | 2 |
| 9001 | 10:10 | -1 | 1 |
| 9001 | 10:15 | -1 | 0 |

经过第 3 步取最大值：

| article_id | max_uv |
|---|---:|
| 9001 | 2 |

因此文章 `9001` 的最大同时在线人数为 `2`。


#### 同时在线问题分类

1. 用户表 & 订单表
2. 按类细分

---

## 6. 留存问题

留存问题的核心在于如何保持用户的长期活跃与粘性，它直接反映了产品吸引力、用户满意度和运营效能。在业务题中，只要出现“留存率”、“次日留存”等关键词，即可迅速定位为留存问题。


- 新用户留存：主要考察一批新用户(首次登录/活跃)后续的留存情况，通过分析新用户的行为数据，企业可以找出影响新用户留存的关键因素，如产品易用性、用户体验、价值感知等，并据此优化产品或服务，提高新用户留存率。
- 产品功能留存：对产品或服务中各个功能的留存情况进行跟踪和分析。通过对比不同功能的留存率，企业可以了解哪些功能更受用户欢迎，哪些功能需要改进或优化。
- 用户生命周期：用户生命周期是指用户从首次接触产品或服务到最终流失或成为忠实用户的整个过程。通过跟踪和分析用户在生命周期中的不同阶段的留存情况，可以制定更有针对性的用户运营策略，如新手引导、成长激励、挽留措施等，从而延长用户生命周期，提高用户价值和忠诚度。
- 来源入口留存：类似于产品功能留存，主要考察通过不同渠道/入口进入产品的用户后续的留存情况。诚然，用户是否留存很大程度上取决于产品本身是否有足够的吸引力，但从哪里来的用户质量更高，也是我们需要关注的。

### 留存问题核心参数

- **新用户7日留存率**
  - 计算公式：7日留存率 = （第7天仍然活跃的用户数 / 第1天新增用户数）* 100%
  - 解释：该指标反映了新用户在注册或首次使用产品后的第7天是否仍然活跃。高留存率意味着产品能够吸引新用户并保持他们的兴趣，低留存率则可能表明产品存在问题或用户体验不佳
  - 例子：假设某应用在1月1日有100位新注册用户，为了计算这100位用户的七日留存率，我们需要关注这些用户在1月7日（即注册后的第七天）是否再次使用该产品。

- **新用户7日内留存率**
    - 计算公式：用户在新增或使用产品后一周内（第1天至第7天）每天回到产品的用户数相加并去重，然后除以首日新增用户数 = （第1天至第7天仍然活跃的用户数 / 第1天新增用户数）* 100%
    - 解释：该指标反映了新用户在注册或首次使用产品后的第2天至第7天是否仍然活跃。高留存率意味着产品能够吸引新用户并保持他们的兴趣，低留存率则可能表明产品存在问题或用户体验不佳
    - 例子：如果第一天新增了100个用户，在接下来的七天内，分别有4、2、15、10、8、5、17个用户回到产品(这些用户不重复)，那么7日内留存率为（4+2+15+10+8+5+17）/ 100 = 61/100 = 61%。

### 留存问题常用函数

1. `DATE_FORMAT()`：统一日期粒度，登录时间如果带时分秒，通常先转成日期，再按“日期 + 用户”去重。

    ```sql
    date_format(log_time, 'yyyy-MM-dd')
    ```

2. `MIN()` / `ROW_NUMBER()`：找首次登录日期，如果题目考察**新用户留存**，需要先确定每个用户第一次登录的日期。

    ```sql
    MIN(login_date)
    ```

    或者：

    ```sql
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY login_date
    )
    ```

    然后取 `rk = 1`。资料明确说明这两种方式都可以用于计算首次登录日期。

3. `DATEDIFF()`：计算两个日期相差多少天，是留存判断最核心的函数。

    ```sql
    datediff(后续登录日期, 基准日期)
    ```

4. `IF()` / `CASE WHEN`：给留存打标

    ```sql
    IF(datediff(b.login_date, a.login_date) = 1, 1,0)
    ```
    
    即满足留存条件打 `1`，不满足打 `0`。


5. `MAX()`：把同一个用户的多条留存记录压成一个标签，一个基准用户 `LEFT JOIN` 后可能对应多条后续登录记录，因此可以：

    ```sql
    MAX(
        IF(datediff(b.login_date, a.login_date) = 1, 1, 0)
    )
    ```

    只要该用户有一次满足条件，最终就是 `1`。资料的高效写法也是先按“日期 + 用户”聚合，并用 `MAX()` 得到用户级留存标签。

6. `COUNT(DISTINCT)` / `SUM()` / `COUNT()`：计算留存率

    - 方法一：

        ```sql
        COUNT(DISTINCT IF(flag = 1, user_id, NULL))
        COUNT(DISTINCT user_id)
        ```

    - 方法二：先保证**日期 + 用户只有一行**，再：

        ```sql
        SUM(flag) / COUNT(1)
        ```
        
        这种方式可以避免大量使用 `COUNT(DISTINCT)`，在大数据场景下一般效率更高。


### 留存问题解法

留存问题基本固定为**三步走**：

1. 确定基准用户群体
2. 为留存打标
3. 计算留存率

#### 1. 确定基准用户群体

先判断题目到底要追踪哪批用户：

* **新用户留存**：找每个用户的首次登录日期
* **每日活跃用户留存**：当天登录过的用户就是基准用户

所以第一步最重要的问题是： **最终要观察的是哪一批用户后续有没有回来？**

如果原始表一天内同一用户有多条行为记录，应先：

```sql
GROUP BY 日期, user_id
```

保证每天每个用户只有一条，否则后续 JOIN 容易造成数据膨胀。资料将这一点作为留存题的重要注意事项。

#### 2. 基准用户 `LEFT JOIN` 后续活跃记录并打标

基准用户作为主表，LEFT JOIN 后续全量活跃数据，再用 `DATEDIFF()` 判断是否符合留存条件。使用 `LEFT JOIN`，是为了让**后续没有回来的人仍然保留在基准用户中。

基本结构：

```sql
FROM base_user a
LEFT JOIN activity b
    ON a.user_id = b.user_id
```

然后：

```sql
IF(
    datediff(b.login_date, a.base_date) = 留存天数, 1, 0
) AS flag
```

同一用户可能关联到多条后续记录，因此通常再按：

```sql
GROUP BY base_date, user_id
```

并使用：

```sql
MAX(flag)
```

最终得到：

| base_date | user_id | flag |
| --------- | ------- | ---: |
| 01-01     | A       |    1 |
| 01-01     | B       |    0 |
| 01-01     | C       |    1 |

#### 3. 计算留存率

核心公式：留存率 = 符合留存条件的用户数 / 基准用户总数

```sql
SELECT
    base_date,
    SUM(flag) / COUNT(1) AS retention_rate
FROM t
GROUP BY base_date;
```

#### 例子：计算每天新用户的次日留存率

原始登录表 `user_login`：

| user_id | login_date |
| ------- | ---------- |
| A       | 2026-01-01 |
| A       | 2026-01-02 |
| A       | 2026-01-04 |
| B       | 2026-01-01 |
| B       | 2026-01-03 |
| C       | 2026-01-02 |
| C       | 2026-01-03 |

要求：计算**每天首次登录用户的次日留存率**。

1. 找到每个用户首次登录日期

    ```sql
    WITH base_user AS (
        SELECT
            user_id,
            MIN(login_date) AS first_login_date
        FROM user_login
        GROUP BY user_id
    )
    ```

    得到：

    | user_id | first_login_date |
    | ------- | ---------------- |
    | A       | 2026-01-01       |
    | B       | 2026-01-01       |
    | C       | 2026-01-02       |

    因此：

    ```text id
    2026-01-01 新用户：A、B
    2026-01-02 新用户：C
    ```

2. `LEFT JOIN` 后续登录记录并打留存标签

    ```sql
    SELECT
        a.first_login_date,
        a.user_id,
        MAX(
            CASE
                WHEN datediff(b.login_date, a.first_login_date) = 1
                THEN 1
                ELSE 0
            END
        ) AS next_day_flag
    FROM base_user a
    LEFT JOIN user_login b
        ON a.user_id = b.user_id
    GROUP BY
        a.first_login_date,
        a.user_id;
    ```

    得到：

    | first_login_date | user_id | next_day_flag |
    | ---------------- | ------- | ------------: |
    | 2026-01-01       | A       |             1 |
    | 2026-01-01       | B       |             0 |
    | 2026-01-02       | C       |             1 |

    解释：

    ```text
    A：01-01 首次登录，01-02 又登录 -> 次日留存 = 1
    B：01-01 首次登录，01-02 没登录 -> 次日留存 = 0
    C：01-02 首次登录，01-03 又登录 -> 次日留存 = 1
    ```

    这里使用 `MAX()` 是因为一个用户可能关联到多条后续登录记录，只要其中一条满足次日条件，最终标签就是 `1`。

3. 计算次日留存率

    ```sql
    WITH base_user AS (
        SELECT
            user_id,
            MIN(login_date) AS first_login_date
        FROM user_login
        GROUP BY user_id
    ),
    user_flag AS (
        SELECT
            a.first_login_date,
            a.user_id,
            MAX(
                CASE
                    WHEN datediff(b.login_date, a.first_login_date) = 1
                    THEN 1
                    ELSE 0
                END
            ) AS next_day_flag
        FROM base_user a
        LEFT JOIN user_login b
            ON a.user_id = b.user_id
        GROUP BY
            a.first_login_date,
            a.user_id
    )

    SELECT
        first_login_date,
        COUNT(1) AS new_user_num,
        SUM(next_day_flag) AS next_day_user_num,
        ROUND(SUM(next_day_flag) / COUNT(1), 2) AS next_day_retention
    FROM user_flag
    GROUP BY first_login_date;
    ```

    结果：

    | first_login_date | new_user_num | next_day_user_num | next_day_retention |
    | ---------------- | -----------: | ----------------: | -----------------: |
    | 2026-01-01       |            2 |                 1 |               0.50 |
    | 2026-01-02       |            1 |                 1 |               1.00 |

---

## 7. 开窗/聚合函数（重点）

### 波峰波谷

### 掐头去尾

### 合并区间

### 空值填充


### 前后列转换


### 相互关注


### 无效搜索


### 相邻问题

---
## 8. 正则表达

### 正则表达式概述

- 字符类
    - `.`：匹配任意单个字符（换行符除外）[例如，正则表达式 `a.b` 可以匹配 "acb"、"a1b"、"a b" 等字符串，但不能匹配 "ab" 或 "a\nb"]
    - `[]`：匹配方括号内的任意一个字符 [例如，正则表达式 `[abc]` 可以匹配 "a"、"b" 或 "c"，但不能匹配 "d" 或 "ab"]
    - `[^]`：匹配不在方括号内的任意一个字符 [例如，正则表达式 `[^abc]` 可以匹配除 "a"、"b"、"c" 之外的任意字符，如 "d"、"e"、"1" 等]

- 转义字符
    - `\d`：匹配任意一个数字字符（0-9）[例如，正则表达式 `\d` 可以匹配 "0"、"1"、"2" 等数字字符，但不能匹配字母或特殊字符]
    - `\D`：匹配任意一个非数字字符 [例如，正则表达式 `\D` 可以匹配 "a"、"b"、"!" 等非数字字符，但不能匹配数字字符]
    - `\w`：匹配任意一个字母、数字或下划线字符 [例如，正则表达式 `\w` 可以匹配 "a"、"1"、"_" 等字符，但不能匹配空格或特殊字符]
    - `\W`：匹配任意一个非字母、数字或下划线字符 [例如，正则表达式 `\W` 可以匹配空格、"!"、"@" 等非字母、数字或下划线字符，但不能匹配字母、数字或下划线]
    - `\s`：匹配任意一个空白字符（包括空格、制表符、换行符等）[例如，正则表达式 `\s` 可以匹配空格、"\t"、"\n" 等空白字符，但不能匹配字母或数字]
    - `\S`：匹配任意一个非空白字符 [例如，正则表达式 `\S` 可以匹配 "a"、"1"、"!" 等非空白字符，但不能匹配空格或制表符]
    - `\b`：匹配一个单词边界（单词的开头或结尾）[例如，正则表达式 `\bword\b` 可以匹配 "word" 这个单词，但不能匹配 "sword" 或 "words"]
    - `\B`：匹配一个非单词边界（单词内部）[例如，正则表达式 `\Bword\B` 可以匹配 "sword" 或 "words"，但不能匹配 "word" 这个单词]
    - `\A`：匹配输入字符串的开头 [例如，正则表达式 `\Aword` 可以匹配以 "word" 开头的字符串，但不能匹配 "sword" 或 "words"]
    - `\Z`：匹配输入字符串的结尾 [例如，正则表达式 `word\Z` 可以匹配以 "word" 结尾的字符串，但不能匹配 "sword" 或 "words"]
    - `\z`：匹配输入字符串的结尾（与 `\Z` 类似，但不允许换行符） [例如，正则表达式 `word\z` 可以匹配以 "word" 结尾的字符串，但不能匹配 "sword" 或 "words"]
    - `\n`：匹配换行符 [例如，正则表达式 `\n` 可以匹配换行符，但不能匹配其他字符]
    - `\r`：匹配回车符 [例如，正则表达式 `\r` 可以匹配回车符，但不能匹配其他字符]
    - `\t`：匹配制表符 [例如，正则表达式 `\t` 可以匹配制表符，但不能匹配其他字符]
    - `\f`：匹配换页符 [例如，正则表达式 `\f` 可以匹配换页符，但不能匹配其他字符]
    - `\v`：匹配垂直制表符 [例如，正则表达式 `\v` 可以匹配垂直制表符，但不能匹配其他字符]
    - `\e`：匹配转义符 [例如，正则表达式 `\e` 可以匹配转义符，但不能匹配其他字符]

- 数量限定符
    - `*`：匹配前面的子表达式零次或多次 [例如，正则表达式 `a*` 可以匹配 ""、"a"、"aa"、"aaa" 等字符串]
    - `+`：匹配前面的子表达式一次或多次 [例如，正则表达式 `a+` 可以匹配 "a"、"aa"、"aaa" 等字符串，但不能匹配 ""]
    - `?`：匹配前面的子表达式零次或一次 [例如，正则表达式 `a?` 可以匹配 "" 或 "a"，但不能匹配 "aa"]
    - `{n}`：匹配前面的子表达式恰好 n 次 [例如，正则表达式 `a{3}` 可以匹配 "aaa"，但不能匹配 "aa" 或 "aaaa"]
    - `{n,}`：匹配前面的子表达式至少 n 次 [例如，正则表达式 `a{2,}` 可以匹配 "aa"、"aaa"、"aaaa" 等字符串，但不能匹配 "a"]
    - `{n,m}`：匹配前面的子表达式至少 n 次，至多 m 次 [例如，正则表达式 `a{2,4}` 可以匹配 "aa"、"aaa"、"aaaa"，但不能匹配 "a" 或 "aaaaa"]

- 边界匹配
    - `^`：匹配输入字符串的开头 [例如，正则表达式 `^a` 可以匹配以 "a" 开头的字符串，但不能匹配 "ba" 或 "ca"]
    - `$`：匹配输入字符串的结尾 [例如，正则表达式 `a$` 可以匹配以 "a" 结尾的字符串，但不能匹配 "ab" 或 "ac"]

- 特殊字符
    - `|`：匹配左边或右边的子表达式 [例如，正则表达式 `a|b` 可以匹配 "a" 或 "b"，但不能匹配 "c"]
    - `()`：将多个字符组合成一个单元 [例如，正则表达式 `(ab)+` 可以匹配 "ab"、"abab"、"ababab" 等字符串]
    - `(?:)`：非捕获分组，用于组合多个字符，但不捕获匹配结果 [例如，正则表达式 `(?:ab)+` 可以匹配 "ab"、"abab"、"ababab" 等字符串，但不会捕获匹配结果]
    - `(?=)`：正向前瞻，用于匹配某个位置后面跟着的内容，但不包括该内容在内 [例如，正则表达式 `a(?=b)` 可以匹配 "a" 后面跟着 "b" 的位置，但不包括 "b"]
    - `(?! )`：负向前瞻，用于匹配某个位置后面不跟着的内容，但不包括该内容在内 [例如，正则表达式 `a(?!b)` 可以匹配 "a" 后面不跟着 "b" 的位置，但不包括 "b"]

### 相关函数

#### `RLIKE` 

用于判断字符串是否匹配指定的正则表达式模式，若匹配则返回 true，否则返回 false。它本质上是一个布尔运算符，常用于 WHERE 子句中进行条件筛选（`string_expression RLIKE regex_pattern`）。

```sql
--- sample data
CREATE TABLE employees (
    id INT,
    name STRING
);
INSERT INTO employees VALUES
(1, 'John'),
(2, 'Alice'),
(3, 'Jack');
 
-- 使用 rlike 进行筛选
SELECT *
FROM employees
WHERE name rlike '^J.*';
```


#### REGEXP_EXTRACT

从输入字符串中按照正则表达式进行匹配，并返回指定组的匹配结果。这里的组是指正则表达式中用括号 () 括起来的部分，索引从 1 开始计数，若 index 为 0 则返回整个匹配的字符串（`REGEXP_EXTRACT(STRING string_expression, STRING regex_pattern, INT index)`）。

```sql
-- sample data
CREATE TABLE logs (
    log_id INT,
    log_content STRING
);
INSERT INTO logs VALUES
(1, '[2025-01-25 10:00:00] [登录] 用户 John 登录系统'),
(2, '[2025-01-25 11:00:00] [退出] 用户 Alice 退出系统');
 
-- 使用 regexp_extract 提取操作类型
SELECT
    log_id,
    log_content,
    REGEXP_EXTRACT(log_content, "\\[(.*?)\\]\\s*\\[(.*?)\\]", 1) AS second_bracket_content,
    REGEXP_EXTRACT(log_content, "\\[(.*?)\\]\\s*\\[(.*?)\\]", 2) AS second_bracket_content
FROM logs;

```

#### REGEXP_REPLACE

将输入字符串中匹配正则表达式的部分替换为指定的字符串（`REGEXP_REPLACE(STRING string_expression, STRING regex_pattern, STRING replacement)`）。

```sql
-- sample data
CREATE TABLE products (
    product_id INT,
    product_name STRING
);
INSERT INTO products VALUES
(1, 'iPhone 14'),
(2, 'iPad Pro 12.9');
 
-- 使用 regexp_replace 替换数字
SELECT
    product_id,
    regexp_replace(product_name, '[0-9]+', '') AS new_product_name
FROM products;
```

---
## 9. JSON字符串与解析

### JSON介绍

#### 什么是 JSON

JSON（**JavaScript Object Notation**）是一种轻量级的数据交换格式，本质上是用**文本形式组织和表示结构化数据**，方便数据存储、传输和解析。

数仓中常见于：

* **埋点 / 日志数据**：记录用户 ID、操作时间、事件类型、页面、按钮等行为信息。
* **扩展业务字段**：把不同业务特有的维度、指标放进公共 JSON 字段，避免频繁给大表新增字段。
* **配置、API 数据**：JSON 对象适合描述一个实体或配置。
* **列表数据**：JSON 数组适合保存商品列表、文章列表、收藏列表等。

#### JSON 的两种基本形式

JSON 主要分为 **JSON Object（对象）** 和 **JSON Array（数组）**。

* **JSON Object：`{}`**

    由多个 `key : value` 组成，通常描述一个对象：

    ```json
    {
        "name": "Alice",
        "age": 30,
        "city": "Beijing",
        "isStudent": false,
        "hobbies": ["reading", "swimming"]
    }
    ```

    可以理解为：

    ```text
    {
        属性1 : 值1,
        属性2 : 值2,
        属性3 : 值3
    }
    ```

* **JSON Array：`[]`**

    表示一组数据，通常包含多个 JSON Object：

    ```json
    [
    {"name":"Alice","age":30},
    {"name":"Bob","age":28},
    {"name":"Charlie","age":32}
    ]
    ```

    可以理解为：

    ```text
    [
        JSON对象1,
        JSON对象2,
        JSON对象3
    ]
    ```

* JSON 嵌套
    - JSON 套 JSON
    - JSON 套 ARRAY
    - ARRAY 套 JSON
    - ARRAY 套 ARRAY
    - 其他复杂嵌套


#### JSON 字符串与 SQL 数据类型

SQL 表中看到 `'{"name":"Alice","age":30}'` 这样的字段，通常是 **JSON 字符串**，它的类型是 `STRING`。

要进一步使用数组、结构体等操作，需要先解析：`JSON STRING` -> 解析 -> `STRUCT` / `ARRAY` / `MAP`。

解析之后，不同数据类型的访问方式不同：

| 数据类型     | 常见操作              |
| -------- | ----------------- |
| `STRUCT` | `data.name`       |
| `MAP`    | `data['name']`    |
| `ARRAY`  | `explode(data)` 等 |


### JSON函数

#### 1. `GET_JSON_OBJECT()`

用于**根据 JSON 路径，从 JSON 字符串中提取指定元素**。适合提取少量字段，尤其是嵌套字段。

##### 语法

```sql
get_json_object(json_string, path)
```

* `json_string`：待解析 JSON 字符串
* `path`：JSON 路径
* `$`：表示 JSON 根节点
* `$.a.b`：表示依次进入 `a -> b`

例如：

```json
{
  "name": "Alice",
  "scores": {
    "math": 85
  }
}
```

- 取 `name`：

    ```sql
    get_json_object(student_info, '$.name')
    ```

- 取 `math`：

    ```sql
    get_json_object(student_info, '$.scores.math')
    ```

##### 常见用法

* **提取一个普通字段**

    ```sql
    get_json_object(json_col, '$.name')
    ```

* **提取嵌套字段**

    ```sql
    get_json_object(json_col, '$.user.name')
    ```

* **提取数组中的指定元素**

    ```sql
    get_json_object(name, '$[0]')
    ```

    即先取 JSON 数组的第一个对象，再进行后续解析。

##### 注意点

* 一次调用主要提取**一个元素**。
* 提取多个字段需要调用多次：

    ```sql
    SELECT
        get_json_object(info, '$.name'),
        get_json_object(info, '$.age'),
        get_json_object(info, '$.city')
    FROM t;
    ```

    这样会对同一 JSON 字符串重复解析，因此字段较多时性能可能较差。资料特别强调了这一点。


#### 2. `JSON_TUPLE()`

用于**一次从 JSON 字符串中提取多个字段**，适合多个同层字段的解析。相比多次调用 `get_json_object()`，只需要解析一次 JSON，效率通常更高。

##### 语法

```sql
json_tuple(
    json_string,
    key1,
    key2,
    key3,
    ...
)
```

注意：key **不需要写 `$.`**。

例如：

```json
{
  "name":"Alice",
  "math_score":85,
  "english_score":90
}
```

```sql
SELECT
    json_tuple(
        student_info,
        'name',
        'math_score',
        'english_score'
    ) AS (name, math_score, english_score)
FROM student;
```

结果：

| name  | math_score | english_score |
| ----- | ---------: | ------------: |
| Alice |         85 |            90 |

##### 常见用法

* **一次解析多个同层字段**

    ```sql
    json_tuple(
        json_col,
        'name',
        'age',
        'city'
    )
    ```

* **嵌套 JSON 分层解析**

    例如：

    ```json
    {
    "name":"Alice",
    "scores":{
        "math":85,
        "english":90
    }
    }
    ```

    可以分两层解析：先取 name、scores，再解析 scores

    ```sql
    SELECT
        name,
        math_score,
        english_score
    FROM student_data s

    LATERAL VIEW json_tuple(
        s.student_info,
        'name',
        'scores'
    ) t1 AS name, scores_json

    LATERAL VIEW json_tuple(
        scores_json,
        'math',
        'english'
    ) t2 AS math_score, english_score;
    ```

原表：

| student_info                                         |
| ---------------------------------------------------- |
| `{"name":"Alice","scores":{"math":85,"english":90}}` |

第一层解析后：

| name  | scores_json                |
| ----- | -------------------------- |
| Alice | `{"math":85,"english":90}` |

第二层解析后：

| name  | math_score | english_score |
| ----- | ---------: | ------------: |
| Alice |         85 |            90 |


#### 3. `FROM_JSON()`

`from_json()` 用于**把整个 JSON STRING 映射成 STRUCT / ARRAY / MAP 等结构化类型**。

当 JSON 很复杂、嵌套层级较多，或者需要继续使用 `explode()`、数组函数、MAP 操作时，使用 `from_json()` 更合适。资料将它总结为对复杂 JSON **“化繁为简”**。

##### 函数结构

```sql
from_json(json_str, schema)
```

* `json_str`：JSON 字符串
* `schema`：JSON 对应的目标结构，需要描述字段名和数据类型
* 返回：对应的 `STRUCT / ARRAY / MAP` 等结构化数据

##### Schema

常见映射：

| JSON 结构   | Schema                                                 |
| --------- | ------------------------------------------------------ |
| 普通对象      | `struct<name:string,age:int>`                          |
| 嵌套对象      | `struct<user:struct<id:int,name:string>,score:double>` |
| JSON 数组   | `array<struct<course:string,score:int>>`               |
| JSON 键值结构 | `map<string,int>`                                      |

例如：

```json
{
  "name":"Alice",
  "age":30
}
```

```sql
SELECT 
    from_json(
        json_str, 
        'struct<name:string,age:int>' 
    ) AS json_data 
FROM t;
```

得到：`STRUCT<name:string, age:int>`，之后可以`json_data.name`、`json_data.age` 直接访问。


##### 常见用法

* **复杂 JSON 一次性结构化**

    ```sql
    from_json(
        json_col,
        'struct<
            name:string,
            scores:struct<math:int,english:int>
        >'
    )
    ```
    
    通过 struct_name.field_name 访问嵌套字段。

* **JSON Array -> ARRAY -> `explode()`**

    ```json
    [
        {"course":"math","score":95},
        {"course":"english","score":90}
    ]
    ```

    ```sql
    LATERAL VIEW explode(
        from_json(
            json_str,
            'array<struct<course:string,score:int>>'
        )
    ) tmp AS course_info
    ```

* **解析为 MAP**

    ```sql
    from_json(
        json_str,
        'array<map<string,string>>'
    )
    ```

    炸裂后某个元素 `a` 是 MAP，则：

    ```sql
    a['id']
    a['name']
    ```

##### 注意点

- `schema` 需要与 JSON 的**结构和数据类型匹配**：
- Mapping
    - JSON 对象 -> STRUCT
    - JSON 数组 -> ARRAY
    - 键值结构 -> MAP

#### 4. `LATERAL VIEW JSON_TUPLE`

用于在保留原表字段的同时，**把 JSON 中多个字段解析成独立列**。

##### 语法

```sql
SELECT
    ...
FROM table_name LATERAL VIEW json_tuple(
    json_col,
    'key1',
    'key2'
) tmp AS col1, col2;
```

例如原表：

| id | user_info                   |
| -: | --------------------------- |
|  1 | `{"name":"Alice","age":30}` |

SQL：

```sql
SELECT
    id,
    name,
    age
FROM user LATERAL VIEW json_tuple(
    user_info,
    'name',
    'age'
) tmp AS name, age;
```

结果：

| id | name  | age |
| -: | ----- | --: |
|  1 | Alice |  30 |


> 注意它和 `explode()` 的作用不同：JSON的LATERAL VIEW 主要是**把 JSON 中的多个 key 解析成多列**，而 `explode()` 是**把 ARRAY / MAP 拆成多行**。


#### 5. `TO_JSON()`（

`to_json()` 和 `from_json()` 方向相反，用于**把结构化数据转换成 JSON 字符串**；常和 `STRUCT` / `ARRAY` / `MAP` 等类型配合使用。


##### 语法

```sql
to_json(struct_or_array_or_map)
```

##### 常见用法

| id | name | age |
| -: | ---- | --: |
|  1 | 张三   |  20 |

可以先构造 MAP：

```sql
to_json(
    map(
        'id', CAST(id AS STRING),
        'name', CAST(name AS STRING),
        'age', CAST(age AS STRING)
    )
)
```

得到类似：

```json
{"id":"1","name":"张三","age":"20"}
```

##### 注意点

* 所有 key 需要保持同一数据类型
* 所有 value 也需要保持同一数据类型

---

## 10. SQL高性能优化

---