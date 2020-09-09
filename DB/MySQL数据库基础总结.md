
# MySQL数据库基础总结 #


## mysql的数据操纵语言DML ##


### 查询语句select ###

- **sql查询的书写顺序**

```
select [ column_name] from [ table_name] where [ column_name] group by [ column_name] having [ column_name] order by [ column_name] 
``` 

- **sql查询的执行顺序**  
  from...需要从哪个数据表检索数据  
  where...过滤表中数据的条件  
  group by...如何将上面过滤出的数据分组  
  having...上面已经分组的数据进行过滤的条件   
  select...查看结果集中的哪个列，或列的计算结果  
  order by...按照什么样的顺序来查看返回的数据 

 
### 修改语句update ###

- **基本语法**  
```
update [ table_name] set [ column_name] = value
```

- **使用示例**   
```
  update Tmp_Element as a left join (
select Dateyear,sum(t1.PM2_5) as SUMYear from air_quality_city_day_data t1
group by Dateyear) as b on a.Dateyear = b.Dateyear
set a.SUM_PM25_Year = b.SUMYear；
```


### 删除语句delete ###

```有条件的删除数据
delete from  [ table_name] where [ column_name];         
```
```高效的清空表数据
truncate table [ table_name];          
```
```删除表结构和表数据
drop table [ table_name];       
```


### 插入语句insert ###

```
insert into [ table_name]（ [ column_name]）values （[value]）
```

### mysql中的变量 ###

1. 	局部变量： 局部变量主要用在函数以及存储过程中	```declare c int ；set c = 8;``` 
2. 	会话变量： 会话变量仅对当前客户端连接有效	```set @var := ‘abc’;``` 
3. 	全局变量： 什么时候都有效。```set @@var := ‘abc’;``` 


### mysql中的临时表TEMPORARY ###

创建临时表：Create TEMPORARY TABLE Tmp_Element（...）；  

       *说明： 1，括号中的内容可以是和正式表一样的创建语句，也可以是一个查询结果  
              2，临时表只在当前会话有效，我们使用的服务器会定时清理临时表，大概5分钟左右  
              3，同一个语句中不能多次用到同一个临时表


### 关联查询 ###

- left join: 以左表为基准，去匹配右表的数据，如果匹配到就显示，匹配不到则为null。  
- right join：以右表为基准，去匹配左表的数据，如果左表匹配到就显示，匹配不到则为null。  
- inner join： 两张表能够互相匹配到的数据才显示，不存在任意一方就不会显示。  
- full outer join：mysql中没有完整链接。 


### 嵌套查询 ###

mysql支持嵌套查询，即任一查询结果都可以被以表或者值的形式与其他查询结果相关联；  
使用：能用简单sql语句分步骤实现的就不要使用关联查询，能用关联实现的就不要用嵌套查询。


### UNION 和UNION ALL ###

将具有相同列数的查询结果，合并成一个结果集，其中UNION会去除重复项而union all则不会。


### if语句 ###

    if  [search_condition] then [statement_list];
    else [statement_list];
    end if;  

### case语句 ###

    case when [search_condition] then [statement_list] else [statement_list] end;

注：一般if语句用于查询语句之外，case语句用于查询条件之中；


### 循环语句 ###
    
	while：while [search_condition] do [statement_list]  end while;
    repeat：repeat [statement_list] until [search_condition] end repeat;
    loop：lp:loop [statement_list]
                if [search_condition] then leave lp; end if;
        end loop;

说明：循环不能在普通的查询页面或者作为独立的sql命令使用，loop循环需要加标签且配合if判断执行；


## MySQL的数据定义语言DDL ##

### 结构创建create ###

    1. create database [databasename];		--创建数据库
    2. create table [tablename];		--创建数据表
    3. create FUNCTION [FUNCTIONname];		--创建函数
     ......

### 结构更改alter ###

    1. alter table [tablename] add [column] datatype;	-- 给表增加列
    2. alter table [tablename] drop [column];		-- 删除表中某一列
    3. 修改函数，存储过程等其他结构......
    
### 结构删除drop ###

    1. drop database [databasename];		--删除数据库
    2. drop table [tablename];		--删除数据表
    3. drop FUNCTION [FUNCTIONname];		--删除函数
     ......

