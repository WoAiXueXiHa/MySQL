## 1. 为什么有索引也不一定用索引

先建立一个基础表：

```sql
create table orders (
    id bigint primary key,
    user_id bigint not null,
    status varchar(20) not null,
    amount decimal(10, 2) not null,
    note varchar(500),
    index idx_user_id (user_id),
    index idx_status (status)
);
```

> MySQL uses a cost-based optimizer to determine the best way to resolve a query.

先说结论：**索引不是 MySQL 必须走的路，它只是优化器手里的一条候选路线。优化器最后选哪条路，看的是估算成本。**

可以把 MySQL 想象成一个快递员。表里有 100 万行数据，现在要找 `user_id = 1001` 的几条记录。索引就像小区楼栋导航，直接告诉你去几号楼几单元，把 100 万行数据缩小成几行小范围，当然快。

而索引的本质是：**快速找到具有特定列值的行。**

> Indexes are used to find rows with specific column values quickly.

但是如果要查的是：

```sql
select * from orders where status = 'paid';
```

假设 `status = 'paid'` 占了全表 80%。这时候索引像什么？像导航告诉你“小区里 80% 的房间都要送”。你先查导航，再一家家上门，可能还不如直接按楼栋顺序扫过去。

`select *` 会让这种情况更加明显。还是这条查询：

```sql
select * from orders where status = 'paid';
```

因为要查询全部字段，需要拿到 `id`、`user_id`、`status`、`amount`、`note` 等列。但是 `idx_status` 这个二级索引里只有：

```text
status + 主键 id
```

所以流程会变成：

```text
idx_status -> 找到很多 paid 记录 -> 拿到很多 id -> 回到 primary 查整行
```

如果匹配行很多，回表次数也会很多。

所以判断一条查询走索引快不快，不能只问：**where 条件有没有索引？**

还要继续看：

- 这个索引能不能明显缩小范围？
- 返回的列能不能被索引覆盖？
- 如果不能覆盖，需要回表多少次？

假设执行：

```sql
explain select * from orders where status = 'paid';
```

可能看到这个结果：

```text
possible_keys: idx_status   # 可能能用的索引
key: null                   # 最终实际用的索引
```

这说明 MySQL 知道 `idx_status` 可能有关，但是优化器估算之后，觉得用它不划算，最后没有选择这个索引。

官方文档说：

> If key is NULL, MySQL found no index to use for executing the query more efficiently.

如果 `key` 是 `null`，说明 MySQL 没有找到能让这个查询更高效执行的索引。

## 2. 优化器根据什么选择索引

> Storage engines collect statistics about tables for use by the optimizer.

存储引擎会收集表的统计信息，给优化器使用。所以，优化器不是凭感觉选索引。它会依赖统计信息，估算每条路线要读多少行、成本多高。

### 2.1 统计信息是什么

还是用开头的 `orders` 表，MySQL 需要知道这些信息：

```text
orders 大概有多少行？
idx_user_id 的不同值多不多？
idx_status 的不同值多不多？
某个索引值平均会命中多少行？
```

比如：

```text
user_id：可能有 50 万个不同值
status：可能只有 paid、pending、cancelled 三个值
```

那么优化器就能判断：

```text
where user_id = 1111 可能命中很少行
where status = 'paid' 可能命中很多行
```

### 2.2 基数和选择性

> If there is a choice between multiple indexes, MySQL normally uses the index that finds the smallest number of rows (the most selective index).

如果有多个索引可以选择，MySQL 通常会选择能找到最少行的索引，也就是选择性更高的索引。

可以这样理解：

- 基数：一个索引列大概有多少种不同的值。
- 选择性：这个条件能不能把范围缩小。

比如：

```text
user_id 有 50 万种不同值，选择性高。
status 只有 3 种不同值，选择性低。
```

所以这两种查询不一样：

```sql
select * from orders where user_id = 1111;
select * from orders where status = 'paid';
```

第一个更容易走 `idx_user_id`，因为它更可能把范围缩小到很少的行。

第二个不一定走 `idx_status`，因为它可能命中很多行。

### 2.3 估算行数

> To estimate how many rows must be read for each ref access.

MySQL 需要估算每种访问方式大概要读多少行。这也是看 `explain` 的 `rows` 字段时要理解的东西。

如果看到：

```text
key: idx_user_id
rows: 3
```

可以理解为：优化器估算走 `idx_user_id` 大概要读 3 行。

再看：

```text
key: idx_status
rows: 800000
```

可以理解为：优化器估算走 `idx_status` 要读很多行。

注意：`rows` 是估算值，不一定等于真实返回行数。

### 2.4 成本

优化器会估算查询过程中各种操作的成本，可以先把成本理解为：

- 要读多少索引记录。
- 要读多少数据行。
- 要不要回表。
- 要不要排序。
- 能不能直接从索引返回。

比如这条查询：

```sql
select * from orders where user_id = 1111;
```

流程可能是：

1. 走 `idx_user_id`。
2. 找到少量 `user_id = 1111` 的索引记录。
3. 拿到主键 `id`。
4. 回到 `primary` 查整行。
5. 返回结果。

如果只命中了 3 行，回表 3 次，成本较小，还可以接受。

但如果是这条查询：

```sql
select * from orders where status = 'paid';
```

流程可能是：

1. 走 `idx_status`。
2. 找到大量 `paid` 索引记录。
3. 拿到大量主键 `id`。
4. 大量回到 `primary` 查整行。
5. 返回结果。

