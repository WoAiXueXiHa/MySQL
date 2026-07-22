# MySQL 内连接和外连接：从匹配关系到业务查询

在关系型数据库中，业务数据通常不会全部放在一张表里。用户、订单、任务、消息、日志等实体会被拆成多张表，再通过字段之间的关系关联起来。

`join` 解决的核心问题是：

```text
两张表中的哪一行，应该和哪一行拼接成一条查询结果？
```

这个“匹配规则”通常写在 `on` 后面。例如：

```sql
on tasks.id = outbox.task_id
```

本文通过一个任务表和一个 outbox 表，讲清楚 MySQL 中 `inner join` 和 `left join` 的原理、使用场景、主表和从表的判断方法，以及写查询时最容易出错的地方。

## 1. 前置准备

假设 LearnQ 中有一个任务系统：

- `tasks` 表保存任务本身。
- `outbox` 表保存需要投递到 redis 队列的任务消息。

这是一种常见的业务建模方式：核心业务数据放在主业务表中，消息投递、异步处理、补偿任务等信息放在辅助表中。

### 1.1 创建表

```sql
create table tasks (
    id bigint primary key auto_increment comment '任务id',
    title varchar(100) not null comment '任务标题',
    status varchar(32) not null comment '任务状态'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '任务表';

create table outbox (
    id bigint primary key auto_increment comment 'outbox记录id',
    task_id bigint not null comment '关联任务id',
    status varchar(32) not null comment '投递状态'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '任务投递表';
```

### 1.2 插入测试数据

```sql
insert into tasks (title, status) values
    ('learn mysql', 'pending'),
    ('write learnq', 'pending'),
    ('review net', 'succeed'),
    ('learn redis', 'pending');

insert into outbox (id, task_id, status) values
    (101, 1, 'sent'),
    (102, 2, 'pending'),
    (103, 99, 'pending');
```

这组数据中有三类情况：

| 情况 | 说明 |
| --- | --- |
| `task_id = 1` | outbox 记录能匹配到任务 1 |
| `task_id = 2` | outbox 记录能匹配到任务 2 |
| `task_id = 99` | outbox 记录找不到对应任务，是异常数据 |

而 `tasks` 表中的任务 3 和任务 4 暂时没有对应的 outbox 记录。

两张表的关系可以理解为：

```text
tasks.id = outbox.task_id
```

也就是：

- `tasks.id` 是任务自己的主键。
- `outbox.task_id` 表示这条 outbox 记录属于哪个任务。
- 一条任务可能有 outbox 记录，也可能没有。
- 一条 outbox 记录理论上应该能找到对应任务。

## 2. join 的基本原理

`join` 会根据指定条件，把多张表中的行组合成一张临时结果集。

MySQL 中，`join` 默认表示内连接，日常写业务查询时建议明确写出连接语义：需要两边都匹配时写 `inner join`，需要保留某一侧数据时写 `left join`。另外，`left outer join` 等价于 `left join`，`outer` 关键字通常可以省略。

可以先把它理解成三步：

1. 从第一张表取出一行。
2. 到第二张表中寻找满足 `on` 条件的行。
3. 按连接类型决定是否保留没有匹配成功的行。

连接类型决定“匹配不上时怎么办”：

| 连接类型 | 匹配上的行 | 左表没匹配上 | 右表没匹配上 |
| --- | --- | --- | --- |
| `inner join` | 保留 | 不保留 | 不保留 |
| `left join` | 保留 | 保留，右表列补 `null` | 不单独保留 |

所以，学习连接查询时要抓住两个问题：

- `on` 后面写什么：决定两张表如何匹配。
- 使用哪种 `join`：决定匹配失败的数据是否保留。

## 3. 如何区分主表和从表

在 SQL 查询里，主表通常不是绝对的，而是由查询目标决定。

可以按下面的方法判断：

### 3.1 先问：这次查询以谁为中心？

如果问题是“查所有任务，以及它们有没有 outbox 记录”，那么中心是任务：

```text
主表：tasks
从表：outbox
```

适合使用：

```sql
from tasks t
left join outbox o on t.id = o.task_id
```

