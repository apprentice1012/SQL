![image](https://github.com/apprentice1012/SQL/assets/126549223/9c11471f-3ce2-402e-984f-64e99af8dea3)# 基数(CARDINALITY)

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
&emsp;&emsp;之前的例子中owner列的基数很低，仍然用它来做示例，发现无论owner中的数据是什么在CBO中都展示的2499条，主打一个公平。这是因为没有收集直方图信息的时候CBO默认是均衡的。所以说执行计划中的CBO是假的，执行计划中的Rows是通过统计信息和一些数学公式计算得来的。**在做SQL优化的时候经常需要帮助执行计划CBO计算出比较准确的Rows，之所以说较为精确是因为CBO是无法获得最精确的值，在对表进行数据统计的时候，统计信息一般不会按照100%来统计，即使按照100%来统计但是表中的数据是动态的，另外计算Rows的数学公式目前也是有缺陷的，故而无法获得精确值。如果Rows每次都能获得精确值，我们就只需要关注业务逻辑，表设计，SQL写法以及如何建立索引，不需要担心CBO执行错误计划了。</br>

&emsp;&emsp;Oracle12c的新功能SQL Plan Directives一定程度解决了Rows估算不准而引发的性能问题。</br>

&emsp;&emsp;为了让CBO能选择正确的执行计划，需要对需要查询的列收集直方图信息，从而告知CBO数据分布是不均衡的，让CBO在计算Rows时参考直方图来计算。
