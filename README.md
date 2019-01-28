###  SQL 优化

1. SQL 语句执行效率差分析
    网速不给力，不稳定
    服务器内存不够，或SQL 被分配的内存不够
    SQL 语句设计不合理
    索引不合理
    没有有效索引视图
    表结构太大，没有有效分区
    数据库设计不规范，存在大量数据冗余
    统计信息过期等等

2. 找出导致性能慢的原因
    SQL 性能检测工具：sqlServer 
    分析出SQL 慢的相关语句，即执行时间常长，占系统资源，cpu过多的


2-1） 分析执行时间计划读取情况
    select * from dbo. Product

    查看执行时间和CPU占用时间
    set statistics time on
    select * from dbo.Product
    set statistics time off

    查看查询对I/0的操作情况
    set statistics io on
    select * from dbo.Product
    set statistics io off

    如果物理读取次数和预读次说比较多，可以使用索引进行优化
    查询>>高级设置选项：将上述两个选择打钩（time / io）

    查看执行计划
    数据库引擎优化对索引进行分析优化

3. 语句优化
3-1） select 查询
    保证不查询多余的列与行
        尽量避免使用 select * 使用具体列，避免多余列
        使用 where 限定具体要查询的数据，避免多余行
        使用 top / distinct 关键字，减少重复行

    
    -1）慎用 distinct 
    查询字段少时使用，字段过多，大大降低查询效率
    如：
    select distinct * from
    ((orde o left join orderproduct op on o.orderNum=op.orderNum)
    inner join product p on op.proNum=p.productNum)

    select * from
    ((orde o left join orderproduct op on o.orderNum=op.orderNum)
    inner join product p on op.proNum=p.productNum)

    使用 distinct 执行时间更长


    -2）慎用 union 关键字
    union 把多个查询语句结果集合到一个结果集中返回
    union语句必须满足：1.列数相同 2.对应列数的数据类型要保持兼容
    union all 代替union 不排序过滤的关键字


    -3）判断表中是否存在数据
    select count(*) from product
    或
    select top(1) id from product   效率更快

    -4）使用exist 代替 in 
    select * from A where idin (select id from B)
    改为
    select * from A where id exist(selct 1 from A.id=B.id)


    -5）where
    where子句使用 ！= 或 <> 操作符优化，索引将被放弃使用，会进行全表查询
    select id from A where id !=5;
    优化
    select id from A where id>5 or id<5;



    -6）连接查询优化
    inner join 结果集大小取决于左右两表满足条件的数量
    left join  左表大小，right join 相反
    完全和交叉连接 取决左右两表数据总量
 
    select * from
    ((select * from order where OrderID>1000) 
    o left join orderproduct op 
    on
    o.orderNum=op.orderNum)

    select * from 
    (orde o left join orderproduct op 
    on
    o.orderNum=op.orderNum)
    where
    o.OrderID>1000
    减少连接表提高查询效率


3-2）insert 语句
    使用临时表暂存中间结果
       避免程序中多次扫描主表, 减少了阻塞，提高了并发性能
       避免频繁创建和删除临时表，以减少系统资源的浪费
       尽量避免向客户端返回大数据量

    创建临时表1
    create table #tb1
    (
        id int,
        name nvarchar(20),
        createTime datetime
    )
    declare @i int
    declare @sql carchar(1000)
    set @i=0
    while (@i<100000>) --循环插入10w条数据
    begin
        set @i=@i+1
        set @sql=' insert into #tb1
    value('+convert(varchar(10),@i+)',''erzi'+convert(nvarchar(30),@i)+''','''+convert(nvarchar(30),getdate())+''')'
        exec(@sql)
    end

    执行时间50s

    --创建临时表2
    create table #tb2
    (
    id int,
    name nvarchar(30),
    createTime datetime
    )
    declare @i int
    declare @sql varchar(8000)
    declare @j int
    set @i=0
    while (@i<10000)  --循环插入10w条数据
    begin 
        set @j=0
    set @sql=' insert into #tb2 select '+convert(varchar(10),@i*100+@j)+',''erzi'+convert(nvarchar(30),@i*100+@j)+''','''+convert(varchar(50),getdate())+''''
        set @i=@i+1
    while(@j<10)
      begin   
        set @sql=@sql+' union all select '+convert(varchar(10),@i*100+@j)+',''erzi'+convert(nvarchar(30),@i*100+@j)+''','''+convert(varchar(50),getdate())+''''
        set @j=@j+1
      end 
      exec(@sql)
    end

    drop table #tb2
    select count(1) from #tb2


    注：insert into select批量插入，明显提升效率，尽量避免一个个循环插入




3-3）优化修改删除语句
    如果同时修改或删除过多数据，会造成cpu利用率过高从而影响别人对数据库的访问
    分批操作数据
    delete product where id<1000
    delete product where id>=1000 and id<2000
    delete product where id>=2000 and id<3000
    .....



4. limit 分页优化
    SELECT id FROM A LIMIT 1000,10      很快
    SELECT id FROM A LIMIT 10000,10     很慢
    优化
    select id from A order by id limit 10000,10;
    或
    select id from A order by between 10000 and 100010;



5. 批量插入优化
    INSERT into person(name,age) values('A',11)
    INSERT into person(name,age) values('B',12)
    INSERT into person(name,age) values('C',18)
    优化
    INSERT into person(name,age) value('A',11),('B',12),('C',18);






    总结：优化最重要的是在于平时设计语句，数据库的习惯，方式



