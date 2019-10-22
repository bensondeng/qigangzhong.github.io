# mysql-exists-in

## 1、in和exists

in是把外表和内表作hash连接，而exists是对外表作loop循环，每次loop循环再对内表进行查询，一直以来认为exists比in效率高的说法是不准确的。**如果查询的两个表大小相当，那么用in和exists差别不大；如果两个表中一个较小一个较大，则子查询表大的用exists，子查询表小的用in。**

例如：表A(小表)，表B(大表)

```sql
select * from A where cc in(select cc from B)　　-->效率低，用到了A表上cc列的索引；

select * from A where exists(select cc from B where cc=A.cc)　　-->效率高，用到了B表上cc列的索引。
```

相反的：

```sql
select * from B where cc in(select cc from A)　　-->效率高，用到了B表上cc列的索引

select * from B where exists(select cc from A where cc=B.cc)　　-->效率低，用到了A表上cc列的索引。
```

## 2、not in 和not exists

`not in` 逻辑上不完全等同于`not exists`，如果你误用了`not in`，小心你的程序存在致命的BUG，请看下面的例子：

```sql
create table #t1(c1 int,c2 int);

create table #t2(c1 int,c2 int);

insert into #t1 values(1,2);

insert into #t1 values(1,3);

insert into #t2 values(1,2);

insert into #t2 values(1,null);


select * from #t1 where c2 not in(select c2 from #t2);　　-->执行结果：无，对于not exists查询，内表存在空值对查询结果没有影响；对于not in查询，内表存在空值将导致最终的查询结果为空。

select * from #t1 where not exists(select 1 from #t2 where #t2.c2=#t1.c2)　　-->执行结果：1　　3，对于not exists查询，外表存在空值，存在空值的那条记录最终会输出；对于not in查询，外表存在空值，存在空值的那条记录最终将被过滤，其他数据不受影响。
```

正如所看到的，not in出现了不期望的结果集，存在逻辑错误。如果看一下上述两个select 语句的执行计划，也会不同，后者使用了hash_aj，所以，请尽量不要使用not in(它会调用子查询)，而尽量使用not exists（它会调用关联子查询）。如果子查询中返回的任意一条记录含有空值，则查询将不返回任何记录。如果子查询字段有非空限制，这时可以使用not in，并且可以通过提示让它用hasg_aj或merge_aj连接。

如果查询语句使用了not in，那么对内外表都进行全表扫描，没有用到索引；而not exists的子查询依然能用到表上的索引。**所以无论哪个表大，用not exists都比not in 要快。**

## 3、in 与 = 的区别

```sql
select name from student where name in('zhang','wang','zhao');
```

与

```sql
select name from student where name='zhang' or name='wang' or name='zhao'
```

结果是相同的
