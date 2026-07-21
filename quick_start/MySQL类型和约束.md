# MySQL 数据类型和约束：从字段设计到表结构建模

设计 MySQL 表结构时，数据类型和约束是最基础、也最容易影响后续维护成本的部分。

数据类型决定一列可以存什么、能存多大、精度如何计算、占用多少空间，以及是否适合建立索引。约束则用于限制数据的合法性，例如字段不能为空、主键不能重复、外键必须引用已存在的数据。

本文从实际业务建模角度出发，介绍 MySQL 中常用的数据类型和约束，并给出一份完整建表示例。

## 1. 数据类型

学习 MySQL 数据类型时，不建议只靠背语法。更重要的是理解业务需求：这一列保存的是什么数据，未来可能增长到多大，是否需要精确计算，是否经常用于查询和索引。

可以先建立一个基本认知：

```text
数据类型 = 给字段定规则
```

它决定了字段的以下行为：

- 能保存什么类型的数据。
- 能保存多大的数据。
- 数值精度如何处理。
- 底层大致占用多少空间。
- 是否适合排序、比较和建立索引。

### 1.1 整数类型

整数适合保存 ID、数量、次数、状态码、开关标记等不需要小数的数据。

MySQL 常见整数类型如下：

| 类型 | 存储空间 | 有符号范围 | 无符号范围 |
| --- | --- | --- | --- |
| `tinyint` | 1 字节 | -128 到 127 | 0 到 255 |
| `smallint` | 2 字节 | -32768 到 32767 | 0 到 65535 |
| `mediumint` | 3 字节 | -8388608 到 8388607 | 0 到 16777215 |
| `int` | 4 字节 | 约 -21 亿到 21 亿 | 0 到约 42 亿 |
| `bigint` | 8 字节 | 极大范围 | 极大范围 |

常见选择可以参考：

| 场景 | 常用类型 | 说明 |
| --- | --- | --- |
| 用户 ID、订单 ID、任务 ID | `bigint` | 增长空间更大，适合长期业务 |
| 次数、分钟数、库存数量 | `int` | 普通业务通常够用 |
| 状态码、小枚举编号 | `tinyint` | 取值范围小，表达清晰 |
| 是否删除、是否启用 | `tinyint(1)` 或 `boolean` | MySQL 中 `boolean` 是 `tinyint(1)` 的别名 |

示例：

