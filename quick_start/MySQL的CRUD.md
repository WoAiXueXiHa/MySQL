# MySQL 的 CRUD：从基础操作到聚合统计

CRUD 是数据库最常见的四类数据操作：

- `create`：新增数据，对应 `insert`。
- `retrieve`：查询数据，对应 `select`。
- `update`：更新数据，对应 `update`。
- `delete`：删除数据，对应 `delete`。

本文围绕一组简单的学生成绩表，系统介绍 MySQL 中 CRUD 的基础语法、常见使用方式和需要注意的边界问题。文中的 sql 语句统一使用小写风格。

## 1. create：新增数据

在插入数据之前，先创建一张学生信息表：

```sql
create table stu (
    id int primary key auto_increment comment '主键id',
    sn int not null unique comment '学号',
    name varchar(20) not null comment '姓名',
    qq varchar(20) default null comment 'qq号'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '学生信息表';
```

### 1.1 全列插入

全列插入要求 `values` 中的值和表字段顺序一一对应：

```sql
insert into stu values (100, 1000, 'aaa', null);
insert into stu values (101, 1001, 'bbb', '123456');
```

全列插入的问题是对表结构顺序依赖较强。如果后续新增字段、调整字段顺序，sql 维护成本会变高。

### 1.2 指定列插入

更推荐的写法是指定列名：

```sql
insert into stu (sn, name, qq) values
    (1002, 'ccc', '741852'),
    (1003, 'ddd', null);
```

指定列插入有两个好处：

- sql 含义更清晰，可读性更好。
- 可以省略有默认值、自增长或允许为 `null` 的字段。

查询插入结果：

```sql
select id, sn, name, qq from stu;
```

### 1.3 插入时处理唯一键冲突

如果插入的数据违反主键或唯一键约束，普通 `insert` 会失败。例如，`id` 或 `sn` 已存在时，再插入相同值就会触发唯一冲突。

如果希望在唯一键冲突时改为更新已有记录，可以使用 `insert ... on duplicate key update`：

```sql
insert into stu (id, sn, name)
values (100, 10010, 'asd')
on duplicate key update
    sn = 10010,
    name = 'asd';
```

这个语句的含义是：

- 如果没有主键或唯一键冲突，就执行插入。
- 如果发生主键或唯一键冲突，就执行后面的更新。

可以通过 `row_count()` 查看影响行数：

```sql
select row_count();
```

常见结果含义：

- 返回 `1`：插入了一行新数据。
- 返回 `2`：发生冲突，并更新了已有数据。
- 返回 `0`：发生冲突，但更新后的值和原值相同。

### 1.4 replace into

`replace into` 也可以处理主键或唯一键冲突：

```sql
replace into stu (sn, name) values (2001, 'asdsad');
replace into stu (sn, name) values (2001, 'aasda');
```

它的规则是：

- 如果没有主键或唯一键冲突，直接插入。
- 如果发生主键或唯一键冲突，先删除旧记录，再插入新记录。

因此，`replace into` 和 `insert ... on duplicate key update` 并不完全等价。`replace into` 是删除后插入，可能影响自增长值、触发器、外键关系和字段默认值。日常业务中，如果只是希望更新已有行，通常优先使用 `insert ... on duplicate key update`。

## 2. retrieve：查询数据

查询使用 `select`。为了演示查询语法，先创建一张成绩表：

```sql
create table exam (
    id int primary key auto_increment comment '主键id',
    name varchar(20) not null comment '姓名',
    chinese decimal(5, 2) not null default 0.00 comment '语文成绩',
    math decimal(5, 2) not null default 0.00 comment '数学成绩',
    english decimal(5, 2) not null default 0.00 comment '英语成绩'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '考试成绩表';
```

插入测试数据：

