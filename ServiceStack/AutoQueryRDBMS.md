# AutoQuery RDBMS

### 一、概述

AutoQuery RDBMS仅通过POCO Request DTO类定义就可以快速开发高性能，可完全查询的类型化RDBMS数据驱动服务，并[支持](https://github.com/ServiceStack/ServiceStack.OrmLite#8-flavours-of-ormlite-is-on-nuget)基于[OrmLite与RDBMS无关的高性能API的](https://github.com/ServiceStack/ServiceStack.OrmLite)[大多数主要RDBMS](https://github.com/ServiceStack/ServiceStack.OrmLite#8-flavours-of-ormlite-is-on-nuget)。

需要强调的重要一点是，AutoQuery服务只是普通的ServiceStack服务，使用相同的请求管道，可以映射到任何用户定义的路由，以所有注册格式可用，并可以从现有的类型化服务客户端使用。

除了利用ServiceStack的现有功能之外，以这种方式最大化重用还可以减少开发人员在实现、定制、自省和消费ServiceStack服务时重用其现有知识所需的认知开销

### 二、入门指南

像ServiceStack的其他可组合特性一样，自动查询特性是通过注册一个插件来启用的；如下所示

```C#
Plugins.Add(new AutoQueryFeature { MaxLimit = 100 });//MaxLimit选项确保每个查询AutoQuery最大返回条数设置为100行
```

这就是启用自动查询功能所需要的全部内容。AutoQueryFeature在ServiceStack中。服务器NuGet包，其中包含增值功能，利用任何OrmLite和Redis，可以添加到你的项目

```
PM> Install-Package ServiceStack.Server
```

如果您没有配置OrmLite，它可以通过指定您默认的数据库和连接字符串一起注册:

```C#
container.Register<IDbConnectionFactory>(new OrmLiteConnectionFactory(":memory:", SqliteDialect.Provider));
```

上面的配置注册了一个内存Sqlite数据库，尽管自动查询测试套件在所有支持的RDBMS提供程序中工作，你可以自由选择使用你注册的数据库。

现在一切都配置好了，我们可以创建第一个服务了。为了实现OData的电影评级查询的理想API，我们只需要为我们的服务定义请求DTO

```C#
[Route("/movies")]
public class FindMovies : QueryDb<Movie>
{
    public string[] Ratings { get; set; }
}
```

路由使用如下：

```html
/movies?ratings=G,PG-13
```

客户端请求方式：

客户端继续受益于类型化API因为它是常规的DTO请求，我们还免费获得端到端类型化API

```C组
var movies = client.Get(new FindMovies { Ratings = new[]{"G","PG-13"} })
```

虽然这提供了我们想要的理想API，但所需的最小代码只是使用希望查询的表声明一个新的请求DTO

```C#
public class FindMovies : QueryDb<Movie> {}
```

就像ServiceStack中的其他请求DTO一样，使用ServiceStack预定义的路由

### 三、了解IQuery

QueryDb<T>只是一个抽象类，它实现IQuery<T>并返回类型化的QueryResponse<T>：

```C#
public abstract class QueryDb<T> : QueryDb, 
    IQuery<T>, IReturn<QueryResponse<T>> { }

public interface IQuery
{
    int? Skip { get; set; }           // 指定逃过多少行
    int? Take { get; set; }           // 指定最多返回多少行数据
    string OrderBy { get; set; }      // 指定字段升序
    string OrderByDesc { get; set; }  // 指定字段降序
    string Include { get; set; }      // 指定聚合查询以包括
    string Fields { get; set; }       // 指定返回数据的字段
}
```

IQuery接口包括所有查询都支持的共享特性，并充当接口标记，告诉ServiceStack为每个请求DTO创建查询服务。

### 四、自定义AutoQuery实现

查询行为可以完全定制，只需提供自己的服务实现代替，例如:

```C#
// 覆写自定义实现
public class MyQueryServices : Service
{
    public IAutoQueryDb AutoQuery { get; set; }

    // 同步
    public object Any(FindMovies query)
    {
        using var db = AutoQuery.GetDb(query, base.Request);
        var q = AutoQuery.CreateQuery(query, base.Request, db);
        return  AutoQuery.Execute(query, q, base.Request, db);
    }

    // 异步
    public async Task<object> Any(QueryRockstars query)
    {
        using var db = AutoQuery.GetDb(query, base.Request);
        var q = AutoQuery.CreateQuery(query, base.Request, db);
        return await AutoQuery.ExecuteAsync(query, q, base.Request, db);
    }    
}
```

这基本上是自动查询特性为每个IQuery请求DTO生成的内容，除非已经存在针对请求DTO的服务，在这种情况下，它使用现有的实现。

#### 1、了解AutoQuery的实现过程

获取这个自动查询请求的DB连接:

```C#
using var db = AutoQuery.GetDb(query, base.Request);
```

默认情况下，它遵循用于解析约定的DB连接，同样您可以使用自己的自定义解析来覆盖该约定。

创建一个填充类型SqlExpression:

```C#
var q = AutoQuery.CreateQuery(query, base.Request, db);
```

同事也可以使用下面这种剪短写法

```C#
Dictionary<string,string> queryArgs = Request.GetRequestParams();// 获取所有请求参数条件
var q = AutoQuery.CreateQuery(dto, queryArgs, Request, db);
```

它从请求DTO上的类型化属性以及HTTP请求QueryString或FormData上的任何无类型化键/值对构造一个OrmLite SqlExpression。

此时，您可以检查AutoQuery构造的SqlExpression，或者附加任何额外的自定义条件来限制请求的范围，比如将结果限制为经过身份验证的用户等。

从请求构造查询之后，就是执行它了

```C#
return AutoQuery.Execute(dto, q, Request);
```



由于实现是琐碎的，我们可以显示实现内联:

```C#
public QueryResponse<Into> Execute<Into>(
    IDbConnection db, ISqlExpression query)
{
    var q = (SqlExpression<From>)query;
    return new QueryResponse<Into>
    {
        Offset = q.Offset.GetValueOrDefault(0),
        Total = (int)db.Count(q),
        Results = db.LoadSelect<Into, From>(q),
    };
}
```

本上只返回QueryResponse<Into>响应DTO中的结果。我们还看到，每个查询发出两个请求，一个用于检索查询的总计数，另一个用于实际结果，这可能受到请求的Skip或Take或配置的AutoQueryFeature.MaxLimit的限制。

#### 2、 返回自定义结果

为了在自定义响应DTO中指定返回结果，你可以从QueryDb< from，Into>便利基类:

```C#
public abstract class QueryDb<From, Into> : QueryDb, IQuery<From, Into>, IReturn<QueryResponse<Into>> { }
```

正如我们从类定义中可以看出的那样，它告诉AutoQuery您想要查询from，但是返回的结果是指定的into类型。这允许能够返回一组经过策划的列，例如:

```C#
public class QueryCustomRockstars : QueryDb<Rockstar, CustomRockstar> {}

public class CustomRockstar
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int? Age { get; set; }
    public string RockstarAlbumName { get; set; }
}
```

在上面的示例中，我们只返回结果的子集。像RockstarAlbumName这样不匹配的属性会被忽略，从而可以重用定制的DTO用于不同的查询。

#### 3、返回嵌套的相关结果（Reference）

自动查询还利用了OrmLite的引用支持，让你返回带注释属性的相关子记录，例如:

```c#
public class QueryRockstars : QueryDb<Rockstar> {}

public class Rockstar
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int? Age { get; set; }

    [Reference]
    public List<RockstarAlbum> Albums { get; set; } 
}
```

#### 4、连接表（Joining Tables）

AutoQuery允许我们利用OrmLite最近对加入类型化sqlexpress的支持

我们可以告诉AutoQuery连接多个表使用IJoin<T1,T2>接口标记:

```
public class QueryRockstarAlbums 
  : QueryDb<Rockstar,CustomRockstar>, IJoin<Rockstar,RockstarAlbum>
{
    public int? Age { get; set; }
    public string RockstarAlbumName { get; set; }
}
```

上面的例子告诉AutoQuery使用在两个表之间隐式的OrmLite引用约定来查询Rockstar和RockstarAlbum表的内部连接。

请求DTO允许我们跨联接表查询字段，其中每个字段与包含该字段的第一个表匹配。您可以使用完全限定的{Table}{Field}约定来匹配字段，例如针对RockstarAlbum的RockstarAlbumName.name 列。

这个字段的映射也适用于响应DTO，现在来自上述CustomRockstar类型的RockstarAlbumName将被填充:

```C#
public class CustomRockstar
{
    ...
    public string RockstarAlbumName { get; set; }
}
```

#### 5、多表连接

使用适当的 Join<，>通用接口来指定跨多个表的连接。AutoQuery支持连接多达5个表与一个IJoin<>接口，例如: IJoin<T1, T2, T3, T4, T5>

或者，任意数量的表可以通过使用多个IJoin<>接口注释请求DTO来连接在一起，例如:

```C#
IJoin<T1,T2>,IJoin<T2,T3>,IJoin<T3,T4>,//...
```

#### 6、支持左联

你可以通过使用ILeftJoin<，>标记接口来告诉AutoQuery使用一个左连接，例如:

```C#
public class QueryRockstarAlbumsLeftJoin : QueryDb<Rockstar, CustomRockstar>, 
    ILeftJoin<Rockstar, RockstarAlbum>
{
    public int? Age { get; set; }
    public string AlbumName { get; set; }
}
```



### 五、可定制的特定的查询

到目前为止的查询显示了Request DTO字段的默认行为，该字段使用=操作数来匹配查询表上的字段，从而执行精确匹配。高级查询可以通过下面示例中所示的可配置属性来实现，该示例使用显式模板来启用自定义查询，例如：

```C#
public class QueryRockstars : QueryDb<Rockstar>
{
    //默认为 'AND FirstName = {Value}'
    public string FirstName { get; set; }  

    //集合默认为'FirstName IN ({Values})'
    public string[] FirstNames { get; set; } 

    [QueryDbField(Operand = ">=")]
    public int? Age { get; set; }

    [QueryDbField(Template = "UPPER({Field}) LIKE UPPER({Value})", Field = "FirstName")]
    public string FirstNameCaseInsensitive { get; set; }

    [QueryDbField(Template="{Field} LIKE {Value}", Field="FirstName", ValueFormat="{0}%")]
    public string FirstNameStartsWith { get; set; }

    [QueryDbField(Template="{Field} LIKE {Value}", Field="LastName", ValueFormat="%{0}")]
    public string LastNameEndsWith { get; set; }

    [QueryDbField(Template = "{Field} BETWEEN {Value1} AND {Value2}", Field="FirstName")]
    public string[] FirstNameBetween { get; set; }

    [QueryDbField(Term = QueryTerm.Or)]
    public string LastName { get; set; }
}
```

我们将介绍每个示例，以便更好地了解可用的定制。

#### 1、对集合的查询

ServiceStack属性不区分大小写，并允许使用逗号分隔的语法填充集合，因此以下查询:

```
/rockstars?firstNames=A,B,C
```

将填充字符串数组

```C#
public string[] FirstNames { get; set; } 
```

默认情况下，IEnumerable属性是使用SQL {Field} IN ({Values})模板查询的，该模板将字符串数组转换为适合SQL使用的引号和转义逗号分隔的值列表，如下所示:

```C#
"FirstName" IN ('A','B','C')
```

{Field}属性根据它匹配的OrmLite字段定义被替换，对于具有[Alias("FName")]属性的联接表，该属性将被替换为全限定名e。例如:"Table"."FName"。

自动查询还支持多元化版本，它通过减少字段名的s作为回退，这就是FirstNames能够匹配FirstName表属性的方式。

#### 2、可定制的操作数

最简单的定制之一是改变在查询中使用的操作数，例如:

```C#
[QueryDbField(Operand = ">=")]
public int? Age { get; set; }
```

将改变Age的比较:

```C#
"Age" >= {Value}
```

#### 3、可定制的模板

通过提供定制模板，更大的定制实现，通过坚持TSQL兼容片段将确保它被每个RDBMS供应商支持，例如:

```C#
[QueryDbField(Template = "UPPER({Field}) LIKE UPPER({Value})",Field="FirstName")]
public string FirstNameCaseInsensitive { get; set; }
```

如上所述，{Field}变量的占位符被替换为符合条件的表字段，而{Value}被引用并转义以防止SQL注入。

Field="FirstName"告诉AutoQuery忽略属性名并使用指定的字段名。

#### 4、格式化值

你可以使用ValueFormat修饰符在引号内的{Value}占位符中插入自定义字符，我们使用这个占位符来插入带有引号的值的%通配符，使查询可以开始和结束，例如:

```C#
[QueryDbField(Template="{Field} LIKE {Value}", 
            Field="FirstName", ValueFormat="{0}%")]
public string FirstNameStartsWith { get; set; }

[QueryDbField(Template="{Field} LIKE {Value}", 
            Field="LastName", ValueFormat="%{0}")]
public string LastNameEndsWith { get; set; }
```



#### 5、两者之间 BETWEEN  AND 

格式化多个值的另一种方法是使用Value{N}变量占位符，它允许支持多个值的语句，如SQL 's BETWEEN语句:

```C#
[QueryDbField(Template = "{Field} BETWEEN {Value1} AND {Value2}", 
            Field = "FirstName")]
public string[] FirstNameBetween { get; set; }
```

#### 6、改变查询的行为

默认情况下，查询的作用类似于过滤器，并且每个条件都与和布尔术语组合在一起，以进一步过滤结果集。可以通过指定Term=QueryTerm将其更改为在字段级别使用OR。或修饰词,如:

```C#
[QueryDbField(Term=QueryTerm.Or)]
public string LastName { get; set; }
```

然而，你的API应该努力保持相同的行为和调用语义，建议在API级别改变行为，这样每个属性都能保持一致的行为，例如:

```c#
[QueryDb(QueryTerm.Or)]
public class QueryRockstars : QueryDb<Rockstar>
{
    public int[] Ids { get; set; }
    public List<int> Ages { get; set; }
    public List<string> FirstNames { get; set; }
}
```

在本例中，每个属性都包含在内，其中指定的每个值都添加到返回的结果集。

#### 7、动态特性

.NET属性通常是用属性定义的，而ServiceStack也允许动态添加属性，允许它们在其他地方定义，从DTO中分离，例如:

```C#
typeof(QueryRockstars)
    .GetProperty("Age")
    .AddAttributes(new QueryDbFieldAttribute { Operand = ">=" });
```

#### 8、隐式的约定

上面的示例展示了如何使用显式属性在具体属性上定义特殊显式查询。虽然这对于理解如何创建定制的查询是必要的，具体的属性和显式的属性可以通过使用内置的约定完全避免，在AutoQueryFeature插件上可配置。

使用隐式约定，我们可以回到用空体定义请求DTO:

```
public class QueryRockstars : QueryDb<Rockstar> {}
```

内置约定允许使用基于约定的命名来使用自描述属性查询具有预期行为的字段。为了更详细地探索这一点，让我们看看内置的约定是如何定义的:

```c#
const string GreaterThanOrEqualFormat = "{Field} >= {Value}";
const string GreaterThanFormat =        "{Field} >  {Value}";
const string LessThanFormat =           "{Field} <  {Value}";
const string LessThanOrEqualFormat =    "{Field} <= {Value}";
const string NotEqualFormat =           "{Field} <> {Value}";
const string IsNull =                   "{Field} IS NULL";
const string IsNotNull =                "{Field} IS NOT NULL";

ImplicitConventions = new Dictionary<string, string> 
{
    {"%Above%",         GreaterThanFormat},
    {"Begin%",          GreaterThanFormat},
    {"%Beyond%",        GreaterThanFormat},
    {"%Over%",          GreaterThanFormat},
    {"%OlderThan",      GreaterThanFormat},
    {"%After%",         GreaterThanFormat},
    {"OnOrAfter%",      GreaterThanOrEqualFormat},
    {"%From%",          GreaterThanOrEqualFormat},
    {"Since%",          GreaterThanOrEqualFormat},
    {"Start%",          GreaterThanOrEqualFormat},
    {"%Higher%",        GreaterThanOrEqualFormat},
    {">%",              GreaterThanOrEqualFormat},
    {"%>",              GreaterThanFormat},
    {"%!",              NotEqualFormat},
    {"<>%",             NotEqualFormat},

    {"%GreaterThanOrEqualTo%", GreaterThanOrEqualFormat},
    {"%GreaterThan%",          GreaterThanFormat},
    {"%LessThan%",             LessThanFormat},
    {"%LessThanOrEqualTo%",    LessThanOrEqualFormat},
    {"%NotEqualTo",            NotEqualFormat},

    {"Behind%",         LessThanFormat},
    {"%Below%",         LessThanFormat},
    {"%Under%",         LessThanFormat},
    {"%Lower%",         LessThanFormat},
    {"%Before%",        LessThanFormat},
    {"%YoungerThan",    LessThanFormat},
    {"OnOrBefore%",     LessThanOrEqualFormat},
    {"End%",            LessThanOrEqualFormat},
    {"Stop%",           LessThanOrEqualFormat},
    {"To%",             LessThanOrEqualFormat},
    {"Until%",          LessThanOrEqualFormat},
    {"%<",              LessThanOrEqualFormat},
    {"<%",              LessThanFormat},

    {"Like%",           "UPPER({Field}) LIKE UPPER({Value})"},
    {"%In",             "{Field} IN ({Values})"},
    {"%Ids",            "{Field} IN ({Values})"},
    {"%Between%",       "{Field} BETWEEN {Value1} AND {Value2}"},
            
    {"%IsNull",         IsNull},
    {"%IsNotNull",      IsNotNull},
};

EndsWithConventions = new Dictionary<string, QueryDbFieldAttribute>
{
    { "StartsWith", new QueryDbFieldAttribute { 
          Template= "UPPER({Field}) LIKE UPPER({Value})", ValueFormat= "{0}%" 
    	}
    },
    { "Contains", new QueryDbFieldAttribute { 
          Template= "UPPER({Field}) LIKE UPPER({Value})",ValueFormat= "%{0}%" 
    	}
    },
    { "EndsWith", new QueryDbFieldAttribute { 
          Template= "UPPER({Field}) LIKE UPPER({Value})", ValueFormat= "%{0}" 
    	}
    },
};
```

上面字典中的字符串键描述了每个规则映射到的属性。我们将通过几个例子来看看如何使用它们:

下面的要求就像你期望的那样，可以返回所有年龄超过42岁的摇滚明星。

```
/rockstars?AgeOlderThan=42
```

它通过匹配规则工作，因为AgeOlderThan确实以OlderThan结尾:

```C#
{"%OlderThan", "{Field} > {Value}"},
```

通配符%是解析为Age的字段名的占位符。既然已经找到了匹配，它就使用已定义的{Field} > {Value}模板创建一个查询来实现所需的行为。

如果你想使用包含的>=操作数，我们可以使用一个规则与>=模板:

```
/rockstars?AgeGreaterThanOrEqualTo=42
```

哪一个符合规则:

```C#
{"%GreaterThanOrEqualTo%", "{Field} >= {Value}"},
```

当通配符在模式的两端:

```C#
{"%GreaterThan%", "{Field} > {Value}"},
```

因此，可以字段名，它匹配这两个规则:

```
/rockstars?AgeGreaterThan=42
/rockstars?GreaterThanAge=42
```

另一个替代使用冗长的后缀是内置的简短语法:

```C#
>Age Age >= {Value}
Age> Age > {Value}
<Age Age < {Value}
Age< Age <= {Value}
```

根据< >操作符出现在字段名之前还是之后，使用合适的操作数:

```
/rockstars?>Age=42
/rockstars?Age>=42
/rockstars?<Age=42
/rockstars?Age<=42
```

使用{Values}或Value{N}格式指定查询应该作为一个集合处理，例如:

```C#
{"%Ids",       "{Field} IN ({Values})"},
{"%In",        "{Field} IN ({Values})"},
{"%Between%",  "{Field} BETWEEN {Value1} AND {Value2}"},
```

它允许在QueryString上指定多个值:

```
/rockstars?Ids=1,2,3
/rockstars?FirstNamesIn=Jim,Kurt
/rockstars?FirstNameBetween=A,F
```

#### 9、更高级的约定

更高级的约定可以直接在StartsWithConventions和EndsWithConventions字典中指定，这些字典允许使用完整的[QueryDbField]属性定制，例如:

```C#
EndsWithConventions = new Dictionary<string, QueryDbFieldAttribute>
{
    { "StartsWith", new QueryDbFieldAttribute { 
          Template = "UPPER({Field}) LIKE UPPER({Value})", ValueFormat = "{0}%" }},
    { "Contains", new QueryDbFieldAttribute { 
          Template = "UPPER({Field}) LIKE UPPER({Value})", ValueFormat = "%{0}%" }},
    { "EndsWith", new QueryDbFieldAttribute { 
          Template = "UPPER({Field}) LIKE UPPER({Value})", ValueFormat = "%{0}"
    }},
};
```

正常使用示例：

```
/rockstars?FirstNameStartsWith=Jim
/rockstars?LastNameEndsWith=son
/rockstars?RockstarAlbumNameContains=e
```

这还表明，使用完全限定的{table}{Field}引用约定，隐式约定也可以应用于像RockstarAlbumNameContains这样的连接表字段。

#### 10、明确的约定

虽然隐式约定可以在一个空的“无约定”DTO上调用，你也可以为你最终使用的约定指定具体的属性:

```C#
public class QueryRockstars : QueryDb<Rockstar>
{
    public int? AgeOlderThan { get; set; }
    public int? AgeGreaterThanOrEqualTo { get; set; }
    public int? AgeGreaterThan { get; set; }
    public int? GreaterThanAge { get; set; }
    public string FirstNameStartsWith { get; set; }
    public string LastNameEndsWith { get; set; }
    public string LastNameContains { get; set; }
    public string RockstarAlbumNameContains { get; set; }
    public int? RockstarIdAfter { get; set; }
    public int? RockstarIdOnOrAfter { get; set; }
}
```

先发优势的客户机请求

AutoQuery方法中隐藏的一个不太明显的优点是，当具体的属性被添加到服务器甚至还不知道的客户端请求DTO时，所有的事情仍然可以工作!这是可能的，因为服务客户端生成相同的HTTP请求，匹配隐式约定，使它可以在客户端上执行新的抢占式类型查询，而不必重新部署服务器。虽然在大多数情况下，客户机和服务器共享相同的dto，这不大可能是一个流行的用例，但在需要时，这是一个令人惊讶的好特性。

定义良好的服务契约的优点

规范化你最终使用的约定的好处是，它们可以被ServiceStack的类型服务客户端使用:

```C#
var response = client.Get(new QueryRockstars { AgeOlderThan = 42 });
```

以及使你的API自我描述，并让你访问所有的ServiceStack的元数据服务，包括:

- [Metadata pages](https://docs.servicestack.net/metadata-page)
- [Swagger UI](https://docs.servicestack.net/swagger-api)
- [Postman Support](https://docs.servicestack.net/postman)

这就是为什么我们建议在将API部署到生产环境之前，将您想要允许的约定形式化。

当发布你的API时，你也可以通过禁用无类型的隐式约定来确定只有显式约定被使用:

```C#
Plugins.Add(new AutoQueryFeature { EnableUntypedQueries = false });//禁用无类型的隐式约定来确定只有显式约定被使用
```

#### 11、原始SQL过滤

还可以指定原始SQL过滤器。在提供更大灵活性的同时，它们也面临着OData表达式所存在的许多问题。

原始SQL过滤器也不能从限制对已声明字段的匹配和值的自动转义和引号中获益，尽管它们仍然通过OrmLite的illegalsqlfragmenttoken进行验证，以防止SQL注入攻击。但是由于他们仍然允许调用SQL函数，原始SqlFilters不应该被启用时，不受信任的方访问，除非他们已经配置使用一个命名的OrmLite DB连接，这是只读和锁定，以便它只能访问它所允许的。

如果这样做是安全的，RawSqlFilters可以启用:

```C#
Plugins.Add(new AutoQueryFeature { 
    EnableRawSqlFilters = true,
    UseNamedConnection = "readonly",
    IllegalSqlFragmentTokens = { "ProtectedSqlFunction" },
});
```

一旦启用，你可以使用_select， _from， _where修饰符来添加自定义SQL片段，例如:

```
/rockstars?_select=FirstName,LastName

/rockstars?_where=Age > 42
/rockstars?_where=FirstName LIKE 'Jim%'

/rockstars?_select=FirstName&_where=LastName = 'Cobain'

/rockstars?_from=Rockstar r INNER JOIN RockstarAlbum a ON r.Id = a.RockstarId
```

注意:为了可读性，上面例子中的url编码省略了。

虽然原始SqlFilters会影响执行的查询，但它们不会改变返回的响应DTO。

### 六、进阶学习记录

#### 1、自定义字段 Custom Fields

你也可以自定义你想要返回的字段使用字段属性在所有自动查询服务，例如:

```
?Fields=Id,Name,Description,JoinTableId
```

字段仍然需要在响应DTO上定义，因为此特性不会更改响应DTO模式，只会更改填充的字段。这确实改变了要执行的底层RDBMS选择，还受益于RDBMS和App服务器之间的带宽减少。

在指定自定义字段时，你可以添加一个有用的JSON自定义是ExcludeDefaultValues，例如:

```
/query?Fields=Id,Name,Description,JoinTableId&jsconfig=ExcludeDefaultValues
```

这将从JSON响应中删除任何默认值的值类型字段，例如:

- [github.servicestack.net/repos.json?fields=Name,Homepage,Language,Updated_At](http://github.servicestack.net/repos.json?fields=Name,Homepage,Language,Updated_At)
- [github.servicestack.net/repos.json?fields=Name,Homepage,Language,Updated_At&jsconfig=ExcludeDefaultValues](http://github.servicestack.net/repos.json?fields=Name,Homepage,Language,Updated_At&jsconfig=ExcludeDefaultValues)

#### 2、通配符

可以使用通配符快速引用表上的所有字段。* 格式,如:

```
?fields=id,departmentid,department,employee.*
```

这是一个扩展到手动列出Employee表中的每个字段的速记，对于连接多个表的查询很有用，例如:

```C#
[Route("/employees", "GET")]
public class QueryEmployees : QueryDb<Employee>,
    IJoin<Employee, EmployeeType>,
    IJoin<Employee, Department>,
    IJoin<Employee, Title> 
{
    //...
}
```

#### 3、获取字段不相同的值

要只查询具有不同字段的结果，可以在自定义字段列表前加上distinct前缀，例如

使用查询字符串:

```
?Fields=DISTINCT City,Country
```

使用c#客户端:

```C#
var response = client.Get(new QueryCustomers { 
    Fields = "DISTINCT City,Country"
})
```

我们可以在Northwind现有的自动查询请求dto中使用此功能:

```
[Route("/query/customers")]
public class QueryCustomers : QueryDb<Customer> { }
```

返回所有独特城市和国家的北风客户:

[northwind.netcore.io/query/customers?Fields=DISTINCT City,Country](http://northwind.netcore.io/query/customers?Fields=DISTINCT City,Country)

或者只是返回他们所在的国家:

[northwind.netcore.io/query/customers?Fields=DISTINCT Country](http://northwind.netcore.io/query/customers?Fields=DISTINCT Country)

#### 4、分页和排序

在核心IQuery接口上定义的一些功能，对所有查询都可用，可以对结果进行页面和排序:

```C#
public interface IQuery
{
    int? Skip { get; set; }
    int? Take { get; set; }
    string OrderBy { get; set; }
    string OrderByDesc { get; set; }
}
```

这工作如你所期望的，你可以修改返回的结果集:

```
/rockstars?skip=10&take=20&orderBy=Id
```

或通过ServiceClient访问时:

```C#
client.Get(new QueryRockstars { Skip=10, Take=20, OrderBy="Id" });
```

在对结果进行分页时，将使用IQuery的主键添加隐式OrderBy，以确保返回结果的可预测顺序。您可以通过指定自己的OrderBy来更改使用的OrderBy:

```C#
client.Get(new QueryRockstars { Skip=10, Take=20, OrderByDesc="Id" });
```

或删除默认行为:

```C#
Plugins.Add(new AutoQueryFeature { 
    OrderByPrimaryKeyOnPagedQuery = false 
});
```

#### 5、多字段排序

AutoQuery还支持用逗号分隔的字段名列表指定多个OrderBy，例如:

```
/rockstars?orderBy=Id,Age,FirstName
```

同样，客户端访问：

```C#
client.Get(new QueryRockstars { OrderBy = "Id,Age,FirstName" });
```

以相反的顺序对特定字段排序

当指定多个顺序的by 's时，你可以通过在字段名前加上-来反向排序特定字段，例如:

```
?orderBy=Id,-Age,FirstName
?orderByDesc=-Id,Age,-FirstName
```

#### 6、服务客户端支持 Service Clients Support

使用类型化DTO来定义服务契约的一个主要好处是，它允许使用ServiceStack的.net服务客户端，这为大多数.net和PCL客户端平台提供端到端的API，而不需要代码生成。

随着查询中更丰富的语义可用，我们已经能够增强服务客户端的新的GetLazy()的API，它允许延迟响应流提供大型结果集的透明分页，例如:

```C#
var results = client.GetLazy(new QueryMovies { 
    Ratings = new[]{"G","PG-13"}
}).ToList();
```

由于GetLazy返回一个懒惰的IEnumerable<T>序列，它也可以在LINQ表达式中使用：

```C#
var top250 = client.GetLazy(new QueryMovies { 
        Ratings = new[]{ "G", "PG-13" } 
    })
    .Take(250)
    .ConvertTo(x => x.Title);
```

#### 7、内置CsvFormat

Mime Types and Content-Negotiation

作为常规的ServiceStack服务，我们从自动查询服务中得到的另一个好处是利用了ServiceStack的内置格式。

CSV格式在这里特别出色，因为查询返回单个表格结果集，这使它非常适合CSV。在许多方面，CSV是最可互操作的数据格式之一，因为大多数数据导入和操作程序(包括数据库和电子表格)都对CSV提供原生支持，允许深度和无缝集成。

ServiceStack提供了许多方法来请求你喜欢的内容类型，最简单的是使用。{format}在/pathinfo的结尾，例如:

```
/rockstars.csv
/movies.csv?ratings=G,PG-13
```

CSV格式响应可以使用相同范围的自定义响应JSON允许类型化结果排除默认值列时返回有限的自定义字段`?fields`:

- Camel Humps Notation: `?jsconfig=edf`
- Full configuration: `?jsconfig=ExcludeDefaultValues`

#### 8、命名连接

自动查询也可以很容易地配置，以查询任何数量的不同的数据库注册在您的AppHost。

在下面的例子中，我们配置我们的主RDBMS使用SQL Server和注册一个命名连接来指向一个报告PostgreSQL RDBMS:

```C#
var dbFactory = new OrmLiteConnectionFactory(connString, SqlServer2012Dialect.Provider);
container.Register<IDbConnectionFactory>(dbFactory);

dbFactory.RegisterConnection("Reporting", pgConnString, PostgreSqlDialect.Provider);
```

任何普通的自动查询服务如QueryOrders将使用默认的SQL服务器连接，而QuerySales将在PostgreSQL报告数据库上执行查询:

```C#
public class QueryOrders : QueryDb<Order> {}

[ConnectionInfo(NamedConnection = "Reporting")]
public class QuerySales : QueryDb<Sales> {}
```

在自动查询请求DTO中指定[ConnectionInfo]请求过滤器属性的另一种方法是在POCO表中指定指定的连接，例如:

```C#
[NamedConnection("Reporting")]
public class Sales { ... }

public class QuerySales : QueryDb<Sales> {}
```

#### 9、自动查询中包括聚合

AutoQuery支持在查询的结果集上运行其他聚合查询。要在查询响应中包含聚合，请在自动查询请求DTO的include属性中指定它们，例如:

```C#
var response = client.Get(new Query Rockstars { Include = "COUNT(*)" })
```

或者在包括查询字符串参数，如果你调用自动查询服务从浏览器，例如:

```
/rockstars?include=COUNT(*)
```

然后在QueryResponse<T>中发布结果。元字符串字典和是可访问的:

```
response.Meta["COUNT(*)"] //= 7
```

默认情况下，SQL聚合白名单中的任何函数都可以被引用:

```
AVG, COUNT, FIRST, LAST, MAX, MIN, SUM
```

可以通过修改SqlAggregateFunctions集合e来添加或删除。例如您可以允许使用一个CustomAggregate SQL函数:

```
Plugins.Add(new AutoQueryFeature { 
    SqlAggregateFunctions = { "CustomAggregate" }
})
```

#### 10、聚集查询使用

聚合函数的语法是按照它们在SQL中的用法建模的，因此它们非常熟悉。最基本的用法是，你可以指定聚合函数的名称，它将使用*作为默认参数，所以你也可以查询COUNT(*):

```
?include=COUNT
```

它还支持SQL别名:

```sql
COUNT(*) Total
COUNT(*) as Total
```

这是用来改变什么关键的结果是保存为:

```
response.Meta["Total"]
```

可以按列名，如下：

```
COUNT(LivingStatus)
```

如果一个参数匹配主要表中的一列文字按原样参考使用,如果它匹配表中加入一列取代它的完全限定参考和当它不匹配任何列,数据传递初始否则它自动转义,援引和传入的字符串。

独特的修饰语也可以使用，所以一个复杂的例子类似于:

```
COUNT(DISTINCT LivingStatus) as UniqueStatus
```

将上述函数的结果保存在:

```
response.Meta["UniqueStatus"]
```

可以将任意数量的聚合函数组合在逗号分隔的列表中:

```
Count(*) Total, Min(Age), AVG(Age) AverageAge
```

返回结果如下:

```
response.Meta["Total"]
response.Meta["Min(Age)"]
response.Meta["AverageAge"]
```

#### 11、包含总数 Include Total

一个查询可用的总记录可以通过在QueryString中添加来包含在响应中，例如:

```
/query?Include=Total
```

或根据DTO的要求客户端请求:

```
var response = client.Get(new MyQuery { Include = "Total" });
```

或者配置所有请求都返回Total：

```C#
Plugins.Add(new AutoQueryFeature {
    IncludeTotal = true
})
```

查询性能说明：

AutoQuery将所有其他聚合函数(如Total)组合在一起，并在同一个查询中执行它们以获得最佳性能。

### 七、混合AutoQuery服务

Hybrid AutoQuery Services

​		通过创建自定义服务实现来修改自动查询从传入请求自动填充的SqlExpression查询，可以很容易地增强AutoQuery服务。除了使用OrmLite的类型化API来执行标准的DB查询之外，您还可以利用定制SQL片段的高级RDBMS特性。作为示例，我们将查看techstack的实现。io基本QueryPosts服务，支持techstack中的每个Post feed，它的自定义实现继承了QueryDb<Post>自动查询服务的所有可查询功能，并为任何技术id添加了高级功能，是用于后台查询多个列的自定义高级属性。

除了继承所有默认的查询功能在一个QueryDb<Post>自动查询服务，自定义实现还:

1. 防止返回任何已删除的 Post
2. `closed`除非查询专门针对关闭的标签或状态，否则防止返回任何具有状态的 Post。
3. 通过使用PostgreSQL高级数组数据类型查询post  `string` 标签或int技术id，避免任何表连接。
4. 使用Anytechnologyid返回组织中链接到或标记有指定技术的任何 post。

```C#
[Route("/posts", "GET")]
public class QueryPosts : QueryDb<Post>
{
    // 由 AutoQuery 处理
    public int[] Ids { get; set; }
    public int? OrganizationId { get; set; }
    public int[] OrganizationIds { get; set; }
    public string[] Types { get; set; }

    // 由自定义实现处理
    public int[] AnyTechnologyIds { get; set; }
    public string[] Is { get; set; }
}

[CacheResponse(Duration = 600)]
public class PostPublicServices : PostServicesBase
{
    public IAutoQueryDb AutoQuery { get; set; }

    public object Any(QueryPosts request)
    {
        var q = AutoQuery.CreateQuery(request, Request.GetRequestParams(), base.Request); //Populated SqlExpression
        q.Where(x => x.Deleted == null);
        
        var states = request.Is ?? TypeConstants.EmptyStringArray;
        if (states.Contains("closed") || states.Contains("completed") || states.Contains("declined"))
            q.And(x => x.Status == "closed");
        else
            q.And(x => x.Hidden == null && (x.Status == null || x.Status != "closed"));

        if (states.Length > 0)
        {
            var labelSlugs = states.Where(x => x != "closed" && x != "open")
                .Map(x => x.GenerateSlug());
            if (labelSlugs.Count > 0)
                q.And($"ARRAY[{new SqlInValues(labelSlugs).ToSqlInString()}] && labels");
        }

        if (!request.AnyTechnologyIds.IsEmpty())
        {
            var techIds = request.AnyTechnologyIds.Join(",");
            var orgIds = request.AnyTechnologyIds.Map(id => GetOrganizationByTechnologyId(Db, id))
                .Where(x => x != null)
                .Select(x => x.Id)
                .Join(",");
            if (string.IsNullOrEmpty(orgIds))
                orgIds = "NULL";

            q.And($"(ARRAY[{techIds}] && technology_ids OR organization_id in ({orgIds}))");
        }

        return AutoQuery.Execute(request, q, base.Request);
    }
}
```

上面的实现还缓存了所有被定义在带有[CacheResponse]属性注释的服务中的querypost响应。

### IAutoQueryDb API

为了增加在自定义服务实现中使用自动查询功能的通用性，IAutoQueryDb支持并行同步和异步api，如果需要在同步方法中使用无法重构的异步api的自动查询功能:

```C#
public interface IAutoQueryDb : IAutoCrudDb
{
    // 解决此请求使用的数据库连接的通用API
    IDbConnection GetDb<From>(IRequest req = null);

    // 使用与源和输出目标相同的模型生成一个填充的、类型化的OrmLite SqlExpression
    SqlExpression<From> CreateQuery<From>(IQueryDb<From> dto, Dictionary<string, string> dynamicParams, 
        IRequest req = null, IDbConnection db = null);

    // Execute an OrmLite SqlExpression using the same model as the source and output target
    QueryResponse<From> Execute<From>(IQueryDb<From> model, SqlExpression<From> query, 
        IRequest req = null, IDbConnection db = null);

    // Async Execute an OrmLite SqlExpression using the same model as the source and output target
    Task<QueryResponse<From>> ExecuteAsync<From>(IQueryDb<From> model, SqlExpression<From> query, 
        IRequest req = null, IDbConnection db = null);

    // Generate a populated and Typed OrmLite SqlExpression using different models for source and output target
    SqlExpression<From> CreateQuery<From, Into>(IQueryDb<From,Into> dto, Dictionary<string,string> dynamicParams, IRequest req = null, IDbConnection db = null);

    // Execute an OrmLite SqlExpression using different models for source and output target
    QueryResponse<Into> Execute<From, Into>(IQueryDb<From, Into> model, SqlExpression<From> query,IRequest req = null, IDbConnection db = null);

    // Async Execute an OrmLite SqlExpression using different models for source and output target
    Task<QueryResponse<Into>> ExecuteAsync<From, Into>(IQueryDb<From, Into> model, SqlExpression<From> query, IRequest req = null, IDbConnection db = null);
}
```

在`IAutoQueryDb`继承`IAutoCrudDb`以下和API可以访问两个自动查询和CRUD功能。

同样，新的AutoQuery Crud API也具有同步和异步实现：

```C#
public interface IAutoCrudDb
{
    // Inserts new entry into Table
    object Create<Table>(ICreateDb<Table> dto, IRequest req);
    
    // Inserts new entry into Table Async
    Task<object> CreateAsync<Table>(ICreateDb<Table> dto, IRequest req);
    
    // Updates entry into Table
    object Update<Table>(IUpdateDb<Table> dto, IRequest req);
    
    // Updates entry into Table Async
    Task<object> UpdateAsync<Table>(IUpdateDb<Table> dto, IRequest req);
    
    // Partially Updates entry into Table (Uses OrmLite UpdateNonDefaults behavior)
    object Patch<Table>(IPatchDb<Table> dto, IRequest req);
    
    // Partially Updates entry into Table Async (Uses OrmLite UpdateNonDefaults behavior)
    Task<object> PatchAsync<Table>(IPatchDb<Table> dto, IRequest req);
    
    // Deletes entry from Table
    object Delete<Table>(IDeleteDb<Table> dto, IRequest req);
    
    // Deletes entry from Table Async
    Task<object> DeleteAsync<Table>(IDeleteDb<Table> dto, IRequest req);

    // Inserts or Updates entry into Table
    object Save<Table>(ISaveDb<Table> dto, IRequest req);

    // Inserts or Updates entry into Table Async
    Task<object> SaveAsync<Table>(ISaveDb<Table> dto, IRequest req);
}
```

由于其内部预定义的行为，AutoQuery CRUD定制服务实现对其实现具有有限的可定制性，但仍允许您应用定制逻辑，例如应用定制过滤器属性，包括其他验证，增强Response DTO等。

例如，此实现将`[ConnectionInfo]`行为应用于其所有服务，它们将对已注册的Reporting命名连接执行查询：

```
[ConnectionInfo(NamedConnection = "Reporting")]
public class MyReportingServices : Service
{
    public IAutoQueryDb AutoQuery { get; set; }

    public Task<object> Any(CreateReport request) => AutoQuery.CreateAsync(request, base.Request);
}
```

### 自动查询响应过滤器

聚合功能功能建立在自动查询的新`ResponseFilters`支持之上，该功能提供了新的可扩展性选项，使自定义和附加元数据可以附加到自动查询响应中。由于聚合功能支持本身就是一个响应过滤器，可以通过清除它们来禁用它们：

```C#
Plugins.Add(new AutoQueryFeature {
    ResponseFilters = new List<Action<QueryFilterContext>>()
})
```

响应过滤器在每个自动查询之后执行，并传递所执行查询的完整上下文，即：

```C#
class QueryFilterContext
{
    IDbConnection Db             // The ADO.NET DB Connection
    List<Command> Commands       // Tokenized list of commands
    IQuery Request               // The AutoQuery Request DTO
    ISqlExpression SqlExpression // The AutoQuery SqlExpression
    IQueryResponse Response      // The AutoQuery Response DTO
}
```

该`Commands`属性包含该属性的已解析命令列表`Include`，并标记为以下结构：

```C#
class Command 
{
    string Name
    List<string> Args
    string Suffix
}
```

这样，我们可以使用以下自定义响应过滤器将基本计算器功能添加到自动查询中：

```C#
Plugins.Add(new AutoQueryFeature {
    ResponseFilters = {
        ctx => {
            var supportedFns = new Dictionary<string, Func<int, int, int>>(StringComparer.OrdinalIgnoreCase)
            {
                {"ADD",      (a,b) => a + b },
                {"MULTIPLY", (a,b) => a * b },
                {"DIVIDE",   (a,b) => a / b },
                {"SUBTRACT", (a,b) => a - b },
            };
            var executedCmds = new List<Command>();
            foreach (var cmd in ctx.Commands)
            {
                Func<int, int, int> fn;
                if (!supportedFns.TryGetValue(cmd.Name, out fn)) continue;
                var label = !string.IsNullOrWhiteSpace(cmd.Suffix) ? cmd.Suffix.Trim() : cmd.ToString();
                ctx.Response.Meta[label] = fn(int.Parse(cmd.Args[0]), int.Parse(cmd.Args[1])).ToString();
                executedCmds.Add(cmd);
            }
            ctx.Commands.RemoveAll(executedCmds.Contains);
        }        
    }
})
```

现在，用户可以使用任何自动查询请求执行多个基本算术运算！

```C#
var response = client.Get(new QueryRockstars {
    Include = "ADD(6,2), Multiply(6,2) SixTimesTwo, Subtract(6,2), divide(6,2) TheDivide"
});

response.Meta["ADD(6,2)"]      //= 8
response.Meta["SixTimesTwo"]   //= 12
response.Meta["Subtract(6,2)"] //= 4
response.Meta["TheDivide"]     //= 3
```

### 无类型的SqlExpression

如果您需要内省或修改已执行的代码`ISqlExpression`，则将其作为访问很有用，`IUntypedSqlExpression`这样仍可以访问其非泛型API，而不必将其转换回其具体的泛型`SqlExpression<T>`类型，例如：

克隆SqlExpression允许您修改不会影响任何其他响应筛选器的副本。

### 自动查询属性映射

自动查询可以将`[DataMember]`请求DTO 上的属性别名映射到查询的表，例如：

```C#
public class QueryPerson : QueryDb<Person>
{
    [DataMember(Name = "first_name")]
    public string FirstName { get; set; }
}

public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

可以通过以下方式查询：

```
?first_name=Jimi
```

或通过设置全局`JsConfig.Init(Config { TextCase = TextCase.SnakeCase })`约定：

```C#
public class QueryPerson : QueryDb<Person>
{
    public string LastName { get; set; }
}
```

```
?last_name=Hendrix
```

## QueryFilters的可扩展性

我们已经使用Customizable **QueryDbFields**和**Implicit Conventions**涵盖了一些可扩展性选项，最可定制的是使用自定义选项覆盖默认实现，例如：

```C#
public class MyQueryServices : Service
{
    public IAutoQueryDb AutoQuery { get; set; }

    //使用自定义实现覆盖
    public object Any(FindMovies dto)
    {
        var q = AutoQuery.CreateQuery(dto, Request.GetRequestParams(), base.Request);
        return AutoQuery.Execute(dto, q, base.Request);
    }
}
```

通过注册类型化的查询过滤器，还有一个更轻量的选项，例如：

```C#
var autoQuery = new AutoQueryFeature()
  .RegisterQueryFilter<QueryRockstarsFilter, Rockstar>((q, dto, req) =>
      q.And(x => x.LastName.EndsWith("son"))
  )
  .RegisterQueryFilter<IFilterRockstars, Rockstar>((q, dto, req) =>
      q.And(x => x.LastName.EndsWith("son"))
  );

Plugins.Add(autoQuery);
```

注册这样的接口`IFilterRockstars`特别有用，因为它可以将自定义逻辑应用于共享同一接口的许多不同的Query Services。

## 配置拦截和自省每个查询

如前所述，`IQuery`除非已为该查询定义了自定义实现，否则自动查询的工作方式是为每个标记有请求的DTO自动生成ServiceStack服务。也可以更改所生成服务的基类，以便所有查询代替您执行自定义实现。

为此，请创建一个自定义服务，该服务通过您自己的实现继承`AutoQueryServiceBase`并覆盖这两种`Exec`方法，例如：

```
public abstract class MyAutoQueryServiceBase : AutoQueryServiceBase
{
    public override object Exec<From>(IQuery<From> dto)
    {
        var q = AutoQuery.CreateQuery(dto, Request.GetRequestParams(), base.Request);
        return AutoQuery.Execute(dto, q, base.Request);
    }

    public override object Exec<From, Into>(IQuery<From, Into> dto)
    {
        var q = AutoQuery.CreateQuery(dto, Request.GetRequestParams(), base.Request);
        return AutoQuery.Execute(dto, q, base.Request);
    }
}
```

然后告诉自动查询改为使用您的基类，例如：

```
Plugins.Add(new AutoQueryFeature { 
    AutoQueryServiceBaseType = typeof(MyAutoQueryServiceBase)
});
```

现在，它将使AutoQuery代替您执行实现。

## 从初始化中排除自动查询集合

“ [添加ServiceStack参考”中](https://docs.servicestack.net/add-servicestack-reference)支持的所有语言的默认配置 是 [InitializeCollections](https://docs.servicestack.net/csharp-add-servicestack-reference#initializecollections) ，它允许使用更好的客户端API，在该API中，客户端可以假定请求DTO的集合已初始化，从而允许他们使用速记集合初始化程序语法，例如：

```C#
var response = client.Get(new SearchQuestions { 
    Tags = { "redis", "ormlite" }
});
```

