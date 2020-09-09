## ServiceStack OrmLite 学习记录

### 1.   数据库链接方式示例

```C#
//链接字符串
​  private string connectionString =@"Server=rm-2ze5f4464d3agjr48fo.mysql.rds.aliyuncs.com; User Id=ycbz2020;Password=wtras$22@123!;Database=ycbz2020;Pooling=true;MinPoolSize=0;MaxPoolSize=200";
//链接数据库
​  var dbFactory = new OrmLiteConnectionFactory(connectionString, MySqlDialect.Provider);
​  using (var mydb = dbFactory.Open())
        {
                 //do something
                 //
        }
```

 

### 2.   数据库的增、删、改、查、异步操作示例

#### 2.1.  新增

```C#
//插入Insert、InsertOnly
//插入所有对象值

​  db.Insert(new Person { Id = 1, FirstName = "Jimi", LastName = "Hendrix", Age = 27 });

//插入部分字段

​  db.InsertOnly(() => new Person { FirstName = "Amy" });

//类SQL写法

​  var q = db.From<Person>()
	​  .Insert(p => new { p.FirstName });
​  db.InsertOnly(new Person { FirstName = "Amy" }, onlyFields: q)

 
```



#### 2.2.  更新

##### 2.2.1、Update

```c#
//1、更新除ID以外的所有字段

​ db.Update(new Person { Id = 1, FirstName = "Jimi", LastName = "Hendrix", Age = 27});

//2、如果您提供自己的where表达式，它将更新每个字段（包括ID），但改用您的过滤器：

//按条件更新(包括更新ID)

​ db.Update(new Person { Id = 1, FirstName = "JJ" }, p => p.LastName == "Hendrix");

//3、One way to limit the fields which gets updated is to use an Anonymous Type:

//限制要更新的字段的一种方法是使用匿名类型：(只更新条件为LastName = "Hendrix" 的 FirstName="JJ")

​ db.Update<Person>(new { FirstName = "JJ" }, p => p.LastName == "Hendrix");

// 4、Or by using UpdateNonDefaults which only updates the non-default values in your model using the filter specified

//或使用UpdateNonDefaults，它仅使用指定的过滤器更新模型中的非默认值：

​ db.UpdateNonDefaults(new Person { FirstName = "JJ" }, p => p.LastName == "Hendrix");

```

##### 2.2.2、UpdateOnly

```c#
//由于更新部分行是Db的常见用例，因此，我们为此目的添加了许多方法，称为UpdateOnly。
//The lambda syntax lets you update only the fields listed in property initializers, erg
//Lambda语法可让您仅更新属性初始化程序中列出的字段，例如

​  db.UpdateOnly(() => new Person { FirstName = "JJ" });

//The second argument lets you specify a filter for updates
//第二个参数使您可以指定更新过滤器：按条件更新

​  db.UpdateOnly(() => new Person { FirstName = "JJ" }, where: p => p.LastName == "Hendrix");

//an UpdateOnly statement is used to specify which fields should be updated
//更新指定字段的几种方式

​  db.UpdateOnly(new Person { FirstName = "JJ" }, onlyFields: p => p.FirstName);
​  db.UpdateOnly(new Person { FirstName = "JJ", Age = 12 }, onlyFields: p => new { p.FirstName, p.Age });
​  db.UpdateOnly(new Person { FirstName = "JJ", Age = 12 },  onlyFields: p => new[] { "Name", "Age" });

// When present, the second expression is used as the where filter
//按条件更新指定字段的方式

​  db.UpdateOnly(new Person { FirstName = "JJ" },  onlyFields: p => p.FirstName,  where: p => p.LastName == "Hendrix");
 
//除了使用上面的表达式过滤器之外，您还可以选择使用SqlExpression构建器，该构建器在您要以编程方式构造update语句时提供更大的灵活性：
//instead of using the expression filters above you can choose to use an SqlExpression builder which provides more flexibility when you want to programatically construct the update statementl

​  var q = db.From<Person>().Update(p => p.FirstName);
​  db.UpdateOnly(new Person { FirstName = "JJ", LastName = "Hendo" }, onlyFields: q);

//Using an Object Dictionary 使用对象字典
​  var updateFields = new Dictionary<string,object> {
    [nameof(Person.FirstName)] = "JJ",
};
​  db.UpdateOnly<Person>(updateFields, p => p.LastName == "Hendrix");
 
//Using a typed SQL Expression 使用类型化的SQL表达式 

​  var q = db.From<Person>().Where(x => x.FirstName == "Jimi").Update(p => p.FirstName);
​  db.UpdateOnly(new Person { FirstName = "JJ" }, onlyFields: q);

```
##### 2.2.3、UpdateAdd