```sql
insert into exam (name, chinese, math, english) values
    ('zhangsan', 67, 98, 56),
    ('lisi', 87, 78, 77),
    ('wangwu', 88, 98, 90),
    ('zhaoliu', 82, 84, 67),
    ('tianqi', 55, 85, 45),
    ('kunkun', 70, 73, 78),
    ('kunge', 75, 65, 30);
```

### 2.1 查询列

查询所有列：

```sql
select * from exam;
```

日常开发中不建议长期依赖 `select *`：

- 查询列越多，网络传输和内存开销越大。
- 表结构变化后，结果列也会变化，容易影响应用代码。
- 在某些场景下，可能无法充分利用覆盖索引。

更推荐明确指定需要的列：

```sql
select id, name, english from exam;
```

查询列也可以是表达式：

```sql
select id, name, 100 from exam;

select id, name, 100 + math from exam;

select id, name, chinese + math + english as total from exam;
```

`as total` 表示给表达式结果起别名。`as` 可以省略，但保留 `as` 通常更清晰。

结果去重使用 `distinct`：

```sql
select distinct math from exam;
```

`distinct` 会对查询结果中的整行组合去重，而不是只对某一个字段单独去重。例如：

```sql
select distinct name, math from exam;
```

这表示按 `name` 和 `math` 的组合去重。

### 2.2 where 条件

`where` 用于过滤行。常见条件运算符如下：

| 运算符 | 含义 | 示例 |
| --- | --- | --- |
| `=` | 等于 | `name = 'lisi'` |
| `<>` 或 `!=` | 不等于 | `math <> 100` |
| `>`、`>=`、`<`、`<=` | 大小比较 | `english < 60` |
| `between ... and ...` | 闭区间范围 | `chinese between 80 and 90` |
| `in (...)` | 在集合中 | `math in (58, 59, 98, 99)` |
| `like` | 模糊匹配 | `name like 'kun%'` |
| `is null` | 判断为空 | `qq is null` |
| `is not null` | 判断不为空 | `qq is not null` |
| `<=>` | null 安全等于 | `qq <=> null` |

查询英语不及格的同学：

```sql
select name, english
from exam
where english < 60;
```

查询语文成绩在 `[80, 90]` 的同学：

```sql
select name, chinese
from exam
where chinese between 80 and 90;
```

查询数学成绩是 `58`、`59`、`98` 或 `99` 的同学：

```sql
select name, math
from exam
where math in (58, 59, 98, 99);
```

查询姓 `kun` 的同学：

```sql
select name
from exam
where name like 'kun%';
```

`like` 中常用两个通配符：

- `%`：匹配任意长度的任意字符。
- `_`：匹配一个任意字符。

例如：

```sql
select name from exam where name like 'kun%';
select name from exam where name like 'kun__';
```

使用 `like` 时要注意索引效率。`name like 'kun%'` 这种前缀匹配通常有机会使用索引；`name like '%kun'` 或 `name like '%kun%'` 因为前面是不确定内容，通常难以直接利用普通 b-tree 索引。

查询语文成绩好于英语成绩的同学：

```sql
select name, chinese, english
from exam
where chinese > english;
```

查询总分在 200 分以下的同学：

```sql
select name, chinese + math + english as total
from exam
where chinese + math + english < 200;
```

查询语文成绩大于 80，并且不姓 `kun` 的同学：

```sql
select name, chinese
from exam
where chinese > 80
  and name not like 'kun%';
```

复杂条件建议用括号明确优先级。例如，查询姓 `kun` 的同学，或者总分大于 200、语文低于数学且英语大于 80 的同学：

```sql
select name, chinese + math + english as total
from exam
where name like 'kun%'
   or (
        chinese + math + english > 200
        and chinese < math
        and english > 80
   );
```

#### null 的比较

`null` 表示未知值，它不能用普通的 `=` 或 `!=` 判断。

错误示例：

```sql
select name, qq from stu where qq = null;
select name, qq from stu where qq != null;
```

正确写法：

