## Mysql存储过程语法相关

### 一、条件语句if-then-else:

```mysql
create procedure demo_1(in param int)
begin
declare var int;
set var=param-1;
if var=0 then
insert into userinfo(name) values('demo');
else
insert into userinfo(name) values('demo');
end if;

if param=0 then
update userinfo set age=param where name='demo';
else
update userinfo set age=1 where name='demo';
end if;
end
//
```

仅作语法参考并无实际意义。

### 二、case语句：

```mysql
create procedure demo_2(in param int)
begin
case param
when 0 then
insert into userinfo(name) values('demo_0');
when 1 then
insert into userinfo(name) values('demo_1');
else
insert into userinfo(name) values('demo_default');
end case;
end
//
```



调用的话可以：

```mysql
call demo_2(1)//
```

### 三、循环语句while-endwhile

```mysql
create procedure demo_3(in param int)
begin
while param<10 do
insert into userinfo(name) values('demo');
set param=param+1;
end while;
end
//
```



### 四、repeat...end repeat：

它在执行操作后检查结果，而while则是执行前进行检查。相当于do...while



```mysql
create procedure demo_4(in param int)
begin
repeat
insert into userinfo(name) values('demo');
set param=param+1;
until param>10
end repeat;
end
//
```



until表示满足后边的条件才继续循环

### 五、loop...end loop语句

```mysql
create procedure demo_5(in param int)
begin
demo_5:loop
insert into userinfo(name) values('demo_5');
set param=param+1;
if param>10 then
leave demo_5;
end if;
end loop;
end
//
```



call demo_5(7)则输入插入4个demo_5。

### 六、LABLES标号：

标号可以用在begin repeat while 或者loop 语句前，语句标号只能在合法的语句前面使用。可以跳出循环，使运行指令达到复合语句的最后一步。

 

### 七、ITERATE:

通过引用复合语句的标号,来从新开始复合语句



```mysql
create procedure demo_6(in param int)
begin
declare v int;
set v=0;
LOOP_LABLE:loop
if v=3 then
set v=v+1;
ITERATE LOOP_LABLE;
end if;
insert into userinfo(name) values('demo_6');
set v=v+1;
if v>=5 then
leave LOOP_LABLE;
end if;
end loop;
end
//
```



## **MySQL**存储过程的基本函数

### **1、操作字符串类**

```mysql
CHARSET(str) //返回字串字符集
CONCAT (string2 [,... ]) //连接字串
INSTR (string ,substring ) //返回substring首次在string中出现的位置,不存在返回0
LCASE (string2 ) //转换成小写
LEFT (string2 ,length ) //从string2中的左边起取length个字符
LENGTH (string ) //string长度
LOAD_FILE (file_name ) //从文件读取内容
LOCATE (substring , string [,start_position ] ) 同INSTR,但可指定开始位置
LPAD (string2 ,length ,pad ) //重复用pad加在string开头,直到字串长度为length
LTRIM (string2 ) //去除前端空格
REPEAT (string2 ,count ) //重复count次
REPLACE (str ,search_str ,replace_str ) //在str中用replace_str替换search_str
RPAD (string2 ,length ,pad) //在str后用pad补充,直到长度为length
RTRIM (string2 ) //去除后端空格
STRCMP (string1 ,string2 ) //逐字符比较两字串大小,
SUBSTRING (str , position [,length ]) //从str的position开始,取length个字符,
```


注：mysql中处理字符串时，默认第一个字符下标为1，即参数position必须大于等于1 

```mysql
1. mysql> select substring('abcd',0,2);  
2. +-----------------------+  
3. | substring('abcd',0,2) |  
4. +-----------------------+  
5. |            |  
6. +-----------------------+  
7. 1 row in set (0.00 sec)  
8.  
9. mysql> select substring('abcd',1,2);  
10. +-----------------------+  
11. | substring('abcd',1,2) |  
12. +-----------------------+  
13. |   ab        |  
14. +-----------------------+  
15. 1 row in set (0.02 sec)  

TRIM([[BOTH|LEADING|TRAILING] [padding] FROM]string2) //去除指定位置的指定字符
UCASE (string2 ) //转换成大写
RIGHT(string2,length) //取string2最后length个字符
SPACE(count) //生成count个空格
```