```c#
UpdateAdd API提供了几种用于更新现有值的Typed API
 
//Increase everyone's Score by 3 points 原有数值的基础上+3
​  db.UpdateAdd(() => new Person { Score = 3 }); 
//Remove 5 points from Jackson Score 原有数值的基础上-3
​  db.UpdateAdd(() => new Person { Score = -5 }, where: x => x.LastName == "Jackson");
//Graduate everyone and increase everyone's Score by 2 points +2的另一个字段同时修改为true 
​  db.UpdateAdd(() => new Person { Points = 2, Graduated = true });
//Add 10 points to Michael's score 按条件+10
​  var q = db.From<Person>().Where(x => x.FirstName == "Michael");
​  db.UpdateAdd(() => new Person { Points = 10 }, q);

```

注意：UpdateAdd语句中的任何非数字值（例如字符串）都将正常替换。
Any non-numeric values in an UpdateAdd statement (e.g. strings) are replaced as normal.

#### 2.3.  查询

##### 2.3.1、基础查询 

```c#
CEg.     
​  int agesAgo = DateTime.Today.AddYears(-20).Year;

//按条件查询
​  db.Select<Author>(x => x.Birthday >= new DateTime(agesAgo, 1, 1) && x.Birthday <= new DateTime(agesAgo, 12, 31));
//返回某字段及对应数据
​  db.Select<Author>(x => Sql.In(x.City, "London", "Madrid", "Berlin"));
​  db.Select<Author>(x => x.Earnings <= 50);
//模糊查询
​  db.Select<Author>(x => x.Name.StartsWith("A"));//以"A"开始
​  db.Select<Author>(x => x.Name.EndsWith("garzon"));//以"garzon"结束
​  db.Select<Author>(x => x.Name.Contains("Benedict"));//包含"Benedict"
​  db.Select<Author>(x => x.Rate == 10 && x.City == "Mexico");
​  db.Select<Author>(x => x.Rate.ToString() == "10");  
​  db.Select<Author>(x => "Rate " + x.Rate == "Rate 10");  //元数据进行拼接查询
​  Person person = db.SingleById<Person>(1); //通过ID：1获取一条数据
​  Person person = db.Single<Person>(x => x.Age == 42);// 获取一条数据
//获取条数
​  var q = db.From<Person>().Where(x => x.Age > 40).Select(Sql.Count("*"));//返回Count
​  int peopleOver40 = db.Scalar<int>(q);
​  int peopleUnder50 = db.Count<Person>(x => x.Age < 50);
//Scalar\Count使用有什么区别？（没什么区别，两者中获取数据的方法）
 
​  bool has42YearOlds = db.Exists<Person>(new { Age = 42 });
//按条件获取最大值
​  int maxAgeUnder50 = db.Scalar<Person, int>(x => Sql.Max(x.Age), x => x.Age < 50);
//查询字符串数据结果放入string集合
​  var q = db.From<Person>().Where(x => x.Age == 27).Select(x => x.LastName);//只返回LastName 
​  List<string> results = db.Column<string>(q);
//不相同内容的条数
​  var q = db.From<Person>().Where(x => x.Age < 50).Select(x => x.Age);
​  HashSet<int> results = db.ColumnDistinct<int>(q);
//直接放入Dictionary字典
​   var q = db.From<Person>().Where(x => x.Age < 50).Select(x => new { x.Id, x.LastName });//只返回Id、LastName
​  Dictionary<int,string> results = db.Dictionary<int, string>(q);
//直接放入Dictionary字典自动分组
​  var q = db.From<Person>().Where(x => x.Age < 50).Select(x => new { x.Age, x.LastName });//只返回Age、LastName
​  Dictionary<int, List<string>> results = db.Lookup<int, string>(q);
//直接放入键值对
 
​  var q = db.From<StatsLog>().GroupBy(x => x.Name)
    .Select(x => new { x.Name, Count = Sql.Count("*") })//只返回Name和每组个数
    .OrderByDescending("Count");
​  var results = db.KeyValuePairs<string, int>(q);

//您还可以查询JOIN表上的POCO引用，例如：
​  var q = db.From<Table>()
    ​  .Join<Join1>()
    ​  .Join<Join1, Join2>()
   ​  .Where(x => !x.IsValid.HasValue && 
        ​  x.Join1.IsValid &&
        ​  x.Join1.Join2.Name == theName &&
        ​  x.Join1.Join2.IntValue == intValue)
    ​  .GroupBy(x => x.Join1.Join2.IntValue)
    ​  .Having(x => Sql.Max(x.Join1.Join2.IntValue) != 10)
    ​  .Select(x => x.Join1.Join2.IntValue);

//通过TableAlias API，您可以在将同一张表多次连接在一起时指定表别名，以区别于具有多个自引用连接的查询中的任何歧义列，例如：
​  var q = db.From<Page>(db.TableAlias("p1"))
    .Join<Page>((p1, p2) => p1.PageId == p2.PageId && p2.ActivityId == activityId, db.TableAlias("p2"))
    .Join<Page,Category>((p2,c) => Sql.TableAlias(p2.Category) == c.Id)
    .Join<Page,Page>((p1,p2) => Sql.TableAlias(p1.Rank,"p1") < Sql.TableAlias(p2.Rank,"p2"))
    .Select<Page>(p => new {
        ​ ActivityId = Sql.TableAlias(p.ActivityId, "p2")
       ​ });
​  var rows = db.Select(q);



```