```sql
select name, qq from stu where qq is null;
select name, qq from stu where qq is not null;
```

如果需要 null 安全等值比较，可以使用 `<=>`：

```sql
select name, qq from stu where qq <=> null;
```

### 2.3 order by 排序

排序使用 `order by`：

```sql
select name, math
from exam
order by math asc;
```

说明：

- `asc` 表示升序，也是默认排序方式。
- `desc` 表示降序。
- 没有 `order by` 的查询，返回顺序是未定义的，不要依赖默认顺序。

按数学成绩降序：

```sql
select name, math
from exam
order by math desc;
```

多字段排序：

```sql
select name, math, english, chinese
from exam
order by math desc, english asc, chinese asc;
```

上面的排序规则是：先按数学降序；数学相同，再按英语升序；英语也相同，再按语文升序。

按总分从高到低：

```sql
select name, chinese + math + english as total
from exam
order by total desc;
```

MySQL 允许在 `order by` 中使用查询列别名。

### 2.4 limit 分页

`limit` 用于限制返回行数，常用于分页。

查询前 3 条：

```sql
select name, chinese + math + english as total
from exam
order by total desc
limit 3;
```

从第 `offset` 行开始，查询 `size` 条：

```sql
select name, chinese + math + english as total
from exam
order by total desc
limit 3 offset 2;
```

等价写法：

```sql
select name, chinese + math + english as total
from exam
order by total desc
limit 2, 3;
```

需要注意：

- `offset` 从 0 开始。
- `limit 2, 3` 表示跳过 2 条，再取 3 条。
- 分页必须配合稳定的 `order by`，否则翻页结果可能不稳定。
- 大偏移分页性能可能较差，例如 `limit 100000, 20`，后续可以考虑基于主键或游标的分页方式。

### 2.5 聚合函数

聚合函数用于对多行数据进行统计，常见函数如下：

| 函数 | 作用 |
| --- | --- |
| `count()` | 统计行数 |
| `sum()` | 求和 |
| `avg()` | 求平均值 |
| `max()` | 求最大值 |
| `min()` | 求最小值 |

统计学生总数：

```sql
select count(*) as student_count
from exam;
```

统计数学成绩非空的行数：

```sql
select count(math) as math_count
from exam;
```

也可以写成：

```sql
select count(1) as row_count
from exam;
```

`count(*)` 和 `count(1)` 都常用于统计结果行数。`count(column)` 统计该列不为 `null` 的行数，如果字段允许为 `null`，它的结果可能小于 `count(*)`。

例如，统计已经收集到 qq 号的学生数量：

```sql
select count(qq) as qq_count
from stu;
```

查询数学最高分、最低分和平均分：

```sql
select
    max(math) as max_math,
    min(math) as min_math,
    avg(math) as avg_math
from exam;
```

统计全班总分之和：

```sql
select sum(chinese + math + english) as class_total
from exam;
```

查询数学成绩大于 70 的最低分：

```sql
select min(math) as min_math
from exam
where math > 70;
```

聚合函数会把多行压缩成一个统计结果。如果查询列表里同时出现普通列和聚合函数，通常需要配合 `group by` 明确分组规则。

例如，下面这种写法在语义上是不清晰的：

```sql
select name, avg(math)
from exam;
```

`avg(math)` 是全表聚合结果，但 `name` 来自具体某一行。开启 `only_full_group_by` 后，这类查询会直接报错。正确做法是明确分组字段，或者只查询聚合结果。

### 2.6 group by 分组

`group by` 用于按一个或多个字段分组，然后对每组分别执行聚合计算。

为了演示分组，先创建一张课程成绩表：

```sql
create table course_score (
    id bigint primary key auto_increment comment '主键id',
    class_name varchar(20) not null comment '班级名称',
    student_name varchar(20) not null comment '学生姓名',
    course_name varchar(20) not null comment '课程名称',
    score decimal(5, 2) not null default 0.00 comment '成绩'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '课程成绩表';
```