如果问题是“查所有 outbox 记录，以及它们对应的任务是否存在”，那么中心是 outbox：

```text
主表：outbox
从表：tasks
```

适合使用：

```sql
from outbox o
left join tasks t on o.task_id = t.id
```

### 3.2 再问：哪一边的数据必须保留？

`left join` 的左表会全部保留。因此，谁是“必须保留的一方”，谁就应该放在 `left join` 的左边。

常见判断方式：

| 查询目标 | 应保留的数据 | 推荐写法 |
| --- | --- | --- |
| 查所有任务及其投递状态 | 所有任务 | `tasks left join outbox` |
| 查没有 outbox 的任务 | 所有任务，再筛出右表为空 | `tasks left join outbox where outbox.id is null` |
| 查孤立 outbox 记录 | 所有 outbox，再筛出右表为空 | `outbox left join tasks where tasks.id is null` |
| 查两边都存在的数据 | 只保留匹配成功的行 | `inner join` |

### 3.3 主表和从表不等于父表和子表

在建模关系中，`tasks` 通常是父表，`outbox` 通常是子表，因为 `outbox.task_id` 引用 `tasks.id`。

但在查询中，谁是主表取决于这次查询要保留谁：

- 查任务视角，`tasks` 是查询主表。
- 查 outbox 异常记录，`outbox` 是查询主表。

所以不要死记“父表一定是主表”。更准确的判断是：这次查询不能丢哪张表的数据。

## 4. inner join：只保留两边都匹配的数据

`inner join` 表示内连接。它只返回两边都能匹配上的行。

```sql
select
    t.id as task_id,
    t.title,
    t.status as task_status,
    o.id as outbox_id,
    o.status as outbox_status
from tasks t
inner join outbox o
    on t.id = o.task_id;
```

这个查询只会返回任务 1 和任务 2，因为只有它们满足：

```sql
t.id = o.task_id
```

任务 3、任务 4 没有 outbox 记录，不会出现在结果中。`outbox.task_id = 99` 找不到对应任务，也不会出现在结果中。

`inner join` 适合回答这类问题：

- 哪些任务已经生成了 outbox 记录？
- 哪些订单有对应的支付记录？
- 哪些用户已经提交了资料？
- 只关心两边关系都成立的数据。

可以把 `inner join` 理解为集合里的“交集”。

```text
tasks 和 outbox 都能匹配上的部分
```

## 5. left join：保留左表全部数据

`left join` 表示左外连接。它会保留左表的全部行，右表能匹配就拼接，匹配不上就用 `null` 填充右表字段。

```sql
select
    t.id as task_id,
    t.title,
    t.status as task_status,
    o.id as outbox_id,
    o.status as outbox_status
from tasks t
left join outbox o
    on t.id = o.task_id;
```

这条 sql 的语义是：

```text
以 tasks 为主，查询每个任务对应的 outbox 状态。
```

结果中会保留所有任务：

- 任务 1 匹配 outbox 101。
- 任务 2 匹配 outbox 102。
- 任务 3 没有 outbox，右表字段为 `null`。
- 任务 4 没有 outbox，右表字段为 `null`。

`left join` 适合回答这类问题：

- 查所有任务，以及它们是否已经生成 outbox。
- 查所有用户，以及他们是否有订单。
- 查所有课程，以及每门课是否有人选。
- 查主业务数据，同时补充可选关联信息。

## 6. left join 查缺失数据

外连接最常见的使用场景之一，是查“左表有、右表没有”的数据。

### 6.1 查任务有，但 outbox 没有

问题：哪些任务已经创建，但还没有 outbox 记录？

```sql
select
    t.id,
    t.title,
    t.status
from tasks t
left join outbox o
    on t.id = o.task_id
where o.id is null;
```

这类查询的关键是：

```sql
left join ... where right_table.id is null
```

含义是：

1. 先保留左表全部数据。
2. 右表能匹配就拼上。
3. 最后筛出右表没有匹配成功的行。

这个写法常用于：

- 查未生成 outbox 的业务记录。
- 查没有订单的用户。
- 查没有明细的主单。
- 查缺少配置项的数据。

### 6.2 查 outbox 有，但任务没有