##### 2.3.2、主子表查询存储



```C#
public class Customer
{
    [AutoIncrement]
    public int Id { get; set; }
    public string Name { get; set; }

    [Reference] // Save in CustomerAddress table
    public CustomerAddress PrimaryAddress { get; set;}

    [Reference] // Save in Order table
    public List<Order> Orders { get; set; }
}

public class CustomerAddress
{
    [AutoIncrement]
    public int Id { get; set; }
    public int CustomerId { get; set; } //`{Parent}Id` convention to refer to Customer
    public string AddressLine1 { get; set; }
    public string AddressLine2 { get; set; }
    public string City { get; set; }
    public string State { get; set; }
    public string Country { get; set; }
}

public class Order
{
    [AutoIncrement]
    public int Id { get; set; }
    public int CustomerId { get; set; } //`{Parent}Id` convention to refer to Customer
    public string LineItem { get; set; }
    public int Qty { get; set; }
    public decimal Cost { get; set; }
}
```

主子表查询示例

```C#
//查询 Customers 子集中Qty大于10的 Customers 数据集
var q = db.From<Customer>()
          .Join<Order>()
          .Where<Order>(o => o.Qty >= 10)
          .SelectDistinct();
var customers = db.Select<Customer>(q);

//查询条件Qty大于10的Order 数据集
var orders = db.Select<Order>(o => o.Qty >= 10);
// 合并散乱的的与 Customers 有关系 Orders数据
customers.Merge(orders); 
//打印数据
customers.PrintDump();   

The Load* API's are used to automatically load a POCO and all it's child references, e.g:

var customer = db.LoadSingleById<Customer>(customerId);
Using Typed SqlExpressions:

var customers = db.LoadSelect<Customer>(x => x.Name == "Customer 1");



//主子表存储
var customer =  new Customer {
    Name = "Customer 1",
    PrimaryAddress = new CustomerAddress {
        AddressLine1 = "1 Australia Street",
        Country = "Australia"
    },
    Orders = new[] {
        new Order { LineItem = "Line 1", Qty = 1, Cost = 1.99m },
        new Order { LineItem = "Line 2", Qty = 2, Cost = 2.99m },
    }.ToList(),
};

db.Save(customer, references:true);

```



##### 2.3.2、多表联合(自定义字段) 

```c#
//多表联合多条件判断查询，返回自定义字段    
​  var q = dbFac.From<IotOilOmeter>();
​  q.Join<IotWellField>((om, wf) => om.WellFieldId == wf.WellFieldId);
​  Expression<Func<IotOilOmeter, IotWellField, bool>> exp = null;
​  if (requst.DepId > 0)
​  {
	​  exp = (om, wf) => wf.DepId == requst.DepId;
		​  q.And(exp);
	​  }
​  if (requst.WellFieldId > 0)
​  {
	​  exp = (om, wf) => wf.WellFieldId == requst.WellFieldId;
		​  q.And(exp);
	​  }
​  q.LeftJoin<IotDataOilOmeterLatest>((om, ol) => om.OilOmeterId == ol.OilOmeterId);
​  q.Select<IotOilOmeter, IotWellField, IotDataOilOmeterLatest>((om, wf, ol) => new
​  {
	​  om,//所有 IotOilOmeter字段
	​  wf.WellFieldName,
	​  wf.DepId,
	​  ol.CurrentElevation
​  });
​  var list = dbFac.Select<DepIotOilOmeter>(q);//将所有自定义字段放入List<DepIotOilOmeter>中相匹配的字段
​  var sql = dbFac.GetLastSql();//获取最终sql语句
 
//用.WithSqlFilter进行自定义
​  var q = db.From<Table>().Where(x => x.Age == 27).WithSqlFilter(sql => sql + " option (recompile)");
 
​  var q = db.From<Table>().Where(x => x.Age == 27).WithSqlFilter(sql => sql + " WITH UPDLOCK");
​  var results = db.Select(q);

//左联查询排序取前10条
​  var q = dbFac.From<LogExecutor>()
    .LeftJoin<AppUserAuth>((a, b) => a.UserId == b.Id.ToString())
    .Where(a => a.StartTime >= DateTime.Parse(DateTime.Now.AddDays(-7).ToString("yyyy-MM-dd")))
    .GroupBy(a => a.UserId)
    .Select<LogExecutor, AppUserAuth>((a, b) => new { RunCount = Sql.Count("*"), UserName = b.UserName })
    .OrderByDescending(a => Sql.Count("*")).Take(10);
​  var qlist= dbFac.Select<UserWeekRanking>(q);
 
​  var q = dbFac.From<OilWellMonitorDataLatest, IotWellField>((a, b) => a.WellFieldId == b.WellFieldId && b.DepId == request.DepId)
    .Select<OilWellMonitorDataLatest, IotWellField>((a, b) => new
          {
              Id = a.id,
              WellFieldId = a.WellFieldId,
              WellFieldName = a.well_field_name,
              DepId = b.DepId,
                        ......
           });
​  var qlist= dbFac.Select<object>(q);

```