### 结构查询明令show ###

    1. show tables; -- 显示当前数据库中所有表的名称。
    2. show databases; -- 显示mysql中所有数据库的名称。
    3. show columns from database_name.table_name; -- 显示表中列名称。

### 约束 ###

含义：
一种限制，用于限制表中的数据，为了保证表中数据的准确性和可靠性。  
常用的约束：   
	1. PRIMARY KEY：主键约束，它等于非空加唯一，它会默认创建聚集索引。  
	2. foreign key： 保证一个表中的数据匹配另一个表中的值的参照完整性。 
	3. NOT NULL ：非空，用于保证该字段的值不能为空。  
	4. auto_increment ：自增约束。  
	5. UNIQUE：唯一，用于保证该字段的值具有唯一性，可以为空。  
	6. DEFAULT：默认值，用于保证该字段有默认值。  


### 索引 ###

- 优点： 快速定位特定数据，加速表和表之间的连接，提高查询效率；
- 缺点：索引会占用空间，增加维护成本，影响修改数据的效率；
- 使用：在数据量大，或者常用的条件语句（where或者join）中使用；  
使用索引的注意事项：
	1. 索引不会包含有NULL的列
	2. mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作，尽量不要包含多个列的排序，如果需要最好给这些列建复合索引。
	3. 一般情况下不鼓励使用like操作，like ‘%aaa%’不会使用索引，而like ‘aaa%’可以使用索引。
	4. 不使用NOT IN 、<>、！=操作，但<,<=，=，>,>=,BETWEEN,IN是可以用到索引的
	5. 索引要建立在经常进行select操作且值比较唯一的字段上。
	6. where的查询条件里有不等号(where column != …),mysql将无法使用索引。
	7. 如果where字句的查询条件里使用了函数mysql将无法使用索引。
	8. 在join操作中(需要从多个数据表提取数据时)，mysql只有在主键和外键的数据类型相同时才能使用索引，否则即使建立了索引也不会使用。


### 触发器 ###

定义：简单的说，就是一张表发生了某写操作，然后自动触发了预先编写好的若干条SQL语句的执行；
```
CREATE TRIGGER [TRIGGER_name] AFTER INSERT ON [Master_table] FOR EACH ROW begin
INSERT INTO [Form_table](...) VALUES (...);
```


### 存储过程 ###

定义： 
一组可编程的函数，是为了完成特定功能的SQL语句集，经编译创建并保存在数据库中，用户可通过指定存储过程的名字并给定参数(需要时)来调用执行。  
优点：易维护，提高扩展性,安全性,执行效率,不需要发布程序或者重启服务  
缺点：可移植性差，增加服务器压力，不易调试。  
使用：简而言之，能用简单sql语句实现的就不要使用存储过程，较为复杂sql语句集（非业务逻辑），或者不愿将数据逻辑暴漏出来的时候，不可避免地需要频繁调用数据库的时候，使用存储过程是最明智的选择。


### 函数 ###

定义： 与其它语言中的函数一样，函数是指一段可以直接被另一段程序引用的程序，mysql中的函数与存储过程类似，都是一组SQL集， 但是函数可以嵌入到sql语句中使用，而存储过程不能；  
表值函数：返回数据是以数据表的形式；  
标量函数：返回数据是一个具体的值；  
系统函数：mysql系统中为我们定义好可以实现具体功能的一些函数；  
常见的函数举例:  
1. round(x,y)返回参数x的四舍五入的有y位小数的值  
2. count(col)返回指定列中非null值的个数  
3. now() 返回当前的日期和时间     
4. concat(s1,s2...,sn)将s1,s2...,sn连接成字符串  


### 视图 ###

存储的查询语句,当调用的时候,产生结果集,视图充当的是虚拟表的角色


***-- 原子性、一致性、隔离性、持久性***  

### 事务 ###
    定义： 一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务
              begin/START TRANSACTION；开启一个事务
              commit；提交事务
              rollback；事务回滚
              SAVEPOINT；事务保存点

### 锁LOCK ###



