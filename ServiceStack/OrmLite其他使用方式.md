# ServiceStack OrmLite 数据库查询 几个实用方法 

 #### 几种查询方式

**执行SQL语句：** 

```c#
int result = db.SqlScalar<int>("SELECT OBJECT_ID(@name)", new { name = "SomeName" });
```

 

**继承表的实现 （存储于同一个表中）** 

```c#
[Alias("Table")]    
public abstract class MyBaseClass
{
    public String Name { get; set; }
    public String Name1 { get; set; }
    public String Name2 { get; set; }
}

[Alias("Table")]    
public class MyDerivedClassA : MyBaseClass {}

[Alias("Table")]    
public class MyDerivedClassB : MyBaseClass {}
```



 **格式化查询：** 

```c#
db.SelectFmt<Person>("Age > {0}", 40);
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT \"Id\", \"FirstName\", \"LastName\", \"Age\" FROM \"Person\" WHERE Age > 40"));
     
db.SelectFmt<Person>("SELECT * FROM Person WHERE Age > {0}", 40);
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT * FROM Person WHERE Age > 40"));
```

 这两种方法是等效的，还可以通过拼接字符串的方法，构造出复杂的查询条件，第一个参数中可以加多个项{0} {1} {2} 



 **返回指定两个字段到字典：** 

```c#
db.Lookup<int, string>(db.From<Person>().Select(x => new { x.Age, x.LastName }).Where(q => q.Age < 50));
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT \"Age\",\"LastName\" \nFROM \"Person\"\nWHERE (\"Age\" < 50)"));
      
db.Lookup<int, string>("SELECT Age, LastName FROM Person WHERE Age < @age", new { age = 50 });
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT Age, LastName FROM Person WHERE Age < @age"));
      
db.LookupFmt<int, string>("SELECT Age, LastName FROM Person WHERE Age < {0}", 50);
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT Age, LastName FROM Person WHERE Age < 50"));
```

 两种方式等效，可以使用类指定字段名，也可以字符串中指定 



#### 局部更新UpdateOnly

更新表中的部分字段是一个很常用的情形, SS提供有一个方法实现这个功能, 叫做 **UpdateOnly**.

 `UpdateOnly` 表达式中第二个参数用于指定哪个字段被修改:

```c#
db.UpdateOnly(new Person { FirstName = "JJ" }, p => p.FirstName);
```

 **UPDATE "Person" SET "FirstName" = 'JJ'** 

```c#
db.UpdateOnly(new Person { FirstName = "JJ", Age = 12 }, 
    onlyFields: p => new { p.FirstName, p.Age });
```

 **UPDATE "Person" SET "FirstName" = 'JJ', "Age" = 12** 

 

第三个表达式可以指定where子句: 

```c#
db.UpdateOnly(new Person { FirstName = "JJ" }, 
    onlyFields: p => p.FirstName, 
    where: p => p.LastName == "Hendrix");
```

 **UPDATE "Person" SET "FirstName" = 'JJ' WHERE ("LastName" = 'Hendrix')** 



 如果不使用表达式过滤器，也可以使用  ExpressionVisitor 构造器，当你想程序化构造更新表达式时，可以更加灵活: 

```c#
db.UpdateOnly(new Person { FirstName = "JJ", LastName = "Hendo" }, 
  onlyFields: q => q.Update(p => p.FirstName));
```

 **UPDATE "Person" SET "FirstName" = 'JJ'** 

```c#
db.UpdateOnly(new Person { FirstName = "JJ" }, 
  onlyFields: q => q.Update(p => p.FirstName).Where(x => x.FirstName == "Jimi"));
```

 

**UPDATE "Person" SET "FirstName" = 'JJ' WHERE ("LastName" = 'Hendrix')** 

 如果你想使用无限自由的不确定类型模式，基于字符串的表达式，使用 `.Params()`扩展方法对参数转义 (inspired by [massive](https://github.com/robconery/massive)): 

```c#
db.Update<Person>(set: "FirstName = {0}".Params("JJ"), 
                where: "LastName = {0}".Params("Hendrix"));
```

 甚至表名可以是一个字符串， 这样你就可以指定和上面功能相同的 update 操作彻底不需要使用Person模型: 

```c#
db.Update(table: "Person", set: "FirstName = {0}".Params("JJ"), 
          where: "LastName = {0}".Params("Hendrix"));
```

 **UPDATE "Person" SET FirstName = 'JJ' WHERE LastName = 'Hendrix'** 



#### **查询并且生成集合：** 

1 直接返回字典类型结果

```c#
Dictionary<int, string> trackIdNamesMap = db.Dictionary<int, string>(
    "select Id, Name from Track")
```

结构是字典类型，其中的Key来自于数据表的Id字段，value来自于数据表的Name字段，其中的Key也可以使用string类型的字段。value也可以使用其他类型字段。



2 直接返回字典类型结果 ， 并且将结果分组

Lookup returns an Dictionary<K, List<V>> made from the first two columns:

```c#
Dictionary<int, List<string>> albumTrackNames = db.Lookup<int, string>(
    "select AlbumId, Name from Track")
```

 结果是字典类型，其中的键是来自数据表的AlbumId字段，value是一个列表，这个类别是根据前边的AlbumId字段进行分组的，每个键值下有数目不等的Name的列表。 