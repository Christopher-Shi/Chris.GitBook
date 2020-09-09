## mysql-存储过程基础

### 1、什么是存储过程

为以后的使用而保存的一条或多条MySQL语句的集合。
存储过程思想上就是数据库 SQL 语言层面的代码封装与重用。

### 2、为什么要使用存储过程

1. 把处理封装在容易使用的单元中，简化复杂的操作
2. 防止错误保证了数据的一致性
3. 简化对变动的管理。(修改对应表名、列名等修改对应存储过程的代码，对于使用的人不需要知道变化)
4. 提高性能
5. 灵活

总的来说是简单、安全、高性能
缺点：

1. 编写比SQL语句复杂
2. 权限问题（可能无权、一般都是使用存储过程、没有创建存储过程的权限）

### 3、创建存储过程

```
CREATE  PROCEDURE productpricing()
BEGIN
SELECT Avg(prod_price) AS priceaverage
FROM products;
END
```

注意：在命令行中输入的问题

```
mysql> delimiter //
mysql> CREATE PROCEDURE productpricing()
    -> BEGIN
    -> SELECT Avg(prod_price) AS priceaverage
    -> FROM products;
    -> END //
```

### 4、使用存储过程

存储过程实际上是一种函数

```
CALL productpricing();
```

### 5、删除存储过程

```
    drop procedure productpricing；
    drop procedure if EXISTS productpricing；
```

### 6、使用参数

一般，存储过程并不显示结果，而是把结果返回给你指定的变量
变量(variable)内存中一个特定的位置，用来临时存储数据。

```
CREATE PROCEDURE productpricing(
    OUT p1 DECIMAL(8,2),
    OUT ph DECIMAL(8,2),
    OUT pa DECIMAL(8,2)
)
BEGIN
SELECT MIN(prod_price)
INTO p1
FROM products;
SELECT MAX(prod_price)
INTO ph
FROM products;
SELECT avg(prod_price)
INTO pa
FROM products;
END;
```

关键字OUT指出相应的参数用来从存储过程传出 一个值(返回给调用者)。
MySQL支持IN(传递给存储过程)、
OUT(从存 储过程传出，如这里所用)
INOUT(对存储过程传入和传出)类型的参 数。

变量名 所有MySQL变量都必须以@开始。
**调用存储过程**

```
call productpricing(@pricelow,@pricehign,@priceaverage);
```

**查询**

```
SELECT @priceaverage;
SELECT @priceaverage,@pricehign,@pricelow;
```

**使用in和out**
创建

```
CREATE PROCEDURE ordertotal(
    IN onumber INT,
    OUT ototal DECIMAL(8,2)
)
BEGIN
SELECT sum(item_price*quantity)
FROM orderitems
WHERE order_num = onumber
INTO ototal;
END;
```

调用

```
call ordertotal(20005,@total);
```

查询

```
select @total;
```

### 7、建立智能存储过程

迄今为止使用的所有存储过程基本上都是封装 MySQL简单的SELECT语句。虽然它们全都是有效的存储过程例子，但它们所能完成的工作你直接用这些被封装的语句就能完成（如果说它们还能带来更多的东西。那就是使事情更复杂）。只有在存储过程内包含业务规则和智能处理时，它们的威力才真正显现出来。

```
   考虑这个场景。你需要获得与以前一样的订单合计，但需要对合计增加营业税，不过只针对某些顾客（或许是你所在州中那些顾客）。那么，你需要做下面几件事情：
   1、获得合计（和以前一样）
   2、把营业税有条件的添加到合计
   3、返回合计（带或不带税的）
```

我们输入如下代码：

```
-- Name: ordertotal        //   添加注释
-- Parameters: onumber = order number
--             taxable = 0 if not taxable, 1 if taxtable
--             ototal = order total variable

CREATE     PROCEDURE ordertotal (
IN onumber INT,
IN taxable BOOLEAN,
OUT ototal DECIMAL(8,2)
) COMMENT 'Obtain order total, optionally adding tax'
BEGIN
    
        -- Declare variable for total
        DECLARE total DECIMAL(8,2);     //   声明变量   
        -- Declare tax percentage
        DECLARE taxrate INT DEFAULT 6;
        
        -- Get the order total
        SELECT Sum(item_price*quantity)
        FROM orderitems
        WHERE order_num = onumber
        INTO total;
        
        -- Is this taxable?
        IF taxable THEN
            -- yes,so add taxrate to the total
            SELECT total+(total/100*taxrate) INTO total;
        END IF;
        --  And finally, save to out variable
        SELECT total INTO ototal;
END;
此存储过程有很大的变动。首先，增加了注释（前面放置 --）。在存储过程复杂性增加时，这样做特别重要。  
添加了另外一个参数 taxable，它是一个布尔值（如果要增加税则为真，否则为假）。  
在存储过程体中，用 DECLARE语句定义了两个局部变量。 DECLARE要求指定变量名和数据类型，
它也支持可选的默认值（这个例子中的 taxrate的默认被设置为 6%）。SELECT 语句变，因此其结果存储到 total（局部变量）而不是 ototal。  
IF 语句检查taxable是否为真，如果为真，则用另一SELECT语句增加营业税到局部变量 total。

最后，用另一SELECT语句将total（它增加或许不增加营业税）保存到 ototal。  
注意：COMMENT关键字 ，本例子中的存储过程在 CREATE PROCEDURE语句中包含了一个 COMMENT值。  
它不是必需的，但如果给出，将在SHOW PROCEDURE STATUS的结果中显示。

这显然是一个更高级，功能更强的存储过程。为试验它，请用以下两条语句：  
第一条：
call ordertotal(20005, 0, @total);
SELECT @total;
输出：
+--------+
| @total |
+--------+
|  38.47 |
+--------+
第二条：
call ordertotal(20009, 1，@total);
SELECT @total;
输出：
+--------+
| @total |
+--------+
|  36.21 |
+--------+
BOOLEAN值指定为1 表示真，指定为 0表示假（实际上，非零值都考虑为真，只有 0被视为假）。通过给中间的参数指定 0或1 ，可以有条件地将营业税加到订单合计上。
```

这个例子给出了 MySQL的IF 语句的基本用法。 IF语句还支持 ELSEIF和ELSE 子句（前者还使用 THEN子句，后者不使用）。在以后章节中我们将会看到 IF的其他用法（以及其他流控制语句）。

### 8、检查存储过程

为显示用来创建一个存储过程的CREATE语句

```
show create PROCEDURE ordertotal;
```

为了获得包括何时、由谁创建等详细信息的存储过程列表

```
show procedure status;
```

表比较多，用like过滤

```
show procedure status like 'ordertotal';
```

### 