问题：哪些 outbox 记录指向了不存在的任务？

这时查询目标变成了 outbox，所以要把 outbox 放在左边：

```sql
select
    o.id as outbox_id,
    o.task_id,
    o.status as outbox_status
from outbox o
left join tasks t
    on o.task_id = t.id
where t.id is null;
```

这类数据通常是异常数据，也可以叫孤立记录。出现这种情况的原因可能是：

- 没有外键约束。
- 删除了任务，但没有清理 outbox。
- 程序写入顺序有问题。
- 数据修复或导入时破坏了一致性。

这也是判断主表的一个典型例子：虽然建模上 `tasks` 是父表，但这次要查的是“异常 outbox 记录”，所以查询主表应该是 `outbox`。

## 7. 统一使用 left join：通过交换表顺序保留另一侧数据

日常写连接查询时，可以统一使用 `left join`。如果你想保留另一张表的全部数据，不需要换一种连接写法，只需要调整 `from` 后面的表顺序。

例如，查询“所有任务及其 outbox 状态”时，要保留的是 `tasks`，所以把 `tasks` 放在左边：

```sql
select
    t.id as task_id,
    t.title,
    t.status as task_status,
    o.id as outbox_id,
    o.status as outbox_status
from tasks t
left join outbox o
    on t.id = o.task_id;
```

如果问题变成“查询所有 outbox 记录及其对应任务”，要保留的是 `outbox`，那就把 `outbox` 放在左边：

```sql
select
    o.id as outbox_id,
    o.task_id,
    o.status as outbox_status,
    t.title,
    t.status as task_status
from outbox o
left join tasks t
    on o.task_id = t.id;
```

这就是统一使用 `left join` 的核心方法：

```text
谁的数据必须保留，谁就放在 left join 的左边。
```

这样做的好处是阅读顺序稳定：先看主查询对象，再看它关联了哪些辅助信息。团队中所有人都按这个规则写，连接查询会更容易维护。

## 8. on 和 where 的区别

连接查询中，`on` 和 `where` 都能写条件，但它们含义不同：

- `on`：描述两张表如何匹配。
- `where`：在连接结果出来后，再过滤结果行。

对于 `inner join`，很多条件写在 `on` 或 `where` 中，结果可能一样。但对于 `left join`，位置不同会直接改变语义。

### 8.1 正确保留左表全部数据

查询所有任务，并只拼接投递状态为 `pending` 的 outbox：

```sql
select
    t.id,
    t.title,
    o.id as outbox_id,
    o.status as outbox_status
from tasks t
left join outbox o
    on t.id = o.task_id
   and o.status = 'pending'
```

这里 `o.status = 'pending'` 写在 `on` 中，表示只让 pending 状态的 outbox 参与匹配。即使某个任务没有 pending outbox，任务本身仍然会保留。

### 8.2 容易误写成 inner join 的情况

如果把右表条件写到 `where` 中：

```sql
select
    t.id,
    t.title,
    o.id as outbox_id,
    o.status as outbox_status
from tasks t
left join outbox o
    on t.id = o.task_id
where o.status = 'pending';
```

这条 SQL 会过滤掉右表为 `null` 的行，因为 `null = 'pending'` 不成立。结果上，它会丢掉没有 outbox 的任务，很多时候就不再是你想要的“保留左表全部数据”。

判断规则：

- 关联关系写在 `on`。
- 右表的可选匹配条件，如果不想破坏左表保留语义，也写在 `on`。
- 对最终结果的整体过滤，写在 `where`。

## 9. 多表连接的写法

实际业务中经常不止两张表。多表连接可以连续写多个 `join`。

假设还有一张任务执行日志表：

```sql
create table task_logs (
    id bigint primary key auto_increment comment '日志id',
    task_id bigint not null comment '任务id',
    message varchar(200) not null comment '日志内容'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '任务日志表';
```

查询任务、outbox 和日志：

```sql
select
    t.id as task_id,
    t.title,
    o.status as outbox_status,
    l.message
from tasks t
left join outbox o
    on t.id = o.task_id
left join task_logs l
    on t.id = l.task_id;
```

多表连接时，要特别注意结果行数可能膨胀。