插入测试数据：

```sql
insert into course_score (class_name, student_name, course_name, score) values
    ('一班', 'zhangsan', '语文', 67),
    ('一班', 'zhangsan', '数学', 98),
    ('一班', 'lisi', '语文', 87),
    ('一班', 'lisi', '数学', 78),
    ('二班', 'wangwu', '语文', 88),
    ('二班', 'wangwu', '数学', 98),
    ('二班', 'zhaoliu', '语文', 82),
    ('二班', 'zhaoliu', '数学', 84);
```

按班级统计人数：

```sql
select class_name, count(distinct student_name) as student_count
from course_score
group by class_name;
```

按班级统计平均分：

```sql
select class_name, avg(score) as avg_score
from course_score
group by class_name;
```

按班级和课程统计平均分：

```sql
select class_name, course_name, avg(score) as avg_score
from course_score
group by class_name, course_name;
```

`where` 和 `having` 的区别：

- `where` 在分组前过滤原始行。
- `having` 在分组后过滤聚合结果。

例如，先过滤数学课程，再按班级统计数学平均分：

```sql
select class_name, avg(score) as avg_math_score
from course_score
where course_name = '数学'
group by class_name;
```

查询平均分大于 85 的班级：

```sql
select class_name, avg(score) as avg_score
from course_score
group by class_name
having avg(score) > 85;
```

查询结果按平均分降序：

```sql
select class_name, avg(score) as avg_score
from course_score
group by class_name
having avg(score) > 80
order by avg_score desc;
```

再看一个更接近业务状态统计的例子。先创建任务表：

```sql
create table tasks (
    id bigint primary key auto_increment comment '任务id',
    title varchar(100) not null comment '任务标题',
    status varchar(32) not null comment '任务状态',
    retry_count int not null default 0 comment '重试次数',
    created_at datetime default null comment '创建时间'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '任务表';
```

插入测试数据：

```sql
insert into tasks (title, status, retry_count) values
    ('learn go', 'pending', 0),
    ('learn mysql', 'pending', 1),
    ('learn redis', 'succeed', 0),
    ('leetcode', 'failed', 3),
    ('review net', 'succeed', 0);
```

统计每种任务状态下有多少任务：

```sql
select status, count(*) as task_count
from tasks
group by status;
```

统计每种任务状态下的最大重试次数：

```sql
select status, max(retry_count) as max_retry_count
from tasks
group by status;
```

筛选出任务数量大于 1 的状态：

```sql
select status, count(*) as task_count
from tasks
group by status
having count(*) > 1;
```

使用 `group by` 时要注意：查询列中如果出现非聚合字段，这些字段通常应出现在 `group by` 中。否则结果语义不明确，在开启 `only_full_group_by` 的 MySQL 环境中会直接报错。

### 2.7 select 的书写顺序

一个比较完整的查询语句通常按照下面的顺序书写：

```sql
select column_list
from table_name
where condition
group by group_column
having group_condition
order by sort_column
limit row_count offset offset_count;
```

各部分的作用可以概括为：

- `from`：确定从哪张表读取数据。
- `where`：过滤原始行。
- `group by`：对过滤后的数据分组。
- `having`：过滤分组后的聚合结果。
- `select`：决定最终返回哪些列或表达式。
- `order by`：对结果排序。
- `limit`：限制返回数量。

实际执行过程和书写顺序并不完全一致，但按这个结构组织 sql，通常最容易阅读和维护。

## 3. update：更新数据

`update` 用于修改已有数据。

基本语法：

```sql
update table_name
set column_name = expr [, column_name = expr] ...
[where condition]
[order by ...]
[limit row_count];
```

### 3.1 按条件更新

将 `kunkun` 的数学成绩改为 80，语文成绩改为 100：