如果命中了 80 万行，那么成本就非常高。

### 2.5 排序和覆盖索引也会影响选择

如果排序或者分组能利用可用索引的最左前缀，MySQL 可能少做额外排序。比如有这个索引：

```sql
index idx_user_created (user_id, created_at)
```

查询：

```sql
select id, user_id, created_at
from orders
where user_id = 1111
order by created_at;
```

具体流程是：

1. `idx_user_created` 先按照 `user_id` 排。
2. 在 `user_id` 相同的范围内，又按照 `created_at` 排。
3. 所以找到 `user_id = 1111` 这一段时，`created_at` 本身就是有序的。
4. MySQL 不需要额外排序。

如果查询需要的列，某个索引里已经全部包含，MySQL 可以直接从索引里拿结果，不必再读完整数据行。

还是这条查询：

```sql
select id, user_id, created_at
from orders
where user_id = 1111
order by created_at;
```

`idx_user_created` 里已经有了 `user_id + created_at + 主键 id`，覆盖了需要查询的所有字段，所以它可以作为覆盖索引使用。

### 2.6 总结

优化器选择索引，大致看这些东西：

1. 统计信息：表和索引的大致数据分布。
2. 基数：索引列不同值多不多。
3. 选择性：这个条件能不能缩小范围。
4. 估算行数：大概要读多少行。
5. 成本：读索引、读数据行、回表、排序等代价。
6. 覆盖索引：能不能只读索引就返回结果。
7. 排序：索引顺序能不能顺便满足 `order by` 或 `group by`。

## 3. explain / explain analyze 怎么看

> The EXPLAIN statement provides information about how MySQL executes statements.
> Each output row from EXPLAIN provides information about one table.

`explain` 语句提供了 MySQL 如何执行语句的信息。`explain` 的每一行，描述的是 MySQL 访问某一张表的方式。

所以，**explain 不是查询结果，而是 MySQL 打算怎么读表的计划。**

依旧使用开头的表，执行这个语句：

```sql
explain select * from orders where user_id = 1111;
```

可以按照这个顺序看输出：

```text
type -> possible_keys -> key -> rows -> filtered -> extra
```

### 3.1 type：MySQL 用什么方式访问表

常见值可以先记这几个：

- `const`：通过主键或唯一索引，一次最多找到一行。
- `ref`：通过普通索引等值匹配，可能找到多行。
- `range`：通过索引做范围扫描。
- `index`：扫描整棵索引。
- `all`：全表扫描。

### 3.2 possible_keys 和 key

`possible_keys` 是候选索引，`key` 是实际选择的索引。

比如：

```text
possible_keys: idx_status
key: null
```

表示 `idx_status` 理论上可能会使用，但是优化器最后没选它。

再比如：

```text
possible_keys: idx_user_id,idx_user_created
key: idx_user_created
```

表示两个索引都可能有关，最后 MySQL 选了 `idx_user_created`。

### 3.3 rows：估算要读多少行

`rows` 是优化器估算要检查的行数。

比如：

```text
key: idx_status
rows: 800000
```

表示这个索引虽然被用了，但估算要检查 80 万行，索引收益可能不大。

所以 `rows` 和 `key` 要放到一起看。不是看到 `key` 有值就结束，还要关注它预计读多少行。

### 3.4 filtered：读出来以后还能留下多少

`filtered` 表示经过表条件过滤后，预计还有多少比例留下。

比如：

```text
rows: 10000
filtered: 10.00
```

可以这样计算：

```text
10000 * 10% = 1000
```

意思是：MySQL 估算先检查 10000 行，再经过条件过滤，大概剩 1000 行继续参与后面的执行。

### 3.5 extra：补充说明

重点看这几个：

```text
using where：还要用 where 条件过滤。
using index：覆盖索引，直接从索引拿到需要的列。
using filesort：需要额外排序。
using temporary：需要临时表。
```

比如：

```sql
explain
select id, user_id, created_at
from orders
where user_id = 1111
order by created_at;
```

如果使用 `idx_user_created`，并且查询列都在索引里，可能看到：

```text
key: idx_user_created
extra: using index
```

流程是：

1. `idx_user_created` 里有 `user_id`。
2. `idx_user_created` 里有 `created_at`。
3. InnoDB 二级索引里还有主键 `id`。
4. 查询只需要 `id`、`user_id`、`created_at`。
5. 所以不需要回表。

### 3.6 explain analyze 和 explain 的区别

> EXPLAIN ANALYZE runs a statement.

`explain analyze` 会真的执行语句，还会展示优化器预估和实际执行之间的关系，比如实际时间、实际行数、循环次数。

所以具体区别是：

- `explain`：只看执行计划，不执行查询。
- `explain analyze`：会执行语句，并给出实际执行信息。

## 4. 总结

**MySQL 会把索引当成候选路线，然后由优化器根据成本选择一条它认为更划算的执行方式。查询快不快，依靠多个因素共同作用**

索引真正有价值的前提，是它能明显缩小扫描范围。比如 `user_id` 这种基数高、选择性好的字段，通常更容易让 MySQL 走索引；而 `status` 这种只有少量取值的字段，如果某个值占了全表大部分数据，走索引反而可能要先扫大量二级索引，再大量回表，成本不一定比全表扫描低。

MySQL 选索引，本质是在统计信息基础上估算成本；选择性决定能少读多少行，覆盖索引决定能不能少回表，索引顺序决定能不能少排序，explain 用来观察优化器最后怎么选。