笔记：GetLastSql()方法不能能用与直接的SQL查询，该方法为获取执行的完整SQL

##### **2.3.3、格式化查询**

```c#
  db.SelectFmt<Person>("Age > {0}", 40);
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT \"Id\", \"FirstName\", \"LastName\", \"Age\" FROM \"Person\" WHERE Age > 40"));
     
​  db.SelectFmt<Person>("SELECT * FROM Person WHERE Age > {0}", 40);
Assert.That(db.GetLastSql(), Is.EqualTo("SELECT * FROM Person WHERE Age > 40"));
```

 这两种方法是等效的，还可以通过拼接字符串的方法，构造出复杂的查询条件，第一个参数中可以加多个项{0} {1} {2} 

#### 2.4.  删除

```c#
//按条件删除
​  db.Delete<Person>(p => p.Age == 27);
//类型化的SQL表达式
​  var q = db.From<Person>()
	​  .Where(p => p.Age == 27);
​  db.Delete<Person>(q);
//基于字符串的表达式
​  db.Delete<Person>(where: "Age = @age", new { age = 27 });
//使用联接表按条件查询
​  var q = db.From<Person>()
	​  .Join<PersonJoin>((x, y) => x.Id == y.PersonId)
	​  .Where<PersonJoin>(x => x.Id == 2);
​  db.Delete(q);
 
```

#### 2.5.  关于异步

//现在，基本上，OrmLite的大多数公共API都具有同名的异步等效项以及一个附加的常规* Async后缀。 Async API还带有一个可选的CancellationToken，使转换同步代码变得微不足道，您只需在其中添加Async后缀和await关键字即可，如客户订单UseCase升级到Async diff所示，

//例如：

```C#
  await db.InsertAsync(new Employee { Id = 1, Name = "Employee 1" });
​  await db.SaveAsync(product1, product2);
​  var customer = await db.SingleAsync<Customer>(new { customer.Email });
```

### 3.   RDBMS 关系型数据库操作

#### 3. 1、创建实体

```c#
     class Poco 
     {
        public int Id { get; set; }
         public string Name { get; set; }
        public string Ssn { get; set; }
     }
```

#### 3. 2、生成数据库表

```c#
  db.DropTable<Poco>();//删除表
​  db.TableExists<Poco>(); //是否存在表= false
​  db.CreateTable<Poco>(); 
​  db.TableExists<Poco>(); //= true
​  db.ColumnExists<Poco>(x => x.Ssn); //= true
​  db.DropColumn<Poco>(x => x.Ssn);
​  db.ColumnExists<Poco>(x => x.Ssn); //字段是否存在= false
 
     class Poco 
     {
        public int Id { get; set; }
      public string Name { get; set; }
 
         [Default(0)]
         public int Age { get; set; }
     }
 
​  if (!db.ColumnExists<Poco>(x => x.Age)) //= false
	​  db.AddColumn<Poco>(x => x.Age);
​  db.ColumnExists<Poco>(x => x.Age); //= true
```

#### 3. 3、部分api

```c#
  db.AlterTable<Poco>
​  db.AddColumn<Poco>
​  db.AlterColumn<Poco>
​  db.ChangeColumnName<Poco>
​  db.DropColumn<Poco>
​  db.AddForeignKey<Poco>
​  db.DropForeignKey<Poco>
​  db.CreateIndex<Poco>
​  db.DropIndex<Poco>
 
```



 