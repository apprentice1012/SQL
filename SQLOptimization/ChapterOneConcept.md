# 基数(CARDINALITY)

&emsp;&emsp;某个列唯一键(Distinct_key)的数量叫做基数。比如性别列，只有男女之分，所以这一列基数就是2.主键列的基数等于的总行数。基数的高低影响列的数据分布。</br>

```SQL
  select count(distinct owner),count(distinct object_id),count(*)
  from test
```
```
  运行结果:
    29 72462 72462
```
&emps;&emps;运行结果表示owner的基数是29行，表的总行数是72462行，说明owner列中有大量重复值，object_id查询出来的结果与表的总行数相同，说明该列没有重复值相当于主键。<br>

```SQL
  select owner,count(*)
  from test
  group by owner
  order by 2 desc
```

  运行结果：
  ![image](https://github.com/apprentice1012/SQL/assets/126549223/e798de3b-d857-48c8-924f-75eef2a9c929)

&emps;&emps;在上表中拎出来两个数据来分析一下：<br>

```SQL
  select 30808/72462 "persent" from dual
```

```SQL
  select 7/72462 "persent" from dual
```
&emps;&emps;很显然第一个sql的百分比已经超过40%，而第二个只有0.009%。在sql里面超过5% 的查询应该全表嫂扫描，低于5 %的应该走索引。<br>

![image](https://github.com/apprentice1012/SQL/assets/126549223/2218b77d-488a-444b-9d00-9f620c40cee0)

ps：边读边记录的，后续再改。

&emps;&emps;如果某个列基数很低，会导致该列数据分布不均匀。因为不均匀所以在做条件查询的时候就可能走索引页可能走全表。如果怀疑列的数据分布不均匀可以用以下语句来查看数据列分布。</br>

```sql
  select row_name,count(*)
  from table_name
  group by row_name
  order by 2 desc
```

&emps;&emps;如果SQL是单表访问，可能走索引，可能走全表，也可能走物化视图扫描。在不考虑物化视图扫描的情况下，单表访问只可能走索引或者物化视图扫描。</br>

# 选择性

&emps;&emps;基数与总行数的比值再乘以100%就是某个列的选择性。在进行优化的时候只看基数一个指标是没有意义的，要结合总列数去看才会有实际意义。</br>

&emps;&emps;什么样的列需要建立索引？当选择性大于20%时，说明这个列的数据分布是比较均衡的。只有当一个列出现在where条件中且选择性大于20%且当前没有索引的时候当前列是需要建立索引的，从而提升查询性能。总行数只有几百行的话也没什么必要创建索引。</br>

&emps;&emps; **SQL优化的第一个观点：只有大表才会产生性能问题**</br>

&emps;&emps;有人说小表也会出现性能问题啊，比如说经常使用DML操作会产生热点快，也是性能问题。这属于是程序设计的问题，不能归属于优化。如果对一张很少数据的表需要经常使用DML这妥妥的设计问题。</br>

&emps;&emps;调优的第一个脚本：抓出必须创建索引的列</br>

```
SQL脚本
首先执行：
begin
  dbms_stats.flush_database_monitoring_info;
end;
含义：执行存储过程，刷新数据库监控信息。

之后：

```