```sql
create table tasks (
    id bigint primary key auto_increment comment '任务ID',
    retry_count int not null default 0 comment '重试次数',
    status tinyint not null default 0 comment '任务状态',
    deleted tinyint(1) not null default 0 comment '是否删除'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

需要注意，`int(11)`、`tinyint(1)` 中的数字不是取值范围。整数类型的范围由类型本身和是否 `unsigned` 决定。对于普通业务，优先根据数据增长规模选择 `int` 或 `bigint`，不要把显示宽度误认为容量。

`unsigned` 表示无符号，只能存非负数。它可以扩大正数范围，但也会带来一些类型转换和边界问题。对 ID 类字段，如果不确定未来规模，直接使用 `bigint` 通常更稳妥。

### 1.2 小数类型

小数类型要先区分一个关键问题：是否需要精确计算。

| 场景 | 推荐类型 | 原因 |
| --- | --- | --- |
| 价格、余额、订单金额 | `decimal(m,d)` | 精确小数，适合金额 |
| 学习时长、评分、温度、概率、模型相似度 | `double` | 近似值，适合测量和科学计算 |
| 不需要小数的钱 | `bigint` | 用整数保存最小单位，例如分 |

`float` 和 `double` 是浮点数，保存的是近似值，不适合金额这类需要精确计算的数据。比如订单金额、账户余额、支付流水金额，应优先使用 `decimal`，或者使用整数保存最小货币单位。

`decimal(10,2)` 的含义是：

- 最多保存 10 位数字。
- 小数点后保留 2 位。
- 整数部分最多 8 位。
- 可表达范围大致为 `-99999999.99` 到 `99999999.99`。

示例：

```sql
create table orders (
    id bigint primary key auto_increment comment '订单ID',
    amount decimal(10, 2) not null comment '订单金额',
    similarity double default null comment '模型相似度'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

金额字段不要使用 `double` 进行精确比较。例如 `amount = 19.99` 这类判断，在浮点数场景中可能因为近似存储出现非预期结果。

### 1.3 字符串类型

字符串适合保存名称、编码、手机号、邮箱、地址、描述、正文等文本数据。

常见字符串类型如下：

| 类型 | 特点 | 适合场景 |
| --- | --- | --- |
| `char(n)` | 固定长度 | 固定长度编码、国家码、性别编码 |
| `varchar(n)` | 可变长度 | 用户名、标题、手机号、邮箱 |
| `text` | 长文本 | 文章正文、备注、详情描述 |

`char` 是固定长度。例如 `char(10)` 即使只保存 3 个字符，也会按固定长度处理。它适合长度稳定的数据，例如固定长度编码。

`varchar` 是可变长度。例如 `varchar(100)` 表示最多保存 100 个字符，实际占用空间与写入内容长度有关。它适合大多数普通短文本字段。

需要特别注意：`varchar(n)` 中的 `n` 表示字符数，不是字节数。但 MySQL 单行数据有最大长度限制，字段实际可声明的最大长度会受到字符集、其他字段、是否允许 `null`、行格式等因素影响。

以 `utf8mb4` 为例，一个字符最多可能占用 4 个字节。如果声明 `varchar(20000)`，理论字符数很多，但单行最大长度限制可能导致建表失败。日常业务中，应根据真实业务长度设置合理范围，而不是把所有短文本都写成超大 `varchar`。

示例：

```sql
create table users (
    id bigint primary key auto_increment comment '用户ID',
    username varchar(64) not null comment '用户名',
    phone char(11) default null comment '手机号',
    email varchar(128) default null comment '邮箱',
    bio text comment '个人简介'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

选择建议：

- 短文本优先使用 `varchar`。
- 明确固定长度的数据可以使用 `char`。
- 长正文、长备注、详情内容使用 `text`。
- 经常用于索引的字符串字段，不要设计得过长。

### 1.4 日期和时间类型

日期时间字段要先区分：只需要日期，还是需要日期加时间；是否涉及时区转换。

| 类型 | 保存内容 | 常见场景 |
| --- | --- | --- |
| `date` | 日期 | 生日、入学日期、账单日期 |
| `time` | 时间 | 时长、一天中的时间 |
| `datetime` | 日期和时间 | 创建时间、业务发生时间 |
| `timestamp` | 时间戳语义的日期和时间 | 需要受会话时区影响的时间 |

`date` 只保存日期，格式通常是 `YYYY-MM-DD`。

`datetime` 保存日期和时间，范围较大，常用于业务时间，例如订单创建时间、文章发布时间、任务完成时间。

`timestamp` 也保存日期和时间，但它会受到时区设置影响：写入和读取时，MySQL 会根据当前会话时区进行转换。它的范围也比 `datetime` 小，传统范围在 2038 年附近结束，因此不适合保存生日、长期计划时间、遥远未来时间等数据。

如果系统涉及跨国业务，常见做法是：

- 数据库存储统一使用 UTC 时间。
- 应用层根据用户所在时区展示本地时间。
- 对需要表示业务本地日期的字段，例如生日、账单日，使用 `date`，不要用时间戳硬转。

示例：

```sql
create table articles (
    id bigint primary key auto_increment comment '文章ID',
    title varchar(200) not null comment '标题',
    published_at datetime default null comment '发布时间',
    created_at datetime not null default current_timestamp comment '创建时间',
    updated_at datetime not null default current_timestamp on update current_timestamp comment '更新时间'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

### 1.5 一份完整建表示例

下面以学生表为例，把整数、字符串、小数、日期时间和约束组合起来：

```sql
create table student (
    id bigint primary key auto_increment comment '学生ID',
    student_no varchar(32) not null unique comment '学号',
    name varchar(64) not null comment '姓名',
    gender tinyint not null default 0 comment '性别：0未知，1男，2女',
    birthday date default null comment '生日',
    score decimal(5, 2) not null default 0.00 comment '综合成绩',
    phone char(11) default null comment '手机号',
    email varchar(128) default null comment '邮箱',
    status tinyint not null default 1 comment '状态：1正常，2停用',
    created_at datetime not null default current_timestamp comment '创建时间',
    updated_at datetime not null default current_timestamp on update current_timestamp comment '更新时间'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '学生表';
```

这张表体现了几个基本原则：

- 主键使用 `bigint auto_increment`，给长期增长预留空间。
- 学号使用 `unique` 保证业务唯一。
- 姓名、状态、创建时间等核心字段使用 `not null`。
- 成绩使用 `decimal`，避免浮点误差。
- 生日只需要日期，因此使用 `date`。
- 创建时间和更新时间使用默认值自动维护。

## 2. 约束

约束用于保证数据的正确性和一致性。它不是应用层校验的替代品，而是数据库层面的最后一道规则。

常见约束包括：

| 约束 | 作用 |
| --- | --- |
| `not null` | 限制字段不能为空 |
| `default` | 设置字段默认值 |
| `primary key` | 主键，唯一标识一行数据 |
| `auto_increment` | 自增长，常用于主键发号 |
| `unique` | 唯一约束，保证字段值不重复 |
| `foreign key` | 外键，维护表之间的引用关系 |
| `check` | 检查约束，限制字段取值范围 |
| `comment` | 字段或表注释，提升可维护性 |

### 2.1 空属性：null 和 not null

如果字段允许为空，可以写成 `null`，也可以不显式声明，因为 MySQL 字段默认允许 `null`。如果字段必须有值，应声明为 `not null`。

示例：

```sql
create table class (
    class_name varchar(20) not null comment '班级名称',
    class_room varchar(10) not null comment '教室'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

`null` 表示未知或不存在，它不是空字符串，也不是数字 0。`null` 参与普通计算时，结果通常仍然是 `null`：

```sql
select null;
select 1 + null;
```

因此，是否允许 `null` 要根据业务语义决定：

- 必填字段使用 `not null`。
- 可选字段可以允许 `null`。
- 如果字段需要参与计算，通常应避免随意允许 `null`，或者配合 `coalesce` 等函数处理。

### 2.2 默认值：default

`default` 用于给字段设置默认值。当插入数据时没有显式提供该字段，MySQL 会使用默认值。

示例：

```sql
create table test_default (
    name varchar(20) not null comment '姓名',
    age tinyint default 10 comment '年龄',
    gender char(1) default '男' comment '性别'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

插入时可以省略有默认值的字段：

```sql
insert into test_default (name) values ('aaa');

select * from test_default;
```

需要注意：

- `default` 只在字段被省略时自动生效。
- 如果显式插入 `null`，字段是否接受取决于是否声明了 `not null`。
- `default` 不等同于 `not null`。如果字段允许 `null`，即使有默认值，也仍然可以插入 `null`。

日常设计中，经常会把 `not null` 和 `default` 配合使用，例如状态字段、计数字段、是否删除字段：

```sql
status tinyint not null default 1 comment '状态'
```

### 2.3 列描述：comment

`comment` 用于给表或字段添加注释。注释不会改变数据存储行为，但能显著提升表结构的可读性。

示例：

```sql
create table test_comment (
    name varchar(60) not null comment '姓名',
    age tinyint default 10 comment '年龄',
    gender char(1) default '男' comment '性别'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci
  comment = '注释示例表';
```

查看完整建表语句时可以看到注释：

```sql
show create table test_comment;
```

建议为业务含义不明显的字段添加注释，尤其是状态码、枚举值、金额单位、时间语义等字段。

### 2.4 主键：primary key

主键用于唯一标识表中的一行数据。可以把它理解为一行数据的身份证。

主键具有以下特点：

- 主键值必须唯一。
- 主键字段不能为 `null`。
- 一张表只能有一个主键。
- 主键可以由一个字段组成，也可以由多个字段组成。

示例：

```sql
create table test_primary_key (
    id int primary key comment '学号',
    name varchar(64) not null comment '姓名'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

插入重复主键会失败：

```sql
insert into test_primary_key values (1, 'aaa');
insert into test_primary_key values (1, 'bbb');
```

如果表创建时没有设置主键，也可以后续追加：

```sql
alter table test_primary_key
    add primary key (id);
```

删除主键：

```sql
alter table test_primary_key
    drop primary key;
```

生产环境中不建议随意删除或修改主键。主键往往会被索引、外键、业务代码和数据同步任务依赖，变更前必须确认影响范围。

### 2.5 自增长：auto_increment

`auto_increment` 表示由 MySQL 自动为字段生成递增值，常用于主键 ID。

基本规则：

- 自增长字段必须是整数类型。
- 自增长字段必须是索引的一部分，最常见的是主键。
- 一张表最多只能有一个自增长字段。
- 如果手动插入了更大的值，后续自动生成值通常会在此基础上继续增长。

示例：

```sql
create table test_auto_increment (
    id bigint primary key auto_increment comment 'id',
    name varchar(64) not null comment '名称'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

插入数据：

```sql
insert into test_auto_increment (name) values ('aaaa');
insert into test_auto_increment (name) values ('bbbb');
insert into test_auto_increment (id, name) values (100, 'cccc');
insert into test_auto_increment (name) values ('dddd');

select * from test_auto_increment;
```

结果中的 ID 通常会表现为：

```text
+-----+------+
| id  | name |
+-----+------+
|   1 | aaaa |
|   2 | bbbb |
| 100 | cccc |
| 101 | dddd |
+-----+------+
```

`auto_increment` 只保证自动生成值递增，不保证严格连续。事务回滚、插入失败、删除数据等情况都可能导致 ID 中间出现空洞。

### 2.6 唯一约束：unique

`unique` 用于保证某个字段或字段组合的值不重复。主键只能有一个，但唯一约束可以有多个。

典型场景包括：

- 用户邮箱不能重复。
- 手机号不能重复。
- 学号不能重复。
- 同一个业务方的外部订单号不能重复。

示例：

```sql
create table test_unique (
    id bigint primary key auto_increment comment 'id',
    student_no varchar(32) unique comment '学号',
    name varchar(10) not null comment '姓名'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

重复插入相同学号会失败：

```sql
insert into test_unique (student_no, name) values ('01', 'aa');
insert into test_unique (student_no, name) values ('01', 'bb');
```

需要注意，MySQL 的唯一约束允许多个 `null` 值，因为 `null` 表示未知，多个未知值不被视为相等。如果业务要求字段必须唯一且不能为空，应同时声明 `not null unique`：

```sql
email varchar(128) not null unique comment '邮箱'
```

也可以建立联合唯一约束：

```sql
unique key uk_tenant_order (tenant_id, order_no)
```

这表示同一个租户下订单号不能重复，但不同租户可以使用相同订单号。

### 2.7 外键：foreign key

外键用于维护两张表之间的引用关系。外键定义在子表上，引用父表中的主键或唯一键。

例如，班级表是父表，学生表是子表。学生属于某个班级，因此学生表中的 `class_id` 可以引用班级表的 `id`。

```sql
create table myclass (
    id int primary key comment '班级ID',
    name varchar(30) not null comment '班级名'
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;

create table stu (
    id int primary key comment '学生ID',
    name varchar(30) not null comment '学生名',
    class_id int default null comment '班级ID',
    constraint fk_stu_class
        foreign key (class_id) references myclass(id)
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

插入父表数据：

```sql
insert into myclass values (10, '火箭班'), (20, '天体班');
```

插入合法学生数据：

```sql
insert into stu values (100, 'aaa', 10), (101, 'bbb', 20);
```

如果插入一个不存在的班级 ID，会失败：

```sql
insert into stu values (102, 'ccc', 30);
```

如果 `class_id` 允许为 `null`，也可以先不分配班级：

```sql
insert into stu values (102, 'ccc', null);
```

外键的基本要求：

- 父表被引用字段必须是主键或唯一键。
- 子表外键字段的数据类型应与父表字段保持一致。
- 使用 `innodb` 存储引擎。
- 插入子表数据时，外键值必须在父表中存在，或者外键字段为 `null`。

外键还能定义删除或更新父表数据时的行为，例如：

```sql
constraint fk_stu_class
    foreign key (class_id) references myclass(id)
    on delete set null
    on update cascade
```

常见动作包括：

- `restrict` 或 `no action`：限制删除或更新父表中仍被引用的数据。
- `cascade`：父表更新或删除时，子表同步更新或删除。
- `set null`：父表删除或更新后，子表外键字段设置为 `null`。

外键能增强数据库一致性，但也会增加写入和删除时的检查成本。业务系统是否使用外键，应结合团队规范、数据一致性要求和迁移维护成本决定。

### 2.8 检查约束：check

`check` 用于限制字段必须满足某个条件。在 MySQL 8.0.16 之后，`check` 约束会被实际检查。

示例：

```sql
create table test_check (
    id bigint primary key auto_increment comment 'id',
    age int not null comment '年龄',
    gender tinyint not null comment '性别：0未知，1男，2女',
    constraint chk_age check (age >= 0 and age <= 150),
    constraint chk_gender check (gender in (0, 1, 2))
) engine = innodb
  default character set utf8mb4
  default collate utf8mb4_0900_ai_ci;
```

`check` 适合表达简单、稳定、数据库层面可以判断的规则。复杂业务规则仍然应放在应用层处理。

## 3. 设计建议

设计 MySQL 字段和约束时，可以遵循以下原则：

- ID 类字段优先考虑 `bigint`，长期业务不要只看当前数据量。
- 金额使用 `decimal`，不要使用 `float` 或 `double` 保存需要精确计算的钱。
- 短文本使用 `varchar`，长正文使用 `text`，固定长度编码可以使用 `char`。
- 时间字段优先明确语义：业务时间常用 `datetime`，纯日期使用 `date`，跨时区展示由应用层处理。
- 必填字段使用 `not null`，状态和计数字段通常配合 `default`。
- 主键用于标识一行数据，唯一约束用于保证业务唯一。
- 外键能保证引用一致性，但使用前要考虑写入成本、迁移成本和团队规范。
- 字段注释要写清楚业务含义，尤其是状态码、枚举值、金额单位和时间语义。

