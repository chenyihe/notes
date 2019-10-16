# Hive-hive中Left semi join的用法

在日常使用hive中经常会用到类似以下写法：

```sql
--过滤掉t表中在存在在a表，b表或者s表中的user_id
select t.* from t
where not exists(select 1 from s where t.user_id = s.user_id)
  and not exists(select 1 from a where t.user_id = a.user_id)
  and not exists(select 1 from b where t.user_id = b.user_id)
```

由于在hive低版本中不支持in/exists语法，需要用**left semi join**代替

```sql
select t.* from t 
left semi join a on t.user_id = a.user_id 
left semi join b on t.user_id = b.user_id
left semi join s on t.user_id = s.user_id 
```

join 中右表只能在on子句中设置过滤条件，不能在where子句或SELECT中过滤。

### left join 和left semi join的区别

##### 1.联系：

​	都是hive的一种join方式，left join是common join (shuffle join/reduce join)的一种，left semi join是map join 

##### 2.区别：

（1）Semi Join，也叫半连接，是从分布式数据库中借鉴过来的方法。它的产生动机是：对于reduce side join，跨机器的数据传输量非常大，这成了join操作的一个瓶颈，如果能够在map端过滤掉不会参加join操作的数据，则可以大大节省网络IO，提升执行效率。
实现方法很简单：选取一个小表，假设是File1，将其参与join的key抽取出来，保存到文件File3中，File3文件一般很小，可以放到内存中。在map阶段，使用DistributedCache将File3复制到各个TaskTracker上，然后将File2中不在File3中的key对应的记录过滤掉，剩下的reduce阶段的工作与reduce side join相同。
由于 hive 中没有 in/exist 这样的子句（新版将支持），所以需要将这种类型的子句转成 left semi join。left semi join 是只传递表的 join key 给 map 阶段 , 如果 key 足够小还是执行 map join, 如果不是则还是 common join。关于 common join（shuffle join/reduce join）的原理请参考文末 refer。

（2）left semi join 子句中右边的表只能在 ON 子句中设置过滤条件，在 WHERE 子句、SELECT 子句或其他地方过滤都不行。

（3）对待右表中重复key的处理方式差异：因为 left semi join 是 in(keySet) 的关系，遇到右表重复记录，左表会跳过，而 join on 则会一直遍历。

最后的结果是这会造成性能，以及 join 结果上的差异。

（4）left semi join 中最后 select 的结果只许出现左表，因为右表只有 join key 参与关联计算了，而 join on 默认是整个关系模型都参与计算了。