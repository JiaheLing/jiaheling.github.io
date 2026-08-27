---
title: "SQL基础"
---

---
## SQL顺序

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
## TopN问题

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

1. `OVER()`：表示使用窗口计算。
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

并不是所有窗口函数都需要显式指定窗口帧，**排名类函数不需要显式指定窗口帧**。

```sql
ROW_NUMBER()
RANK()
DENSE_RANK()
NTILE()
PERCENT_RANK()
```

因为它们主要关心当前行在整个分区排序以后处于什么位置，不需要显式指定窗口帧 `ROWS` 或 `RANGE`。通常只需要：

```sql
函数() OVER (
    PARTITION BY ...
    ORDER BY ...
)
```


窗口帧主要对下面这些函数更有意义，因为这些函数需要明确当前这一行计算时，到底应该使用哪些行，

```sql
SUM()
AVG()
COUNT()
MIN()
MAX()
FIRST_VALUE()
LAST_VALUE()
```




---
## 连续登陆问题

TBD

---