# 什么是统计信息

&emsp;&emsp;之前说只有大表才会产生性能问题，优化器是怎么知道哪个表是大表？这就要通过统计信息来获取。之前提到的基数，选择性。集群因子等概念也是需要通过统计信息才能获取的。如果没有正确的手机统计信息，那么SQL执行计划就会跑偏，也就可能出现性能问题。收集统计信息就是为了让优化器选择最佳的执行计划，以最少的代价查询出来结果。</br>

&emsp;&emsp;统计信息包含：表的统计信息，列的统计信息，索引的统计信息，系统的统计信息，数据字典的统计信息以及动态性能视图基表的统计信息。暂时只讲解前三种</br>

&emsp;&emsp;表的统计信息包括：表的总行数(num_rows)，表的块数(blocks)以及行的平均长度(avg_row_len)。可以通过查询数据字典DBA_TABLES获取这些信息。</br>

```SQL
select owener,table_name,num_rows,blocks,avg_row_len
from dba_tables
where owner = 'owner' and table_name = 'test';
```

&emsp;&emsp;由于当前表是新建的，所以在dba_tables中是没有信息的。先收集一下信息，之后再查询就有数据了。</br>

```SQL
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(ownername => 'owner',
                                tabname => 'test',
                                estimate_percent => 100,
                                method_opt => 'for all column size auto',
                                no_invalidate => FASLE,
                                degree => 1,
                                cascade => TRUE);

END;
```

&emsp;&emsp;列的统计信息包括：列的基数，列的空值数量和列的数据分布信息(直方图), 可以通过数据字典DBA_TAB_COL_STATISTICS来查询。</br>

```SQL
select column_name,num_distinct,num_nulls,num_buckets,histogram --列名，基数，空值数量，直方图桶数，直方图类型
from dba_tab_col_statistics
where owner = 'owner' and table_name = 'test';
```

&emsp;&emsp;可以用以下脚本来收集表和列的统计信息</br>

```SQL
select a.column_name,b.num_rows,a.num_nulls,a.num_distinct Cardinality,
       round(a.num_distinct/b.num_rows*100,2) selectivity,
       a.histogram,a.num_buckets
from dba_tab_col_statistics a,dba_tables b
where a.owner = b.owner and a.table_name = b.table_name
      and a.owner = 'owener' and a.table_name = 'test';
```

&emsp;&emsp;索引的统计信息包括：索引blevel(索引高度-1)，叶子块的个数，以及集群因子。可以通过数据字典DBA_INDEXES来查询统计信息。在创建索引的时候会自动收集统计信息。</br>

```SQL
select blevel,leaf_blocks,clustering_factor,status
from dba_indexes
where owner = 'owner' and index_name = 'test'
```

&emsp;&emsp;如果需要单独对索引收集信息可以用以下脚本进行收集。</br>

```SQL
BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(ownername => 'owner',
                                indname => 'test');
END;
```

# 统计信息重要参数设置
