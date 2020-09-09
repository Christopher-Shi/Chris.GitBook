# 属性设置
## 配置用户名大小写敏感

- ForceCaseInsensitiveUserNameSearch属性，默认强制不区分，修改为false

- 位置：Configure.AuthRepository.cs

```c#
services.AddSingleton<IAuthRepository>(c =>
    new OrmLiteAuthRepository<AppUserAuth, UserAuthDetails>(c.Resolve<IDbConnectionFactory>()) {
    UseDistinctRoleTables = true,
    ForceCaseInsensitiveUserNameSearch = false
    });            
```

## Configure属性设置

- Configure(IAppHost appHost)
- 位置：Configure.Auth.cs
```c#
IncludeAssignRoleServices = false,	//分配角色服务
GenerateNewSessionCookiesOnAuthentication = false,	//在验证时生成新会话cookie
AllowGetAuthenticateRequests = req => true,   //允许获取身份验证请求
SaveUserNamesInLowerCase = false,	//以小写形式保存用户名
IncludeDefaultLogin = false		//默认登录
```
## 文件大小写过滤
- IWebHost BuildWebHost(string[] args)
- 位置 Program.cs
```c#
options.Limits.MaxRequestBodySize = null; //设置不限制
options.Limits.MaxConcurrentConnections = null;
options.Limits.MaxConcurrentUpgradedConnections = null;
options.AllowSynchronousIO = false;
```
- ConfigureServices(IServiceCollection services)
- 位置 Startup.cs
```c#
//解决Multipart body length limit 134217728 exceeded
 services.Configure<FormOptions>(x =>
 {
     x.ValueLengthLimit = int.MaxValue;
     x.MultipartBodyLengthLimit = int.MaxValue; // In case of multipart
 });
```

# autoquery
## autoquery显示字段

```html
?Fields=Id,Name,Description,JoinTableId
```

```html
/query?Fields=Id,Name,Description,JoinTableId&jsconfig=ExcludeDefaultValues
```
## autoquery多库连接
```c#
var appSettings = new AppSettings();
var dbTypeSys = appSettings.Get<string>("SYS.DbType");
var dbConnectionSys = appSettings.Get<string>("SYS.DbConnection");

//注册主库
var dbFactory = new OrmLiteConnectionFactory(dbConnectionSys, GetConnectionProvider(dbTypeSys));
services.AddSingleton<IDbConnectionFactory>(dbFactory);

var appDataDbConnectionsList = appSettings.GetList("AppDataDbConnections");
//注册多库
foreach (var appDataDbConnection in appDataDbConnectionsList)
{
    var connectionName = appDataDbConnection.Replace(" ", "");
    var dbType = appSettings.Get<string>(connectionName + ".DbType");
    var dbConnection = appSettings.Get<string>(connectionName + ".DbConnection");
    dbFactory.RegisterConnection(connectionName, dbConnection, GetConnectionProvider(dbType));
}
```
specify the named connection on the **POCO Table** instead, e.g:

```c#
[NamedConnection("ISPEC")]
[Alias("dzml_eqim_p01")]
public class DzmlEqimP01Entity
{
    public DateTime RiQi { get; set; }// 
    public string lon { get; set; }// 
    public string lat { get; set; }// 
    public string mc { get; set; }// 
    public string depth { get; set; }// 
    public string DiMing { get; set; }// 
}
```
# AutoCrud
## AutoCrud Extra Options
```c#
Plugins.Add(new AutoQueryFeature {
    MaxLimit = 1000,
    GenerateCrudServices = new GenerateCrudServices {
        AutoRegister = true,
        GenerateOperationsFilter = ctx => {
            if (ctx.TableName == "applications")
            {
                ctx.DataModelName = "Application";
                ctx.RoutePathBase = "/apps";
                ctx.OperationNames = new Dictionary<string, string> {
                    [AutoCrudOperation.Create] = "CreateApp",
                    [AutoCrudOperation.Query] = "SearchApps",
                };
            }
        }
    }
});
```
> Application instead of Applications for the Data Model Name.
> /apps instead of /applications for the Route’s Path
> CreateApp instead of CreateApplications, SearchApps instead of QueryOperations
>
> Other operations retain the default Query + DataModelName, e.g. DeleteApplication.