- 锁的种类   
 
	1. 表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低。  
	2. 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。  
	3. 页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般  


- 锁的操作
   
	1. 读操作：不会阻塞其他用户对同一表请求，但会阻塞对同一表的写请求；  
	2. 写操作：会阻塞其他用户对同一表的读和写操作；  
	3. 加表级锁：LOCK TABLES tbl_name {READ | WRITE}；  
	4. 解表级锁： UNLOCK TABLES；  

锁必须在一个事务内部使用。


### mysql中的事件event ###

事件(event)是用于执行定时或周期性的任务；

事件的创建脚本以及格式模板示例：

	CREATE DEFINER=`ycbz2020`@`%` EVENT `HandleInsertOilwell_prod_data` ON SCHEDULE EVERY 1 DAY STARTS '2020-04-25 16:00:00' ON COMPLETION PRESERVE ENABLE DO BEGIN  
    DECLARE r_code CHAR(5) DEFAULT '00000';  
    DECLARE r_msg TEXT;  
    DECLARE v_error INTEGER;  
    DECLARE v_starttime DATETIME DEFAULT NOW();  
    DECLARE v_randno INTEGER DEFAULT FLOOR(RAND()*100001);  
      
    INSERT INTO t_event_history (dbname,eventname,starttime,randno)   
    #修改下面的作业名
    VALUES(DATABASE(),'HandleInsertOilwell_prod_data', v_starttime,v_randno);    
      
    BEGIN  
        #异常处理段  
        DECLARE CONTINUE HANDLER FOR SQLEXCEPTION    
        BEGIN  
            SET  v_error = 1;  
            GET DIAGNOSTICS CONDITION 1 r_code = RETURNED_SQLSTATE , r_msg = MESSAGE_TEXT;  
        END;  
          
        #此处为实际调用的执行内容
        CALL HandleInsertOilwell_prod_data();  
    END;  
      
    UPDATE t_event_history SET endtime=NOW(),issuccess=ISNULL(v_error),duration=TIMESTAMPDIFF(SECOND,starttime,NOW()),
    errormessage=CONCAT('error=',r_code,', message=',r_msg),randno=NULL WHERE starttime=v_starttime AND randno=v_randno;  
      
	END


## MySQL的数据控制语言DCL ##

定义：是一种可对数据访问权进行控制的指令，它可以控制特定用户账户对数据表、查看表、存储程序、用户自定义函数等数据库对象的控制权。主要由 授权（GRANT） 和 取消权限（REVOKE） 指令组成。

常用权限：  

    - CONNECT  
    - SELECT  
    - INSERT  
    - UPDATE  
    - DELETE  
    - EXECUTE  
    - USAGE  
    - REFERENCES  

语法格式：
  
    GRANT [权限] ON [database] TO [user] WITH [授权选项];
    revoke [权限] on [database] from [user];

示例链接：
(https://www.jb51.cc/mysql/530264.html)

### 备份的分类 ###

**按操作方法分为**  

- 物理备份：即直接复制数据文件；  
- 逻辑备份：将数据导出至文本文件中；  

**按操作时间分为**  

- 冷备份：关闭数据库的情况下进行的备份  
- 热备份：在数据库运行的情况下进行的备份  

**按备份的内容分为**  

- 完整备份：备份所有数据库文件；  
- 增量备份：备份与上一备份节点发生变化的部分


### 逻辑备份工具 mysqldump ###

**一般推荐使用mysqldump逻辑备份，适用于中小型数据库系统**

- 备份整库：`mysqldump -u 用户名 -p 数据库名 > 存放路径\dev.sql `

示例：备份远程数据库文件

    mysqldump -h rm-2ze5f4464d3agjr48fo.mysql.rds.aliyuncs.com -P 3306 -u DevStudy -p devstudy > D:\data\log\dev1.sql

- 恢复整库：`mysql -<username> -p<password> <dbname> < backpath/back.sql`

示例：

    mysql> create database devstudy;
    Query OK, 1 row affected (0.03 sec)
    
    mysql> quit
    Bye
    
    C:\Program Files\MySQL\bin>mysql -u root -p devstudy < D:\data\log\dev1.sql
    Enter password: ******