如果一个任务有 2 条 outbox 记录，又有 3 条日志记录，那么连接后可能得到 `2 * 3 = 6` 行。这不是 MySQL 算错了，而是多对多组合导致的结果。

如果只想统计数量，通常需要先聚合，再连接，或者使用子查询把一侧压缩成一行。

## 10. 同时保留两边数据的写法

MySQL 没有直接提供 `full outer join` 语法。也就是说，不能直接写：

```sql
select ...
from tasks t
full outer join outbox o
    on t.id = o.task_id;
```

如果确实需要同时保留两边所有数据，可以用两次 `left join` 通过 `union all` 合并：

```sql
select
    t.id as task_id,
    t.title,
    o.id as outbox_id,
    o.task_id
from tasks t
left join outbox o
    on t.id = o.task_id

union all

select
    t.id as task_id,
    t.title,
    o.id as outbox_id,
    o.task_id
from outbox o
left join tasks t
    on o.task_id = t.id
where t.id is null;
```

第一段查询保留所有任务，第二段查询只补充“outbox 有、tasks 没有”的孤立记录。这样既能表达同时保留两边数据的需求，又能保持全文统一使用 `left join`。

不过，大多数业务场景并不需要这种写法。更常见的做法是先明确查询主表，然后使用 `left join` 查询主表及其关联信息。

## 11. 常见使用场景

### 11.1 查关联详情

例如查询任务和它的 outbox 投递状态：

```sql
select
    t.id,
    t.title,
    o.status as outbox_status
from tasks t
inner join outbox o
    on t.id = o.task_id;
```

适合只关心两边都存在的数据。

### 11.2 查主表列表并补充状态

例如查询所有任务，同时展示是否有 outbox：

```sql
select
    t.id,
    t.title,
    t.status as task_status,
    o.status as outbox_status
from tasks t
left join outbox o
    on t.id = o.task_id;
```

适合列表页、管理后台、数据看板。

### 11.3 查缺失关系

例如查没有 outbox 的任务：

```sql
select
    t.id,
    t.title
from tasks t
left join outbox o
    on t.id = o.task_id
where o.id is null;
```

适合数据补偿、异常扫描、任务修复。

### 11.4 查孤立数据

例如查 outbox 指向了不存在任务的记录：

```sql
select
    o.id,
    o.task_id,
    o.status
from outbox o
left join tasks t
    on o.task_id = t.id
where t.id is null;
```

适合巡检脏数据、清理历史数据、验证迁移结果。

## 12. 编写 join 查询的建议

写连接查询时，可以按下面的顺序思考：

1. 明确这次查询要回答什么问题。
2. 判断哪张表的数据必须保留。
3. 把必须保留的数据放在 `left join` 的左边。
4. 把表之间的匹配关系写在 `on` 中。
5. 把最终结果过滤条件写在 `where` 中。
6. 如果要查缺失关系，用 `left join ... where right_table.id is null`。

还要注意几条实践原则：

- 表别名要简短但清晰，例如 `tasks t`、`outbox o`。
- 查询列要显式写出，不建议在连接查询中使用 `select *`。
- 多张表有同名字段时，要用表别名限定字段来源。
- 用 `left join` 时，谨慎把右表条件写进 `where`。
- 连接字段应该建立合适索引，例如 `outbox.task_id`。

对于本文中的关系，建议给 `outbox.task_id` 建索引：

```sql
create index idx_outbox_task_id on outbox (task_id);
```

这样 MySQL 在根据 `tasks.id = outbox.task_id` 查找匹配行时，更容易利用索引减少扫描成本。

## 13. 小结

`join` 的核心不是记住语法，而是理解表之间的匹配关系。

- `inner join`：只保留两边都匹配成功的数据。
- `left join`：保留左表全部数据，右表匹配不上就补 `null`。
- `on` 决定两张表如何匹配，`where` 决定最终结果如何过滤。
- 判断主表时，不要只看父表和子表，而要看这次查询必须保留哪张表的数据。

实际开发中，最常用的组合是 `inner join` 和 `left join`。前者用于查询关系已经成立的数据，后者用于保留主业务数据并补充可选关联信息，或者反过来查缺失、查异常。
