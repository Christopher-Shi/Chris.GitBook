## 使用ServiceStack.OrmLite对数据库表结构进行创建（Create）、删除（Drop）、修改（Alter）操作以及对数据进行增加（Insert）、删除（Delete）、修改（Update）、查询（Query）操作
### 1. 数据实体（Data Models）准备
Player与Profile的关系为1:1，Player与GameItem的关系为1：M
#### Player 实体
```csharp
public class Player
{
    public int Id { get; set; } // 'Id' is PrimaryKey by convention

    [Required]
    public string FirstName { get; set; } // Creates NOT NULL Column

    [Alias("Surname")] // Maps to [Surname] RDBMS column
    public string LastName { get; set; }

    [Index(Unique = true),EmailAddress] // Creates Unique Index
    public string Email { get; set; }

    [Reference]
    public List<GameItem> GameItems { get; set; } // 1:M Reference Type saved separately

    [Reference]
    public Profile Profile { get; set; } // 1:1 Reference Type saved separately
    public int ProfileId { get; set; } // 1:1 Self Ref Id on Parent Table
}
```
#### Profile 实体
```csharp
[Alias("PlayerProfile")] // Maps to [PlayerProfile] RDBMS Table
public class Profile
{
    [AutoIncrement] // Auto Insert Id assigned by RDBMS
    public int Id { get; set; }

    [Index]
    public string Username { get; set; }
    public long HighScore { get; set; }
}
```
#### GameItem 实体
```csharp
public class GameItem
{
    [PrimaryKey] // Specify field to use as Primary Key
    [StringLength(50)] // Creates VARCHAR COLUMN
    public string Name { get; set; }

    public int PlayerId { get; set; } // Foreign Table Reference Id

    [StringLength(StringLengthAttribute.MaxText)] // Creates "TEXT" RDBMS Column 
    public string Description { get; set; }
}
```
### 2.创建（Create）数据库表结构
```csharp
const string connectionString = ""; //设置数据库连接字符串
var dbFactory = new OrmLiteConnectionFactory(connectionString, MySqlDialect.Provider);
using (var db = dbFactory.Open()){
    db.CreateTable<Player>();
    db.CreateTable<Profile>();
    db.CreateTable<GameItem>();
}
```
### 3. 删除（Drop）数据表结构
#### 注意删除的顺序，先删除有外键约束的表
```csharp
using (var db = dbFactory.Open()){
    db.DropTable<GameItem>();
    db.DropTable<Player>();
    db.DropTable<Profile>();
}
```
### 4.修改（Alter）数据表结构
#### 新增字段
```csharp
public class GameItem
{
    //.......
    
    public string AddedField { get; set; }
}

db.AddColumn<GameItem>(game => game.AddedField);
```
#### 修改字段
```csharp
public class GameItem
{
    //.......
    
    public string AddedField { get; set; }
    public string NewField { get; set; }
}

db.ChangeColumnName<GameItem>(game => game.NewField, "AddedField");
```
#### 删除字段
```csharp
db.DropColumn<Person>(p => p.NewField);
```
### 5.增加记录
```csharp
var player = new Player
{
    Id = 1,
    FirstName = "North",
    LastName = "West",
    Email = "north@west.com",
    GameItems = new List<GameItem>
    {
        new GameItem { Name = "WAND", Description = "Golden Wand of Odyssey"},
        new GameItem { Name = "STAFF", Description = "Staff of the Magi"},
    },
    Profile = new Profile
    {
        Username = "north",
        HighScore = 100
    }
};
    
db.Save(player, references: true); //将主从表的数据同时增加到数据表中
```
### 6.更新记录
#### 将Player中Id为1的FirstName值修改为North_modify
```csharp
db.UpdateOnly(() => new Player { FirstName = "North_modify" }, p => p.Id == 1);
```
### 7.删除记录
#### 将Player中Id为1的记录删除
```csharp
db.Delete<Player>(p => p.Id == 1);
```
### 8.查询记录
#### 全部的Player
```csharp
var players = db.Select<Player>()
```
#### Id为1的Player
```csharp
var player = db.Single<Player>(x => x.Id == 1)
```
#### 全部的Player信息以及Profile和GameItem信息
```csharp
var playerWithProfileAndGameItems = db.LoadSelect<Player>();
```
### 9.调用存储过程
```csharp
var dynamicParameters = new DynamicParameters();
dynamicParameters.Add("@num", 1);
dynamicParameters.Add("@sum", dbType: DbType.Int32, direction: ParameterDirection.Output);

var result = db.QueryMultiple("p_Text1", dynamicParameters,
    commandType: CommandType.StoredProcedure);

var pingSummaries = result.Read<PingSummary>().AsList();
var sum = dynamicParameters.Get<int>("@sum");

return new ExecProcedureResponse
{
    Sum = sum,
    PingSummaries = pingSummaries
};



public class ExecProcedureResponse
{
    public int Sum { get; set; }
    public List<PingSummary> PingSummaries { get; set; }
}
```















