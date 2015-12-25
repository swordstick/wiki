## 前言


* 本文关注点为普通的业务实用,不涉及解析器的具体实现算法
* 本文只讨论SQL优化，不涉及DB实例优化，线程优化
* 本文所讨论SQL优化，理论部分在关系DB中通用
* 文章正文只描述关键性理论和思路，一些基础性内容，例如索引的适用场景，请查看附录以及链接


## 正文内容简介


* SQL优化关键性理论知识
* MYSQL中执行计划阅读
* SQL优化常用流程
* 附录：包含部分基础知识，附录持续增加更新


## SQL优化关键性理论知识


### 何为逻辑查询和物理查询

* 逻辑查询是通过高级表义性语言SQL来表达的对数据集合的处理需求 , ```也就是常说的SQL语句```
* 物理查询是SQL语言经过解析后，形成数据库系统的执行引擎能够理解和处理的执行树,```也就是执行计划```,而后在物理存储上进行 ```读取``` ```比较``` ```关联```各类操作的过程。
> 执行计划是物理查询的依据

### SQL语句的解析过程和方式

1. **词法分析:** <br>
SQL语句经过解析器,将具体的sql语句的各部分 SELECT,FROM,WHERE 进行词法分析,将sql语句分解为命令字,选项,参数等一系列基本的元素,然后交给查询优化器

2. **启发式优化:**<br> 
对sql语句进行改写,得到mysql认为更优的查询语句

3. **生产执行计划:** <br>
生产若干执行计划

4. **成本运算器选择最优:** <br>
解析器通过数据字典中，所有相关表，字段分布特性，分别计算执行计划的成本，选择出其中最优者 通常EXPLAIN所看到的，就是这个过程的结果

### 物理查询的关键点

* **物理查询是基于解析树生产的**<br>
树的叶子节点都是具体的表或者索引的读取操作,非叶子节点就是联接操作

* **物理查询是两两相连,过程化的**<br>
打个比方有A,B,C三个表,他们都互相关联,那么执行顺序是 A,B,C 三表中的某2个表首先联接,得到结果后,再关联第三个表。加入有100个表,顺序依然是,每次两个表关联,循环不止,直至结束

* **物理查询中有两个关键内容,第一是读取,第二是联接，后文较为详细说明**<br>
读取是指针对单个表通过```表扫描```或者```索引扫描```，```索引选择```的方式读取到该表符合SQL条件的所需数据<br>
联接是指每次将两个不同的数据来源的数据集通过where中描述的关联条件,过滤出符合条件的记录

### 读取操作@物理查询

* 表扫描
* 索引扫描
* 索引选择

### 联接操作@物理查询

####  Simple NEST LOOP JOIN 算法

* 解释<br>
每次从第一张表读取符合条件的一条记录 ,然后将记录与第二张表中的记录进行比较 。第一 张表成为外部表,第二张表称为内部表或者嵌套表

* 举例

```
案例SQL: select * from a,b where a.id=b.id

算法伪代码:
For each row a in A do 
        For each row b in B do
                If found b=r 
                        Then output <a,b>
```

#### Block nested-loops join 算法

* **解释**<br>
针对内部表没有索引时,之前的算法会导致大量的调用。
假如外部表的符合条件有1000行,内部表共有500行,而且内部表没有索引,那么将导致100次对内部表的扫描,共扫描行数 1000次全表扫描。<br>
BNL 算法中,不再根据每条符合条件的外部表行,调用一次内部表全扫描。而是先将外部表符合条件的行分组,比如 N 行记录作为一组放入到 JOIN BUFFER 内存(线程内存)中, 然后扫描内部表,
将每一行内部表记录和这一组 join buffer 内的外部表记录进行比较。 达到减少外部表对内部表的调用次数

* 举例

```
案例SQL: select * from a,b where a.id=b.id

算法伪代码:
For n_rows a in A do
        Stored used a columns from A 
        For each row b in B do
                For each a in join buffer
                        If row satisfy join condition
                                Then output <a,b>
```


