---
title: "SQL基础"
---

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




## 4. 行转列/列转行问题

### 行转列问题概述

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


---

## 10. SQL高性能优化

---