### **2、**数学类

```mysql
ABS (number2 ) //绝对值
BIN (decimal_number ) //十进制转二进制
CEILING (number2 ) //向上取整
CONV(number2,from_base,to_base) //进制转换
FLOOR (number2 ) //向下取整
FORMAT (number,decimal_places ) //保留小数位数
HEX (DecimalNumber ) //转十六进制
注：HEX()中可传入字符串，则返回其ASC-11码，如HEX('DEF')返回4142143
也可以传入十进制整数，返回其十六进制编码，如HEX(25)返回19
LEAST (number , number2 [,..]) //求最小值
MOD (numerator ,denominator ) //求余
POWER (number ,power ) //求指数
RAND([seed]) //随机数
ROUND (number [,decimals ]) //四舍五入,decimals为小数位数]
```

注：返回类型并非均为整数，如：
(1)默认变为整形值

```mysql
1. mysql> select round(1.23);  
2. +-------------+  
3. | round(1.23) |  
4. +-------------+  
5. |      1 |  
6. +-------------+  
7. 1 row in set (0.00 sec)  
8.  
9. mysql> select round(1.56);  
10. +-------------+  
11. | round(1.56) |  
12. +-------------+  
13. |      2 |  
14. +-------------+  
15. 1 row in set (0.00 sec) 
```





(2)可以设定小数位数，返回浮点型数据



```mysql
1. mysql> select round(1.567,2);  
2. +----------------+  
3. | round(1.567,2) |  
4. +----------------+  
5. |      1.57 |  
6. +----------------+  
7. 1 row in set (0.00 sec) 

SIGN (number2 ) //
```



 

### **3、**日期时间类

```mysql
ADDTIME (date2 ,time_interval ) //将time_interval加到date2
CONVERT_TZ (datetime2 ,fromTZ ,toTZ ) //转换时区
CURRENT_DATE ( ) //当前日期
CURRENT_TIME ( ) //当前时间
CURRENT_TIMESTAMP ( ) //当前时间戳
DATE (datetime ) //返回datetime的日期部分
DATE_ADD (date2 , INTERVAL d_value d_type ) //在date2中加上日期或时间
DATE_FORMAT (datetime ,FormatCodes ) //使用formatcodes格式显示datetime
DATE_SUB (date2 , INTERVAL d_value d_type ) //在date2上减去一个时间
DATEDIFF (date1 ,date2 ) //两个日期差
DAY (date ) //返回日期的天
DAYNAME (date ) //英文星期
DAYOFWEEK (date ) //星期(1-7) ,1为星期天
DAYOFYEAR (date ) //一年中的第几天
EXTRACT (interval_name FROM date ) //从date中提取日期的指定部分
MAKEDATE (year ,day ) //给出年及年中的第几天,生成日期串
MAKETIME (hour ,minute ,second ) //生成时间串
MONTHNAME (date ) //英文月份名
NOW ( ) //当前时间
SEC_TO_TIME (seconds ) //秒数转成时间
STR_TO_DATE (string ,format ) //字串转成时间,以format格式显示
TIMEDIFF (datetime1 ,datetime2 ) //两个时间差
TIME_TO_SEC (time ) //时间转秒数]
WEEK (date_time [,start_of_week ]) //第几周
YEAR (datetime ) //年份
DAYOFMONTH(datetime) //月的第几天
HOUR(datetime) //小时
LAST_DAY(date) //date的月的最后日期
MICROSECOND(datetime) //微秒
MONTH(datetime) //月
MINUTE(datetime) //分返回符号,正负或0
SQRT(number2) //开平方
```