### 数据集是个什么鬼

* 这是一个非常核心的概念
* SQL的目的就是查询数据，返回 或者 付诸修改，查询的最终结果是一个数据集，而查询过程中，每次读取，或者关联同样产生一个数据集，物理查询，也就是执行计划的整个执行过程，就
是***一个数据集不断的进行的```读取```,```筛选```,```组合```的过程***，在大脑中形成一个数据流，是非常关键的意识。
* 由数据集的概念，很容易散发出来思考，***整个优化的过程，就是让被处理的数据集尽可能少的过程***。


## MYSQL中执行计划阅读



> SQL的性能主要取决于物理查询<br>
> 物理查询由执行计划决定，所以执行计划的阅读和分析就是重中之中

### 获取```预执行计划```
> MYSQL 5.7 以前版本，只能explain select<br>
> ```预执行计划```并不是真实的执行计划，数据估算是依据数据字典进行，可能存在偏差<br>

*** mysql> ```explain``` select * from k_msong a,k_song b where a.songname=b.songname; ***

| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra | 
| :---- | :---- | :---- | :------ | :---- | :---- | :----- | :------ | :---- | :---- |
| 1 | SIMPLE | a | ALL | NULL   | NULL  |       NULL | NULL | 540208 |  |
| 1 | SIMPLE | b | ref | songname_Index | songname_Index | 303 | a.songname | 1 | Using where |

### 如何读懂执行计划

#### 掌握每行代表的含义

* 掌握相关字段的含义