```sql
select name, math, chinese
from exam
where name = 'kunkun';

update exam
set math = 80,
    chinese = 100
where name = 'kunkun';

select name, math, chinese
from exam
where name = 'kunkun';
```

更新前先查询目标数据，是一个很重要的习惯。尤其是生产环境中，执行 `update` 前应先用相同的 `where` 条件执行 `select`，确认影响范围。

### 3.2 基于原值更新

将总成绩倒数前三的同学数学成绩加 30：

```sql
select name, math, chinese + math + english as total
from exam
order by total asc
limit 3;

update exam
set math = math + 30
order by chinese + math + english asc
limit 3;
```

这里的 `math = math + 30` 是基于原值更新。执行后，原来的总分排序会发生变化，所以再次查询倒数前三时，结果可能已经不是同一批学生。

### 3.3 全表更新

如果没有 `where` 条件，`update` 会影响整张表：

```sql
update exam
set chinese = chinese * 2;
```

全表更新是高风险操作。执行前应确认是否真的需要更新所有行，必要时先备份数据或在事务中操作。

## 4. delete：删除数据

`delete` 用于删除表中的行。

基本语法：

```sql
delete from table_name
[where condition]
[order by ...]
[limit row_count];
```

### 4.1 按条件删除

删除英语成绩不及格的学生：

```sql
select id, name, english
from exam
where english < 60;

delete from exam
where english < 60;
```

和 `update` 一样，执行 `delete` 前建议先用相同条件执行 `select`，确认要删除的数据是否正确。

### 4.2 限制删除数量

如果只想删除部分数据，可以配合 `order by` 和 `limit`：

```sql
delete from exam
order by chinese + math + english asc
limit 1;
```

这条 sql 表示删除总分最低的一名学生。使用 `limit` 时建议配合明确的 `order by`，否则删除哪几行可能不稳定。

### 4.3 全表删除

如果没有 `where` 条件，`delete` 会删除表中所有数据：

```sql
delete from exam;
```

这不会删除表结构，但会删除表里的全部行。生产环境中执行前必须确认库名、表名、条件、备份和事务策略。

如果只是业务上不再展示某条数据，更常见的做法是软删除，也就是用字段标记删除状态，而不是直接物理删除：

```sql
alter table stu
add deleted tinyint(1) not null default 0 comment '是否删除';

update stu
set deleted = 1
where id = 100;
```

软删除可以保留历史数据，便于审计和恢复。但它也要求所有查询都正确过滤删除状态，例如：

```sql
select id, sn, name
from stu
where deleted = 0;
```

### 4.4 delete、truncate 和 drop 的区别

三者都可能让数据消失，但语义不同：

| 语句 | 作用 | 是否保留表结构 |
| --- | --- | --- |
| `delete from table_name where ...` | 删除满足条件的行 | 保留 |
| `truncate table table_name` | 清空整张表 | 保留 |
| `drop table table_name` | 删除整张表 | 不保留 |

示例：

```sql
truncate table exam;

drop table exam;
```

一般来说：

- `delete` 用于按条件删除数据。
- `truncate` 用于快速清空整张表。
- `drop` 用于删除表结构和数据。

`truncate` 和 `drop` 都是高风险操作，通常不应该在没有确认和备份的情况下执行。`truncate` 面向整表清空，不能加 `where` 条件；`drop` 会删除表结构，影响更大。

## 5. 小结

MySQL 的 CRUD 可以概括为：

- `insert` 负责新增数据，推荐指定列插入。
- `select` 负责查询数据，核心能力包括列选择、条件过滤、排序、分页、聚合和分组。
- `update` 负责修改数据，执行前要确认 `where` 条件。
- `delete` 负责删除数据，执行前要先查询确认影响范围。

日常写 sql 时，要特别注意三点：不要依赖无序查询的默认返回顺序，不要把 `null` 当成普通值比较，不要在没有确认条件的情况下执行全表 `update` 或 `delete`。
