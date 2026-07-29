本文要解决四件事：

- InnoDB 的数据从表空间到记录是怎么组织的。
- MySQL 索引为什么能定位和范围扫描。
- 聚簇索引、二级索引、回表和覆盖索引是什么关系。
- 页大小、树高、主键长度和页分裂为什么影响性能。

这一篇只讲 MySQL，重点放在 InnoDB 的索引和页面结构。

B 树为什么从数组、二叉树一步步演变而来，以及 B 树、B+ 树如何查询、分裂和合并，单独见 [B树与B+树底层原理](./B树与B+树底层原理.md)。

先确定官方术语边界：MySQL 官方文档通常使用 `B-tree` 描述 MySQL 和 InnoDB 的普通索引，并明确 InnoDB 索引记录存储在叶子页中。

平时很多资料会说 InnoDB 使用 `B+tree`，主要是在强调“非叶子页负责导航、叶子页保存索引记录、范围扫描沿叶子层推进”这些特征。

学习时可以理解 B+ 树思想，但讲 MySQL 官方事实时，优先说：

**InnoDB 非空间索引是 B-tree 结构，索引记录存储在叶子页中。**

## 1. 先把问题放到磁盘页上

可以把一张 InnoDB 表想象成一本很厚的纸质书：

- 一行记录：书里的一条内容。
- 一个页：书里的一页纸。
- 一个索引：目录和正文一起组织出来的查找结构。
- 一次随机 I/O：从书架里抽出书，再翻到某一页。

如果没有索引，要找 `id = 800000`，就像从第一页开始翻：

```text
第 1 页 -> 第 2 页 -> 第 3 页 -> ... -> 很多页
```

如果有索引，就像先走目录：

```text
根页 -> 中间页 -> 叶子页 -> 找到记录
```

索引快，不是因为它不用读数据，而是因为它把“读很多页”变成了“读少数几层页”。