```
◆ ID:Query Optimizer 所选定的执行计划中查询的序列号

◆ Select_type:所使用的查询类型,主要有以下这几种查询类型
◇ DEPENDENT SUBQUERY:子查询中内层的第一个 SELECT,依赖于外部查询的结果集; ◇ DEPENDENT UNION:子查询中的 UNION,且为 UNION 中从第二个 SELECT 开始的 后面所有 SELECT,同样依赖
于外部查询的结果集;
◇ PRIMARY:子查询中的最外层查询,注意并不是主键查询;
◇ SIMPLE:除子查询或者 UNION 之外的其他查询;
◇ SUBQUERY:子查询内层查询的第一个 SELECT,结果不依赖于外部查询结果集;
◇ UNCACHEABLE SUBQUERY:结果集无法缓存的子查询;
◇ UNION:UNION 语句中第二个 SELECT 开始的后面所有 SELECT,第一个 SELECT 为 PRIMARY
◇ UNION RESULT:UNION 中的合并结果;

◆ Table:显示这一步所访问的数据库中的表的名称;

◆ Type:告诉我们对表所使用的访问方式,主要包含如下集中类型;
◇ all:全表扫描
◇ const:读常量,且最多只会有一条记录匹配,由于是常量,所以实际上只需要读一次 ; ◇ eq_ref:最多只会有一条匹配结果,一般是通过主键或者唯一键索引来访问;
◇ fulltext:
◇ index:全索引扫描;
◇ index_merge:查询中同时使用两个(或更多)索引,然后对索引结果进行 merge 之后再 读取表数据;
◇ index_subquery:子查询中的返回结果字段组合是一个索引(或索引组合 ),但不是一个 主键或者唯一索引;
◇ rang:索引范围扫描;
◇ ref:Join 语句中被驱动表索引引用查询;
◇ ref_or_null:与 ref 的唯一区别就是在使用索引引用查询之外再增加一个空值的查询;
◇ system:系统表,表中只有一行数据;
◇ unique_subquery:子查询中的返回结果字段组合是主键或者唯一约束; ◇

◆ Possible_keys:该查询可以利用的索引. 如果没有任何索引可以使用,就会显示成 null, 这一
项内容对于优化时候索引的调整非常重要;

◆ Key:MySQL Query Optimizer 从 possible_keys 中所选择使用的索引;

◆ Key_len:被选中使用索引的索引键长度;

◆ Ref:列出是通过常量(const),还是某个表的某个字段(如果是 join)来过滤(通过 key) 的;

◆ Rows:MySQL Query Optimizer 通过系统收集到的统计信息估算出来的结果集记录条 数;

◆ Extra:查询中每一步实现的额外细节信息,主要可能会是以下内容:
◇ Distinct:查找 distinct 值,所以当 mysql 找到了第一条匹配的结果后 ,将停止该值的查 询而转为后面其他值的查询;
◇ Full scan on NULL key:子查询中的一种优化方式,主要在遇到无法通过索引访问 null 值的使用使用;
◇ Impossible WHERE noticed after reading const tables:MySQL Query Optimizer 通过 收集到的统计信息判断出不可能存在结果;
◇ No tables:Query 语句中使用 FROM DUAL 或者不包含任何 FROM 子句;
◇ Not exists:在某些左连接中 MySQL Query Optimizer 所通过改变原有 Query 的组成而 使用的优化方法,可以部分减少数据访问次数;
◇ Range checked for each record (index map: N):通过 MySQL 官方手册的描述,当MySQL Query Optimizer 没有发现好的可以使用的索引的时候,如果发现如果来自前面的 表的列值已知,可
能部分索引可以使用。对前面的表的每个行组合,MySQL 检查是否可以 使用 range 或 index_merge 访问方法来索取行。
◇ Select tables optimized away:当我们使用某些聚合函数来访问存在索引的某个字段的 时候,MySQL Query Optimizer 会通过索引而直接一次定位到所需的数据行完成整个查 询。当然,前
提是在 Query 中不能有 GROUP BY 操作。如使用 MIN()或者 MAX()的时 候;
◇ Using filesort:当我们的 Query 中包含 ORDER BY 操作,而且无法利用索引完成排序操 作的时候,MySQL Query Optimizer 不得不选择相应的排序算法来实现。
◇ Using index:所需要的数据只需要在 Index 即可全部获得而不需要再到表中取数据;
◇ Using index for group-by:数据访问和 Using index 一样,所需数据只需要读取索引即 可,而当 Query 中使用了 GROUP BY 或者 DISTINCT 子句的时候,如果分组字段也在索 引中,Extra 
中的信息就会是 Using index for group-by;
◇ Using temporary:当 MySQL 在某些操作中必须使用临时表的时候,在 Extra 信息中就 会出现 Using temporary 。主要常见于 GROUP BY 和 ORDER BY 等操作中。
◇ Usingwhere:如果我们不是读取表的所有数据,或者不是仅仅通过索引就可以获取所有 需要的数据,则会出现 Using where 信息;
◇ Using where with pushed condition:这是一个仅仅在 NDBCluster 存储引擎中才会出现 的信息,而且还需要通过打开 Condition Pushdown 优化功能才可能会被使用。控制参数 为engine_
condition_pushdown
```

#### 掌握如何判断执行顺序

* 执行顺序参考一: ```id``` 
> 当select_type不为DERIVED时，执行顺序由 id字段 从大到小执行
* 执行顺序参考二: ```DERIVED```
> 当select_type为DERIVED时，先行拼装 <derived+id> <br>***比如id为5的行，select_type为DERVIED时，拼装 dervied5 ***<br>
> 寻找在某个行中的table字段，若内容为 ***dervied5*** ，则说明该行为下一条语句,基本上以倒序显示

#### 试试看

```
案例SQL：

Explain select f.name as type,e.* from
( select d.name as pf,c.*
from
( select a.dtid,b.*
from
(select dtid,qid from t_test where addtime>='2012-03-30 00:17:38' and addtime<='2012-06-12 14:27:44' and pfid=12) as a
left join (select id,pfid,title from t_question) as b
on a.qid = b.id ) as c
left join (select id,name from t_platform) as d
on c.pfid=d.id ) as e
left join (select id,name from t_demand_type) as f on e.dtid=f.id ;

```

***对应的执行计划：***

| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 1 | PRIMARY | ```<derived2>``` | ALL | NULL | NULL | NULL | NULL |4749 | | 
| 1 | PRIMARY | ```<derived7>``` | ALL | NULL | NULL | NULL | NULL | 343 | | 
| 7 | DERIVED  | t_demand_type | ALL | NULL | NULL | NULL | NULL | 343 | | 
| 2 | DERIVED | ```<derived3>``` | ALL | NULL | NULL | NULL | NULL | 4749 | | 
| 2 | DERIVED | ```<derived6>``` | ALL | NULL | NULL | NULL | NULL | 5 | | 
| 6 | DERIVED | t_platform | ALL | NULL | NULL | NULL | NULL | 5 | | 
| 3 | DERIVED | ```<derived4>``` | ALL | NULL | NULL | NULL | NULL  | 4749 | | 
| 3 | DERIVED | ```<derived5>``` | ALL | NULL | NULL | NULL | NULL | 142938 | | 
| 5 | DERIVED |  t_question | ALL | NULL | NULL | NULL | NULL | 142938 | | 
| 4 | DERIVED| t_demand | ALL | NULL | NULL | NULL | NULL | 7529 | Using where | 


## SQL优化常用流程


### 1. 简化SQL写法

* 简化SQL写法本身,避免简单事情复杂化。

* 相应的判断标准

| id | 标准 | 
| :---- | :---- |
| 1 | 尽可能减少数据流程中需要```处理的数据量```,需要```关联的表数量```,废弃```不必存在的判断逻辑```,减少```不必要的列值的读取``` (即便不是最终反馈的列 ) | 
| 2 | 撤销不必要的 DISTINCT , 不必要的 UNION |


### 2. 脑补执行计划

* 把自己当成解析器

* 分析每个表的情况,包括索引分布,表的行数,相关列的选择性,然后根据这些情况,再优化写法,***即刻在脑海里就应该形成一个最佳的执行路径***。如果无法达到最优路径,说明写法还需要
修改,一直到认为可以达到最佳执行。

* 相应的判断标准

| id | 标准 |
| :---- | :---- |
| 1 | 是否做到小表驱动大表,确切的说是小数据集驱动大数据集 |
| 2| 是否做到高选择性条件是否给了驱动表,即便可能有别的写法 |
| 3| 独立子查询 IN 写法是否可能出现异常 ,导致关联子查询,<br>写法是否已规避 ,关联子查询 是否可以改写为非关联子查询 |
|4 | 确定没有出现将列进行函数封装 |
| 5 | 避免由系统来完成类型转换(准备关联的字段,类型是否一致) |
| 6 | 读表的方法明确,采用表扫描还是索引读或者索引扫描 |


### 3. EXPLAIN预执行计划分析

* 观察SQL EXPLAIN执行计划 判断是否真实达到了前一个步骤所希望看到的最优执行计划

* 运用储备知识，观察EXPLAIN 和 PROFILING 消耗信息，重点关注以下标准

* 相应的判断标准


| id | 标准 | 
| :---- | :---- |
| 1 | 是否达到了目标关联顺序,即从整体来看最少数据集的目标,小数据集驱动大数据集 | 
| 2 | 是否采取了合理的数据读取方法,表或索引 |
| 3 | 是否导致了异常的关联次数 |


### 重点：时刻关注数据集的走向


<br><br><br><br><br><br><br><br>
## 附录


### 索引何时不可用

* 列字段被封装函数
* 被类型转换
* <>
* like '%ddd'
* IS NULL 
* NOT IN || NOT EXIST 
* 比全表扫描还划不来
* 组合索引的前导列不包含在判断条件中

### 容易出现问题的典型写法

* 子查询，MYSQL的子查询处理不佳，尽可能不要采用子查询
> FEDERATED引擎除外
* OR,两个不同的判断字段条件，放置在OR两侧，在5.6版本以前很容易异常，导致表全扫，5.6版本以前主动写为UNION方式
* 字段的默认值使用NULL，导致查询条件IS NULL，可能用于判断条件的默认字段皆定义为非NULL默认值
* 字段的类型和给出的标量条件不一致，比如 username = 1111 