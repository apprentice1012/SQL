# 基数

&emsp;&emsp;某个列唯一键(Distinct_key)的数量叫做基数。比如性别列，只有男女之分，所以这一列基数就是2.主键列的基数等于的总行数。基数的高低影响列的数据分布。</br>

```SQL
  select count(distinct owner),count(distinct object_id),count(*)
  from test
```
```
  运行结果:
    29 72462 72462
```
&emsp;&emsp;运行结果表示owner的基数是29行，表的总行数是72462行，说明owner列中有大量重复值，object_id查询出来的结果与表的总行数相同，说明该列没有重复值相当于主键。<br>

```SQL
  select owner,count(*)
  from test
  group by owner
  order by 2 desc
```

  运行结果：
  
  ![image](https://github.com/apprentice1012/SQL/assets/126549223/e798de3b-d857-48c8-924f-75eef2a9c929)

&emsp;&emsp;在上表中拎出来两个数据来分析一下：<br>

```SQL
  select 30808/72462 "persent" from dual
```

```SQL
  select 7/72462 "persent" from dual
```
&emsp;&emsp;很显然第一个sql的百分比已经超过40%，而第二个只有0.009%。在sql里面超过5% 的查询应该全表嫂扫描，低于5 %的应该走索引。<br>

![image](https://github.com/apprentice1012/SQL/assets/126549223/2218b77d-488a-444b-9d00-9f620c40cee0)

ps：边读边记录的，后续再改。

&emsp;&emsp;如果某个列基数很低，会导致该列数据分布不均匀。因为不均匀所以在做条件查询的时候就可能走索引页可能走全表。如果怀疑列的数据分布不均匀可以用以下语句来查看数据列分布。</br>

```sql
  select row_name,count(*)
  from table_name
  group by row_name
  order by 2 desc
```

&emps;&emps;如果SQL是单表访问，可能走索引，可能走全表，也可能走物化视图扫描。在不考虑物化视图扫描的情况下，单表访问只可能走索引或者物化视图扫描。</br>

# 选择性

&emsp;&emsp;基数与总行数的比值再乘以100%就是某个列的选择性。在进行优化的时候只看基数一个指标是没有意义的，要结合总列数去看才会有实际意义。</br>

&emsp;&emsp;什么样的列需要建立索引？当选择性大于20%时，说明这个列的数据分布是比较均衡的。只有当一个列出现在where条件中且选择性大于20%且当前没有索引的时候当前列是需要建立索引的，从而提升查询性能。总行数只有几百行的话也没什么必要创建索引。</br>

&emsp;&emsp; **SQL优化的第一个观点：只有大表才会产生性能问题**</br>

&emsp;&emsp;有人说小表也会出现性能问题啊，比如说经常使用DML操作会产生热点快，也是性能问题。这属于是程序设计的问题，不能归属于优化。如果对一张很少数据的表需要经常使用DML这妥妥的设计问题。</br>

&emsp;&emsp;调优的第一个脚本：抓出必须创建索引的列。有两种方法，第一种是使用V$SQL_PLAN抓取，另一种则是通过下述脚本获取。</br>

```
SQL脚本
首先执行：
begin
  dbms_stats.flush_database_monitoring_info;
end;
含义：执行存储过程，刷新数据库监控信息。

之后：

select r.name owner,
  o.name table_name,
  c.name column_name,
  equality_preds,---等值过滤
  equijoin_preds,---等值join 比如 where a.id = b.id
  nonequijoin_preds,---不等join
  range_preds,--范围过滤次数 > >= < <= = between and
  like_preds,---like过滤
  null_preds,---null过滤
  timestamp
from sys.col_user u,sys.obj o,sys_col c,sys.user r
where o.obj = u.obj
  and c.obj = u.obj
  and c.col = u.intcol
  and r,name = 'SCOTT'
  and o.name = 'TEST' 
```
&emsp;&emsp;具体流程：首先先写一条sql把需要进行判断的列名写到where条件里，其次执行存储过程，刷新数据库监控信息，然后查看哪些列在where条件中，接下来筛选出来选择性大于20%的列，最后确保这些列没有创建索引。</br>

&emsp;&emsp;整合全部sql如下：

```SQL
第一个脚本:针对选择性进行SQL优化

select owner,
  column_name,
  num_rows,
  Cardinality,
  selectivity,
  'NEED INDEX' as notice
from(select b.owner,
            a.column_name,
            b.num_rows
            a.num_distinct Cardinality,
            round(a.num_distinct/b.num_rows*100, 2) selectivity
    from dba_tab_col_statistics a, dba_table b
    where a.owner = b.owner
      and a.table_name = b.table_name
      and a.owner = 'SCOTT'
      and a.table_name = 'TEST')
where selectivity >= 20
  and column_name not in (select column_name
                            from dba_ind_columns
                            where table_owner = 'SCOTT'
                              and table_name = 'TEST')
  and column_name in(select c,name
                      from sys.col_user u,sys.obj o,sys_col c,sys.user r
                      where o.obj = u.obj
                        and c.obj = u.obj
                        and c.col = u.intcol
                        and r,name = 'SCOTT'
                        and o.name = 'TEST');
```

# 直方图

&emsp;&emsp;之前说如果一个列的基数很低，那么这个列的数据将会分布不均匀。数据分布不均匀会导致在查询该列的时候，可能走索引，也可能走全表扫描很容易走错执行计划。</br>

&emsp;&emsp;如果没有对基数低的列收集直方图信息，那么基于成本优化器CBO会默认这些列是均衡的。</br>

```SQL
  for all columns size 1---对所有列都不收集直方图信息
```
&emsp;&emsp;之前的例子中owner列的基数很低，仍然用它来做示例，发现无论owner中的数据是什么在CBO中都展示的2499条，主打一个公平。这是因为没有收集直方图信息的时候CBO默认是均衡的。所以说执行计划中的CBO是假的，执行计划中的Rows是通过统计信息和一些数学公式计算得来的。**在做SQL优化的时候经常需要帮助执行计划CBO计算出比较准确的Rows**，之所以说较为精确是因为CBO是无法获得最精确的值，在对表进行数据统计的时候，统计信息一般不会按照100%来统计，即使按照100%来统计但是表中的数据是动态的，另外计算Rows的数学公式目前也是有缺陷的，故而无法获得精确值。如果Rows每次都能获得精确值，我们就只需要关注业务逻辑，表设计，SQL写法以及如何建立索引，不需要担心CBO执行错误计划了。</br>

&emsp;&emsp;Oracle12c的新功能SQL Plan Directives一定程度解决了Rows估算不准而引发的性能问题。</br>

&emsp;&emsp;为了让CBO能选择正确的执行计划，需要对需要查询的列收集直方图信息，从而告知CBO数据分布是不均衡的，让CBO在计算Rows时参考直方图来计算。</br>

```SQL
  for column owner size skewonly---对某一列生成直方图
```

&emsp;&emsp;在生成直方图之后在进行查询CBO的Rows估算基本准确，Rows估算准确那么执行计划就不会有问题了。收集直方图其实就是相当于运行了</br>
```SQL
  select owner,count(*) from test group by owner
```
&emsp;&emsp;这都是针对基数小数据分布不均匀，如果数据分布均匀则不需要直方图来辅助计算。</br>

&emsp;&emsp;什么样的列需要直方图呢？</br>

&emsp;&emsp;出现在where条件中，并且列的选择性小于1%且没有收集过直方图。千万不要对没有出现在where条件中的列收集直方图，那样纯粹是浪费数据库资源。</br>

```SQL
第二个脚本:针对直方图进行SQL优化

select a.owner,
      a.table_name,
      a.column_name,
      b.num_rows,
      b.num_distinct,
      trunc(num_distinct/num_rows*100,2) selectivity,
      'NEED GATHER HISTOGRAM' notice
from dba_tab_col_statistics a,dba_table b
where a.owner = 'SCOTT'
  and a.table_name = 'TEST'
  and a.owner = b.owner
  and a.table_name = b.table_name
  and num_distinct/num_rows < 0.01
  and (a.owner,a.table_name,a.column_name) in
      (select r.name owner,o.name table_name,c.name column_name
       from sys.col_user u,sys.obj o,sys_col c,sys.user r
                      where o.obj = u.obj
                        and c.obj = u.obj
                        and c.col = u.intcol
                        and r,name = 'SCOTT'
                        and o.name = 'TEST');
  and a.histogram = 'NONE';
```

# 回表

&emsp;&emsp;当对一个列创建索引之后，这个索引会包涵这个列的的键值以及键值对应行的RowId。当查询时通过索引中记录的RowId信息访问表的过程就叫做回表。回表一般是单块读，回表次数太多会严重影响SQL性能，回表过多久不应该就索引扫描而是应该走全表扫描。</br>

&emsp;&emsp;**在进行SQL优化的时候，应该注意回表次数，特别是回表的物理I/O次数。</br>

&emsp;&emsp;如果当前SQL有索引扫描且有回表现象，大部分消耗是在回表上面看的。可以通过执行计划查看TABLE ACCESS BY INDEX ROWID BATCHED来判断回表。既然回表会消耗性能，那么不让SQL语句产生回表不就好了，故而有一个问题什么样的语句会产生回表什么样的不会产生回表？</br>

```SQL
  select * from table where ...---会产生回表
  select count(*) from table ---不会产生回表
```
&emsp;&emsp;由于select *一定会产生回表所以在线上环境中严禁使用select *这种语句。当需要查询的列也在索引中那么就不会回表了。可以通过建立组合索引来消除回表，提升查询性能。当一个SQL中有多个过滤条件，但是只有其中一个列或者部分列创建了索引，这个时候会发生回表再过滤。这时候也需要创建组合索引来消除回表。</br>

# 集群因子

&emsp;&emsp;集群因子用于判断索引回表需要消耗的物理I/O次数。集群因子的算法主要是比较两个rowID是否在同一个数据块中，在同一个块中Clustering Factor+0，不再则+1，依次比较得出最后集群因子数据。集群因子大小整体来说介于表的数据块和表的总行数之间。集群因子只会影响索引范围扫描和索引全扫描，因为只有这两种索引扫描方式会有大量数据回表。不会影响索引唯一扫描和索引快速全扫描。因为索引唯一扫描只返回一条数据，而索引快速全扫描不回表。</br>

&emsp;&emsp;如果集群因子接近于表的行数，则说明表的数据和索引顺序差距很大，在进行索引范围扫描或者索引全扫描的时候，回表会读取更多的数据块。</br>

&emsp;&emsp;如果集群因子接近于总块数，则说明表的数据基本上是有序的，而且顺序和索引顺序基本类似。这样在进行索引范围扫描或者索引全扫描的时候，回表只需要读取少量数据块就能完成。</br>

```SQL
第三个脚本:计算集群因子
select sum(case
              when block1 = block2 and file1 = file2 then
                0
              else
                1
              end)CLUSTERING_FACTOR
from(select dbms_rowid.rowid_relative_fno(rowid) file1,
            lead(dbms_rowid.rowid_relative_fno(rowid),1,null) over(order by object_id) file2,
            dbms_rowid.rowid_block_fnumber(rowid) block1,
            lead(dbms_rowid.rowid_block_fnumber(rowid),1,null) over(order by object_id) block2
      from test
      where object_id is not null;)
```

&emsp;&emsp;集群因子影响的是索引回表的物理I/O次数。不能通过重建索引来降低集群因子因为表中和数据的顺序没有任何改变。唯一降低方法是重建表但是这不现实。之前说集群因子只会影响索引范围扫描和索引全扫描，如果这两种扫描不回表或者只有少量回表，不管集群因子多少，对性能都没有影响。如果实在无法消除回表将数据块缓存到buffer catch中这个时候也可以消除集群因子的影响，因为此时根本不需要物理I/O。</br>

# 表于表之间的关系

&emsp;&emsp;关系型数据库中表于表之间会产生关联，在进行关联之前首先要判断表于表之间的关系。表于表之间主要有三大关系1:1，1:N，N：N。两表之间关联如果是1:1的关系则关联后的结果也是1，数据不会存在重复。如果两表属于1:N的关系，关联之后返回的结果集是N的关系。如果两表之间是N：N的关系，那么结果集属于笛卡尔积，N：N的关系一般不存在于内外连接之中，只能存在半连接或者反连接中。如果不知道业务，不知道数据字典，判断表于表之间的关系可以对两表关联列进行汇总统计，就能看出来两表之间是什么关系。