![索引优势](https://gitee.com/binary-whispers/pic/raw/master///20260728210834112.png)

## 2. InnoDB 的存储层级

理解 InnoDB，可以先从这个层级开始：

```text
表空间 -> 段 -> 区 -> 页 -> 记录
```

### 2.1 表空间

表空间是 InnoDB 保存数据的逻辑容器，内部由数据库页组成。

独立表空间模式下，一张 InnoDB 表通常会有自己的 `.ibd` 文件，但不能简单把 `.ibd` 理解成“只有表数据”。表的数据和索引都按照 InnoDB 的页面格式组织在表空间中。

### 2.2 段

段是按用途管理页面的逻辑结构。

一棵索引通常需要管理叶子页和非叶子页，这两类页面的增长方式不同，所以 InnoDB 会从段的角度管理它们需要的空间。

需要明确：
- 段负责按用途组织空间；
- 真正读写数据时，基本单位仍然是页。

### 2.3 区

区是一组连续的页，用于批量管理空间。

默认页大小为 16KB 时，一个 1MB 的区包含 64 个连续页：

$1MB / 16KB = 64$

区的意义是减少页面逐个分配带来的管理成本，并让一部分页面在物理位置上保持连续。

### 2.4 页

页是 InnoDB 管理和读写数据的基本单位。默认页大小是 16KB，由实例初始化时的 `innodb_page_size` 决定。

页不是越大越好：

- 页越大，一个索引页通常能放更多记录，树可能更矮。
- 页越大，一次读取和修改牵动的数据也更多。
- 缓存命中、写入放大和并发访问也会受到影响。

### 2.5 记录

记录是页中真正保存的索引记录或行数据。

同样是记录，在不同索引叶子页中保存的内容不同：

- 聚簇索引叶子记录：主键 + 整行数据
- 二级索引叶子记录：二级索引列 + 主键值

这个差异会继续推导出回表、覆盖索引和主键长度问题。

![InnoDB架构](https://gitee.com/binary-whispers/pic/raw/master///20260729101203464.png)

## 3. MySQL 索引为什么能定位

MySQL 官方文档在 `How MySQL Uses Indexes` 中说，索引用来快速找到具有特定列值的行；没有索引时，MySQL 必须从第一行开始读完整张表。

还要记住一个基础事实：

> Most MySQL indexes are stored in B-trees.

大多数普通索引不是简单数组或链表，而是有序、多路、按页组织的树形结构。

例如：

```sql
create table orders (
    id bigint primary key,
    user_id bigint not null,
    status varchar(20) not null,
    created_at datetime not null,
    amount decimal(10, 2) not null,
    detail varchar(1000),
    index idx_user_id (user_id),
    index idx_created_at (created_at)
) engine = innodb;
```

执行：

```sql
select * from orders where id = 800000;
```

可以先把查找过程理解成：

```text
根页：800000 应该进入哪个子页
中间页：继续缩小 800000 所在范围
叶子页：在本页找到 id = 800000 的记录
```

每下降一层都在缩小范围。

![索引查询](https://gitee.com/binary-whispers/pic/raw/master///20260728211533356.png)

### 3.1 有序、多路、页

理解 MySQL 普通索引，先抓住三个关键词：

```text
有序
多路
页
```

有序决定索引可以定位范围起点，并按顺序继续扫描。

多路表示一个父页可以通过很多目录项指向多个子页，而不是像二叉树一样只有两个分支。

按页组织表示树中的节点不是一个个零散小对象，而是与数据库的页面读写方式结合。

```text
一个页容纳更多目录项
-> 一个父页能管理更多子页
-> 树高可能更低
-> 从根到叶子访问的页更少
```

![降低树高](https://gitee.com/binary-whispers/pic/raw/master///20260728213327252.png)

B 树和 B+ 树为何具备这些特点，见 [B树与B+树底层原理](./B树与B+树底层原理.md)。

### 3.2 为什么适合范围扫描

假设有索引：

```sql
create index idx_created_at on orders(created_at);
```

查询：

```sql
select id, created_at, amount
from orders
where created_at >= '2026-07-01 00:00:00'
  and created_at < '2026-08-01 00:00:00'
order by created_at;
```

MySQL 可以先在 `idx_created_at` 中定位到范围起点，然后按索引顺序向后扫描，直到超过范围终点：

```text
第一步：定位范围起点
第二步：沿有序索引向后读
第三步：超过范围终点就停止
```

MySQL 官方文档说明，B-tree 索引可用于 `=`、`>`、`>=`、`<`、`<=`、`between` 等比较；hash 索引只适合等值比较，不适合查找范围。

```text
B-tree：能够利用顺序，适合等值、范围、排序和前缀匹配。
hash：直接计算位置，适合等值，但不知道下一个有序值在哪里。
```

这也是为什么下面第一条 SQL 有机会利用普通 B-tree 索引定位起点，而第二条通常不能：

```sql
select * from users where name like 'zhang%';
select * from users where name like '%zhang%';
```

![范围扫描](https://gitee.com/binary-whispers/pic/raw/master///20260728213358982.png)

## 4. InnoDB 索引页内部有什么

一个索引页可以粗略理解成：

![01-innodb-index-page-internal-structure](https://gitee.com/binary-whispers/pic/raw/master///20260728213503397.png)

不需要先背每个字节和字段，先理解下面四部分。

### 4.1 页头部信息

页头部保存页面管理信息，比如页属于哪个层级、页中有多少记录、空闲空间在哪里，以及页面之间的关系。

页不是一袋散装记录，它自己带有完整的管理信息。

### 4.2 infimum 和 supremum

可以把它们理解成页内的两个边界记录：

- `infimum`：比本页所有用户记录都小。
- `supremum`：比本页所有用户记录都大。

它们类似页内记录链的起点和终点，方便按顺序组织和扫描记录。

### 4.3 用户记录

用户记录是页面中真正保存的索引记录：

```text
聚簇索引叶子页：用户记录保存整行数据。
二级索引叶子页：用户记录保存二级索引列和主键值。
非叶子索引页：目录记录用于指向下一层页面。
```

### 4.4 页目录

一个 16KB 页面中可能有很多记录。如果页内也从第一条记录开始扫描，会浪费大量比较。

Page Directory 会把页内记录分成若干组。查找时先在有序槽中定位目标记录组，再在组内检查少量记录。

所以一次索引定位包含两层：

```text
树层级定位：根页 -> 中间页 -> 叶子页
页内定位：页目录二分 -> 组内记录链查找
```

![02-page-directory-in-page-lookup](https://gitee.com/binary-whispers/pic/raw/master///20260728213556870.png)

## 5. 聚簇索引与二级索引

官方文档说，InnoDB 索引记录存储在 B-tree 的叶子页中。

真正决定查询路径的是：不同索引的叶子页到底保存什么。

### 5.1 聚簇索引叶子页

每个 InnoDB 表都有一个特殊索引，叫 clustered index，它存储行数据。

如果定义了主键，InnoDB 使用主键作为聚簇索引：

```text
primary key -> 聚簇索引叶子页 -> 整行数据
```

执行：

```sql
select * from orders where id = 800000;
```

找到聚簇索引叶子页时，整行数据已经在那里，不需要再去其他位置取整行。

如果没有主键，InnoDB 会选择第一个所有列都为 `not null` 的唯一索引作为聚簇索引；如果仍然没有，会生成隐藏的行 ID 来组织聚簇索引。

### 5.2 二级索引叶子页

InnoDB 二级索引记录包含二级索引列和这一行的主键列：

```text
idx_user_id 叶子页：user_id + id
primary 叶子页：id + 整行数据
```

执行：

```sql
select * from orders where user_id = 10;
```

流程是：

```text
1. 走 idx_user_id 定位 user_id = 10。
2. 在二级索引叶子页拿到主键 id。
3. 用 id 再走 primary 聚簇索引。
4. 在聚簇索引叶子页取得整行数据。
```

第 3、4 步就是回表。

如果只查询二级索引已有的列：

```sql
select id, user_id
from orders
where user_id = 10;
```

`idx_user_id` 中已经有 `user_id` 和主键 `id`，不需要回到聚簇索引。这就是覆盖索引的核心。

![03-clustered-vs-secondary-leaf-page](https://gitee.com/binary-whispers/pic/raw/master///20260728213616743.png)

![覆盖和回表](https://gitee.com/binary-whispers/pic/raw/master///20260728221014783.png)

### 5.3 InnoDB 是索引组织表

InnoDB 表的数据本身存储在聚簇索引中：

```text
primary 聚簇索引：
id -> 整行数据

idx_user_id 二级索引：
user_id -> id
```

所以不能把 InnoDB 简单理解成：

```text
表文件保存数据
主键索引另外保存指向物理行的地址
```

更准确的模型是：

```text
聚簇索引本身就是表数据的组织方式。
二级索引通过主键值回到聚簇索引。
```

![05-innodb-index-organized-table](https://gitee.com/binary-whispers/pic/raw/master///20260728213710044.png)

## 6. 把一条查询完整串起来

现在用同一张 `orders` 表把索引页、叶子页、范围扫描和回表连起来。

### 6.1 主键等值查询

```sql
select * from orders where id = 800000;
```

```text
聚簇索引根页
-> 根据分隔键选择子页
-> 逐层下降到叶子页
-> 通过页目录定位 id = 800000
-> 直接取得整行
```

主键查询快，是因为聚簇索引把定位路径和行数据放在同一棵树中。

### 6.2 二级索引等值查询

```sql
select * from orders where user_id = 10;
```

```text
idx_user_id 根页
-> idx_user_id 叶子页
-> 取得主键 id
-> primary 根页
-> primary 叶子页
-> 取得整行
```

二级索引先定位候选主键；如果查询需要索引中没有的列，就要回表。

### 6.3 二级索引范围扫描

```sql
select id, created_at
from orders
where created_at >= '2026-07-01 00:00:00'
  and created_at < '2026-08-01 00:00:00';
```

```text
在 idx_created_at 中定位范围起点
-> 沿二级索引叶子层向后扫描
-> 超过范围终点时停止
-> 返回列被索引覆盖，不需要回表
```

如果改成：

```sql
select *
from orders
where created_at >= '2026-07-01 00:00:00'
  and created_at < '2026-08-01 00:00:00';
```

每个候选主键通常还要去聚簇索引取整行：

```text
扫描二级索引叶子页
-> 取得主键 id
-> 用 id 查询聚簇索引
-> 返回整行
```

命中 10 行时回表规模很小；命中 100 万行时，大量离散回表可能比顺序扫描更贵。

所以范围查询的粗略成本是：

```text
总成本 ≈ 定位范围起点
       + 扫描范围覆盖的叶子页
       + 可能发生的回表
```

## 7. 页结构如何影响索引性能

### 7.1 树高为什么重要

可以用一个不精确但有用的模型：

```text
一次索引定位成本 ≈ 从根页到叶子页访问的页数 + 页内查找成本
```

树高是 3：

```text
根页 -> 中间页 -> 叶子页
```

树高是 4：

```text
根页 -> 中间页 1 -> 中间页 2 -> 叶子页
```

多一层，就多一层页面访问和判断。如果页面不在 buffer pool 中，还可能触发磁盘读取。

树高主要受到这些因素影响：

- 表中记录数量。
- 索引键和索引记录大小。
- 页大小。
- 页面填充率和碎片情况。
- 主键长度。

![04-tree-height-3-vs-4-page-cost](https://gitee.com/binary-whispers/pic/raw/master///20260728213648723.png)

树高和扇出的完整推导见 [B树与B+树底层原理](./B树与B+树底层原理.md)。

### 7.2 主键长度为什么影响所有二级索引

每条 InnoDB 二级索引记录都包含主键列。

如果一张表有 5 个二级索引，主键从 8 字节整数变成长字符串，影响会传导到每一棵二级索引：

```text
二级索引叶子记录变大
-> 每页容纳记录减少
-> 二级索引总页数增加
-> buffer pool 能缓存的有效记录减少
-> 树高或页面访问可能增加
```

所以“主键尽量短”不只是为了主键查询，而是在控制所有二级索引的公共负担。

### 7.3 顺序主键和随机主键

单调递增主键通常写入最右侧叶子页，页面大体从左到右增长。

随机 UUID 等键可能落到许多已有页面中间，更容易触及不同页的剩余空间，也可能更频繁遇到需要分裂的页面。

InnoDB 插入聚簇索引记录时会为页面保留一定空间。顺序插入通常形成更稳定的页面填充，随机插入下的页面填充率和分裂位置更不稳定。

这里不要把结论说绝对：

```text
不是所有业务都必须使用自增主键。
也不是 UUID 一定不能作为主键。
```

真正要理解的是：

```text
主键决定聚簇索引的组织顺序，
同时还会被带入每一棵二级索引。
```

### 7.4 页分裂、删除和合并

目标叶子页有空间时，新记录写入正确位置并维护页内组织。

页面空间不足时可能发生分裂：

```text
叶子页分裂
-> 父页新增目录项
-> 父页也满时继续向上分裂
-> 根页分裂时树高增加
```

删除记录后，页面空间可以被后续操作复用。当索引页填充率低于 `MERGE_THRESHOLD` 时，InnoDB 会尝试收缩索引树并释放页面；未指定时默认阈值为 50%。

```text
记录删除
-> 页面空间可以复用
-> 满足条件时页面可能合并
-> 表空间文件是否缩小是另一个层面的问题
```

分裂和合并的通用树结构过程见 [B树与B+树底层原理](./B树与B+树底层原理.md)。

![15-five-leaf-page-performance-relationships-v2](https://gitee.com/binary-whispers/pic/raw/master///20260728220912550.png)

## 8. 和 MyISAM 存储引擎对比

MySQL 官方文档说明，MyISAM 表会创建数据文件和索引文件：

```text
tbl_name.myd：数据文件
tbl_name.myi：索引文件
```

所以 MyISAM 更适合这样理解：

```text
索引文件保存索引项，并指向数据文件中的记录位置。
数据文件保存真实行数据。
```

对比 InnoDB：

```text
InnoDB：
聚簇索引叶子页保存整行数据。
二级索引叶子记录保存主键值。

MyISAM：
数据文件和索引文件分离。
索引定位后根据位置读取数据文件。
```

![06-innodb-clustered-vs-myisam-separated-files](https://gitee.com/binary-whispers/pic/raw/master///20260728220601062.png)

## 9. 常见误区

### 9.1 “有索引就一定快”

不一定。

如果一个条件命中全表大部分数据，走二级索引可能意味着：

```text
扫描大量二级索引记录 + 大量回表
```

这可能比顺序扫描更贵。

### 9.2 “B+tree 永远只有三层”

不能这么说。

树高取决于数据量、页大小、索引记录大小和页面填充情况。只能说多路索引树的分支通常很大，所以树高通常比较低。

### 9.3 “InnoDB 表和索引完全分开”

这句话不适合 InnoDB。

InnoDB 表数据本身存储在聚簇索引中。二级索引是另外的索引树，并通过主键值回到聚簇索引。

## 10. 文章总结

整篇可以压缩成下面这条链路：

```text
InnoDB 表空间由段、区和页组织，页是数据读写和索引组织的基本单位。

普通索引按照 B-tree 结构组织。查询从根页开始，根据分隔键逐层选择子页，最后到叶子页，再通过页目录定位具体记录。

聚簇索引叶子页保存整行数据，所以主键查询到达叶子页就能取得整行。二级索引叶子页保存二级索引列和主键；查询需要其他列时，要使用主键回到聚簇索引，这就是回表。查询需要的列都在二级索引中时，就是覆盖索引。

范围查询先定位起点，再沿有序叶子层扫描到终点。范围越大、回表越多，成本越高，所以有索引不代表一定快。

页大小、索引记录长度和页面填充率共同决定一个页能放多少记录。主键越长，所有二级索引记录都会变大；随机插入更容易落到已有页面中间，页面分裂和空间利用也会受到影响。
```

最后记住：

```text
定位性能：看树高和每层页访问。
扫描性能：看范围覆盖多少叶子页。
回表成本：看二级索引能否覆盖查询列。
空间成本：看索引记录大小和页面填充率。
写入成本：看目标叶子页的位置、剩余空间和分裂概率。
```

## 11. 参考资料

- MySQL 8.4 Reference Manual: How MySQL Uses Indexes  
  https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html
- MySQL 8.4 Reference Manual: Comparison of B-Tree and Hash Indexes  
  https://dev.mysql.com/doc/refman/8.4/en/index-btree-hash.html
- MySQL 8.4 Reference Manual: Clustered and Secondary Indexes  
  https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html
- MySQL 8.4 Reference Manual: The Physical Structure of an InnoDB Index  
  https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html
- MySQL 8.4 Reference Manual: File Space Management  
  https://dev.mysql.com/doc/refman/8.4/en/innodb-file-space.html
- MySQL 8.4 Reference Manual: Pages, Extents, Segments, and Tablespaces  
  https://dev.mysql.com/doc/refman/8.4/en/innodb-file-space.html
- MySQL 8.4 Reference Manual: Files Created by CREATE TABLE  
  https://dev.mysql.com/doc/refman/8.4/en/create-table-files.html
- MySQL 8.4 Reference Manual: The MEMORY Storage Engine  
  https://dev.mysql.com/doc/refman/8.4/en/memory-storage-engine.html
- MySQL 8.4 Reference Manual: CREATE INDEX Statement  
  https://dev.mysql.com/doc/refman/8.4/en/create-index.html
