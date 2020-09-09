

## 使用AutoQuery技术对单表进行增删改查操作

---
**前言**:项目开发中存在大量对单表进行增删改查的操作,本文档对于使用**AutoQuery**技术编写增删改查的方法以**示例+描述**的形式整理成模板,任何人以这个模板为指导都可完成相应接口的开发。

### 1、数据准备

方案：因新增数据时Id字段的信息不需要前端传值,所以对于原始数据表通过Ormlite映射为类时拆分映射为两个类,采用继承的方式构造出包含所有字段信息的类,示例如下：

**原始数据表: UserInfo**
![UserInfo][image_ref_ujxjbxhe]

**Ormlite映射为如下的类：**
```text
    public class UserInfoBase
    {
        public string Name { set; get; }        //姓名
        public int Age { set; get; }            //年龄
    }
    
    public class UserInfo : UserInfoBase        // 采用 : UserInfoBase 的形式继承
    {
        [Index]                                 //索引
        [AutoIncrement]                         //自增
        public int Id { set; get; }             //序号
    }
```
### 2、创建服务编写接口
#### 2.1 查询数据
Autoquery实现的查询数据接口支持以所查表中的任何一个/多个字段查询数据,只需对相应字段传查询条件即可。
```text
[Api("Service Description")] 
[Tag("User")]                                       //同一业务所属接口的标识名
[Route("/User/UserInfo", "GET",                     //接口路由,接口方式
    Summary = "查询个人信息",                       //接口描述
    Notes = "查询个人信息")]
public class QueryUserInfo : QueryDb<UserInfo>      //定义方法类
{

}
```
#### 2.2 新增数据
Autoquery实现的新增数据接口,会新增一条包含所有字段信息的数据，传参字段保存传递值，未传参字段保存默认值。
```text
[Api("Service Description")]
[Tag("User")]
[Route("/User/UserInfo", "POST",
   Summary = "新增个人信息",
   Notes = "新增个人信息")]
public class AddUserInfo : UserInfoBase, ICreateDb<UserInfo>, IReturn<AddUserInfoResponse>
{

}

[Authenticate]                                      //权限认证
public class AddUserInfoResponse                    //定义返回的数据类
{
    public string Name { get; set; } 
    public ResponseStatus ResponseStatus { get; set; }
}
```
#### 2.3 更新数据(PUT)
PUT方式更新数据接口会更新该条数据所有字段信息,不需要改变信息的字段也需将原值传递回去,否则将更新为字段默认值。
```text 
[Api("Service Description")]
[Tag("User")]
[Route("/User/UserInfo", "PUT",
    Summary = "更新个人信息",
    Notes = "更新个人信息")]
public class UpdateUserInfo : UserInfo,
    IUpdateDb<UserInfo>, IReturn<UpdateUserInfoResponse>
{ 

}

public class UpdateUserInfoResponse
{
    public int Id { get; set; }
    public ResponseStatus ResponseStatus { get; set; }
}
```
#### 2.4 更新数据(PATCH)
PATCH方式只更新接口显式提供的字段数据，其余字段不变，一般用于带有权限认证，只能操作部分功能的时候，避免误更改其他字段。
```text 
[Api("Service Description")]
[Tag("User")]
[Route("/User/UserInfo", "PATCH",
    Summary = "更新个人信息",
    Notes = "更新个人信息")]
public class PatchUserInfo : UserInfo,
    IPatchDb<UserInfo>, IReturn<PatchUserInfoResponse>
{ 
    public string Name;     //接口只提供name字段信息的修改
}

public class PatchUserInfoResponse  
{
    public int Id { get; set; }
    public ResponseStatus ResponseStatus { get; set; }
}
```
#### 2.5 删除数据
Autoquery方式实现的删除数据接口,传参匹配条件删除匹配到的所有数据。
```text 
[Api("Service Description")]
[Tag("User")]
[Route("/User/UserInfo", "DELETE",
    Summary = "删除个人信息",
    Notes = "删除个人信息")]
public class DeleteUserInfo : IDeleteDb<UserInfo>, IReturnVoid
{
    public int Id { get; set; }                     //删除接口前端传参字段
}
```
**注:**
编写增、删、改、查接口时只需要将示例中对应关键字(UserInfo)整体替换为新的数据表对应类的名字,同时修改Tag、Route、Summary、Notes等描述信息即可。

### 3、基于查询接口的更多数据获取操作

#### 例1：表格分页显示数据

表格分页显示时需要得到数据总数确定查询次数,按照当前页码跳过部分数据查询一定数量的数据展示,通过组合设置skip和take字段值可以满足这种需求,返回的total字段值代表当前匹配到的数据总数,示例如下:
```
调用接口：
    let request = new QueryUserInfo();
    request.skip = 0;       //跳过0条数据
    request.take = 50;      //返回50条数据
    let response = await client.get(request);
返回值示例:
    {
      "offset": 0,
      "total": 100,      //数据总数
      "results": [
        {
          "id": 0,
          "name": "a",
          "age": 16,
        },
        {
          "id": 1,
          "name": "b",
          "age": 20,
        }
            .
            .
            .
      ]
     }
```
#### 例2：获取某字段信息最大值\最小值\排名等

需要在筛选数据后根据某个字段进行升序/降序排序后再返回,只需在调用接口时给OrderBy/OrderByDesc字段传需要排序的字段名即可,可传单个字段或者多个字段,传多个字段时中间用','隔开。

结合skip和take获取升序排序后第一条数据示例如下:
```
调用接口:
    let request = new QueryUserInfo();
    request.OrderBy = "age,name";
    request.skip = 0;                     
    request.take = 1;
    let response = await client.get(request);
```
结合skip和take获取匹配到的最后一条数据示例如下：
```
调用接口:
    let request = new QueryUserInfo();
    request.OrderByDesc = "id";   //数据根据Id字段降序排序
    request.skip = 0;                     
    request.take = 1;
    let response = await client.get(request);
```
#### 例3：数据聚合操作

可通过对include字段传值实现数据量统计、最大值、最小值、平均值、求和等数据操作,按照"min(age) MinAge"的形式自定义返回的字段名"MinAge"。

| 数据操作   | 字段传参            | SQL语句                                    | 描述                           |
|--------|-----------------|------------------------------------------|------------------------------|
| 数据量统计  | count() Count   | select count(*) as Count from  UserInfo  | 括号中不传参默认*,可传参其他字段，返回匹配到的数据条数 |
| 单字段最小值 | min(age) MinAge | select min(age) as MinAge from UserInfo  | 括号中必须且只能传一个字段名，返回单字段最小值数据    |
| 单字段最大值 | max(age) MaxAge | select max(age) as MaxAge from UserInfo  | 括号中必须且只能传一个字段名，返回单字段最大值数据    |
| 单字段平均值 | avg(age) AvgAge | select avg(age) as AvgAge from UserInfo  | 括号中必须且只能传一个字段名，返回单字段平均值数据    |
| 单字段求和  | sum(age) SumAge | select sum(age) as SumAge from UserInfo  | 括号中必须且只能传一个字段名，返回单字段数据总和     |
```
调用接口:
    let request = new QueryUserInfo();
    request.include = "count() count,min(age) minAge,max(age) maxAge,avg(age) avgAge,sum(age) sumAge";
    let response = await client.get(request);
返回值示例:
    {
      "offset": 0,
      "total": 100,
      "results": [
        {
          "id": 1,
          "name": "a",
          "age": 10,
        }
            .
            .
            .
      ],
      "meta": {
        "count": "100",
        "minAge": "1",
        "maxAge": "35",
        "avgAge": "21.5",
        "sumAge": "5750",
      },
    }
```
注：
* 若想对多个字段求最大值,需要在include中传多个max(参数)分别统计。
* include字段的操作是在筛选后的数据基础上再进行处理，返回的字段在meta中展示。

#### 例4：数值型字段常用的大于、大于等于、小于、小于等于、范围匹配等操作

传参格式:字段名+筛选条件 = 条件参数     eg:request.AgeGreaterThan=20   //年龄字段大于20

| 数据操作              | 字段传参                     | SQL语句                                    | 描述                                    |
|-------------------|--------------------------|------------------------------------------|---------------------------------------|
| 获取字段A大于20的数据      | AGreaterThan=20          | select * from UserInfo where A>20        | 按照格式传参，内部转为 "A > 20" 的形式              |
| 获取字段A大于等于20的数据    | AGreaterThanOrEqualTo=20 | select * from UserInfo where A>=20       | 按照格式传参，内部转为 "A >= 20" 的形式             |
| 获取字段A小于20的数据      | ALessThan=20             | select * from UserInfo where A<20        | 按照格式传参，内部转为 "A < 20" 的形式              |
| 获取字段A小于等于20的数据    | ALessThanOrEqualTo=20    | select * from UserInfo where A<=20       | 按照格式传参，内部转为 "A < = 20" 的形式             |
| 获取字段A介于10和20之间的数据 | ABetween="10,20"         | select * from UserInfo where A between 10 and 20 | 按照格式传参，内部转为 "A BETWEEN 10 AND 20" 的形式 |

#### 例5：字符串型字段常用的首字母匹配、尾字母匹配、包含匹配、多值匹配等操作

传参格式:字段名+筛选条件 = 条件参数     eg:request.NameStartsWith="Chen"  //name字段以"Chen"开头

| 数据操作                  | 字段传参            | SQL语句                                    | 描述                                 |
|-----------------------|-----------------|------------------------------------------|------------------------------------|
| 获取A字段以"a"开头的数据        | AStartsWith="a" | select * from UserInfo  where  A like 'a%' | 按照格式传参，内部转为"A LIKE a%"的形式          |
| 获取A字段以"a"结尾的数据        | AEndsWith="a"   | select * from UserInfo  where  A like '%a' | 按照格式传参，内部转为"A LIKE %a"的形式          |
| 获取A字段中包含"a"的数据        | AContains="a"   | select * from UserInfo  where  A like '%a%' | 按照格式传参，内部转为"A LIKE %a%"的形式         |
| 获取A字段等于"a"或"b"或"c"的数据 | AIn="a,b,c"     | select * from UserInfo  where  A In ('a','b','c') | 按照格式传参，内部转为"A IN ('a','b','c')"的形式 |

#### 例6：查询特定字段的数据

可以通过fields字段中传参需要返回的字段实现只返回特定字段的信息,传参可以是单值也可以是多个值,多个值时中间用','隔开,示例如下:
```
调用接口:
    let request = new QueryUserInfo();
    request.fields = "age,name";
    let response = await client.get(request);
返回值示例:
    {
      "offset": 0,
      "total": 100,
      "results": [
        {
          "age": 16,
          "name": "a"
        },
        {
          "age": 20,
          "name": "b"
        }
            .
            .
            .
      ],
      "meta":null
    }
```

**更多使用方法详见**  [Autoquery](https://docs.servicestack.net/autoquery-rdbms)

[image_ref_ujxjbxhe]: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABOgAAAC3CAIAAAAnyvaPAAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAgAElEQVR4nO3dfZAb530n+F+ToihKtgjFkmU7l7M9BEgFGb8UwmCrMJeqJGuAAaZ2b5gcJlUbbQ2SkgCinCP6cp4q4WqqXK7iHlI3Ll+DiQsEpKzBLd3W7cDxTO3tACFgJ1XRDa5uzGCz8hjiDHogOXu2xVCympJfRFJk3x/93mi8zQCDBub7KRU1aDQaz9Nv6F//nudpRhAE0nnkkUfocHvllVd+4zd+Y9SlABiW7373u1/4whdGXQoAAAAAgD4cGXUBAAAAAAAAADp5yPT6lVdeGUk5AODAfPvb3x51EQAAAAAA+sCYmgoDAAAAAAAA2AqaCgMAAAAAAICtIXAFAAAAAAAAW0PgCgAAAAAAALaGwBUAAAAAAABsDYErAAAAAAAA2BoCVwAAAAAAALC1iQ1cb926NeoiAEyIyT6aJrt2AAAAAJNhYgNXAAAAAAAAmAwIXAEAAAAAAMDWELgCAAAAAACArSFwBQAAAAAAAFt7qPPbr776atdFfOxjH/voRz86oPIcqG9/+9v6l1/4whdGVZKRw6pQYVUAAAAAANhNl8BV9Vd/9VetE3//93+fiN58800iGtPYFQCG56eNyi+28/SzLSKix6ZPPPNHH3LiRgAAAAAA9G0vTYU5juM4Tj/lzTffvHnzZpvZm5mAg61QhXVYCmSaLR+psJaTB+0LRn1+upkJqIVsVzmHg61on6iwhpf9fFNPH5RKwVbM5elhXe5vVVDHTaYVxlSJZibQ73a2/6rQfZ9hr9d9s7of6Mume7+ZCbSsq1YHsCr26SfVr975L3/y2PH//PjHPnj8Yx88dvw/3/mHL/6k+tVePtvlFNC28r2ulZHqbcfXNpzy6qAq1v501u38NqICAwAAwGEwsD6uN2/etI5dm9dWNyOzfiIib6omGBUiVsuqrOfJ7ZwaVNGG5dx5d9IjXZv5OaVGtZTXUE3Ov/8vmopfSdXD7a4CtavMMBW0b4wUOq3iIWq56l2fbbMypuLlgjvp6efqdixWhTdVE2opr/wqUlC/2SIaixSknUbb4SusJ0mRSD3cJbSx+ar4Kf9t8UcvPvbkQ0eOMXd/9uAXwv3798RHnzgq/ujFnza6b3E/V3AnL7RbA7qTilLRAMsGHA6HJ7lJ+fBBRObvv//+Sy+99NJLL73//vsDXGwzE5A33PKZmrLhmplAOE9KxYZ+40E7nWlnNYuzt3RI26HAAAAAcBj02lS4Fzdv3nz66adNE5vXVjcji36izteqzUzAk9zUTwk78tazRgoDiQaJ6LnizyynvxR6rIdPT03FOcFJjuXMRX9cibIrl5N0vtZzzF1hHeG87rU3VSvHqWVdSDZNq8SbqpWlL9b+kjT5OtFsr4Ug2t+q0CrhcSSl0EjOxzczAc/2Yqet5edqqYAnkDGWX9W6W0jsuir2pnI5SamaEoaF89IePss6PAFSazOKVbEfv7jxjcdOHj1ylLnz3n3xY//j49MLP926+sGbf37i5NGfbec/5Op6CPsvppYvXGvGLXaMyuXkpnSG8KZqZUGYZR3LZy5ycY6TdzmBI9axPJR6SaSo9Y033iCil1566bnnnnvkkUcGsuSpeFmIE1W08jczAU/SXRDK6iprZgKe1fPn+r+196UvfYmIvvrVnpLePRpqgQEAAABUewlcWZbteV7dNXlH8tUPKdc4bQIZ+1AukDnB38wEHIaQwuNIai8iBYHza7FdXgkzpBhVH17oLv26R+f6mWkzqX1lpCBwtL3pPXNKen3qjLf10wPl5wSBq7CO5TO1cnyqwjocYf3bhsCqtV5T5857k8nLlXi7+o7TqtiTynree742JQWt3lRNkHYHPyfUMgGPY1WLP8dpVYg///6RDzFEdP/ug5Ofe545euxD0wvv/tfLDz92RHzn+1YlN9zCURiOJakepzLLef2aqLBhKgjOyw51R1N2OX3gP0D6qJWI3njjjcHGrgYV1rN6viY4Lzsc6wWB86v3Nux6ghy7AgMAAMCY6DVwlcZh6ltlPU/ugnLBor+SVnlTxk9cTlKqdjDXOPvKoZ07H0mGHfVUrRyXg2415N5Vgjh1Zj8nCBypwZ0xwDCbipcFan8pL8e6nOA3TlC/rZkJSJGQtDSnezMsp0LbRz2DSydKVVVL4tle7BxsNa+tbhJtGvLWqvFeFb2prOcji8Iu6wjXIxFv3nSMeFOpbY8jkKqVR7AqBuLow0fe+y8vfugzkZ9uXT16jCEiEsWWufS7jaTCOsJkVbrKNnnrYYeDiChSKFCYCoKfyC8IHBkyrmeGGrV+6lOfkmJX6Y9+Y9fWk6G0vfQb2ONIqluREwqsw+GQ5htJELh6wZGkVMs6tW+BAQAAYLIM9TmuzcxyXv9a7iVVS3mVvnbmrnYVNpyPLEppO0sH0k/q3oM7f33zL//NjT/40vd++9/c+IO/vvmX9x7cMc0zNRXnhFqKkpfVsUg8SUpdiU/J/fNau242+bruVddOvC0dzeTVFSl0vByvsJ6kWz+Lspz9xSdf/vKXu890Te3DWWEdjkBmt/tHmtdWN72pgm5FWrDHqrh399h/+uYjX2ZPXPzXj3yZPfafvkn37poqkwk4wnnaTHqUvpbhPOXDDnliYLlusVj/bCQfZokThDLHldXayQdLOR7nBEGrgz1WRQ8eemKaRCKiY48eYd7883eLHubNPz/26JH7d0Xmsen9LNnPlctX5HMIR+vSGnawmYy+j2ub6H7/Xn75ZSlqfe6556Qpzz333Cc/+ck33njj5Zdf7n05rV1Gpa0jbZhCRJlDmirvWZGCoN8ZevKDH/ygtSOuFH7/4z/+Y+/L2XQv1lKU9JjPwQMvMAAAAIClfT0OR2WZj21mLiTdkYjShtavTwaR1URpQI9IwU9EpvyL3MGvy9V5vyyf2Hn3wftXmv/T6z//njTxrbs/vHbzGzvvXb8w9b8/fMSUTpETgmojz3KcMgGH1IK4wDqkRJmhwHsec0rpN2ZKS0l02RtvKhWhpLmDsLm7Y4vODy/tGrU2M8t52qTV87XyuWtSw+lIoXaOrq0S8ZlAWG5J3Zreq1xObkYKZb/fGXFYJ12tvmwEq+LuneOX/9cjzR1pInPr5kPFvzpy43t3Lv4v9PBxZd6peFlwSqm+c9cCnu1FYXbdsT6r5v+cl60S7X6u0HvljYa8Kvbl3e8yTx4joiNHmeMfPipNe3Bf/MXt+yc+FxnYt/g5QeCamYBn2xkvC/Hh93H1+/1E9Oyzz6rJ1UceeeT5559/+eWXpbcGQL7jl/Q4VlMpdzKZJ2+qVoh4wvn1Ctfvl1QqlRs3bkgJYWmKvqmzOrEr75lTU/GycC4T8Hgc28YjeaAFBgAAALA0xIzr7ra7wEljwRgTqLqBP3V51ArrSZLXouddhXU4PNuLQ00P6Xzn1stq1Ppvf1P+4/Wff+9vbv1786zSUy0yrCNcT9XMeQU/JwiL2x7dIyF2ty3G1ulNM3Mh6W7fqFOXvSnH45yWr1MS3PsZo7lr1FphHRfofIS8Ur6ZpLFrOf/U1CkiIme8rBSjtVrLeW/qop+I/BdT1H4UWcNnRrEqjpX/oxq1qo40d45V/q89LM3s1Bnv5vZl3UEip227NDMY5V7R2c2XnSccR5kjJD6gBx+I4gNRfCB+8P6Dn731AfOJ5y1HZtIGp9XWgek0Ia+ICqs7hyjrxnuGZ/Unl6FlXD/5yU+2Ngl+5JFHpLzrIL5BvuNH3lSt4E4mqSAIwuK2J5wn0wrpbTjuZ599Vm3MLE1Rmzo/++yzfZduKl4Waql6WPf1Ay4wAAAAgKX+BmcyPb6VOg7U5Oc4osq69KeaE6qwgXWifJ0i569wWqhXYR31VO0KXfBsW3bjy2vZosHliSyf0ll7x/rq6vo7f/27T/+xYdLu9qb3/JV4PC4NqumhgnCRiGiddYQplaonV8/XBEEpapOvk/f8qa6latuLsXWc5TYdFP0XU8seebgjqY2oRbLbqN0DS/VRq/7vr3zlK9rXcYKfKqyuw568tSKpFBkzroZlNzMXkpuRQllaQVPxK6lVj3GQJvusiqPf3bCc+ej/+8q92f+h7bL6Gch3lhPUw6vS0kNzJKtib26+7Hz84w8fffgIEX1w58EHD5+9++Y/EBE99pkTn4u0G09YG55N1raPq58ThItyYvVUJnAhc25xe9M9W+aEONdTt2pbk+9GzK7n61JVSRr8zXjaq7COMM32VEkpqDaNJiU1dd7raFJyy4IwOytw/oEXGAAAAMBSf4FrP+MJW6qwYVqsnVnO0+Ls9oVMU72ykdoMNzPy3/qmj60X8YNi2Sj0nXv/RLpcq/THH7/yGeHeLdPHm3yd3LNSsSrreW+q5ifiiWh2NpIPbzuFgtvhYZ3yRXTz2uqm9/yVKeWTbeOZ1mFqehnlSAtsIgWBi19JBTyBTMqdTNZTtXL360XLVdFTv1YrSujUzASSJDXhlKqgm6eZuZCkVE1XpSm51DX9XmGTVcG88zYRPfx/FPVv3f3DECP8xFQ83U5BRETeM6eI2nf2lWN8aecx1a2ZCVygK1rXxwNfFXugRq337z648979n771wX/z3L8bxhdJps6dJ48nTJHCLNsyqvDwB54agt1td4Hzy3f8dI+WIdahHBoVqY1H7xtQil1ffPHFH/zgB0S0v6hV5ucEYWgFBgAAAGg1xIxrCzkC9VNmmYj83OK6g63Y7tLyiWMffevuD//4lc8Q0b/9ze9JfxCR49iTxhnVJ9TKTV7P16aIpHaL/tkILfNNjitEHOHAmVo5PqWPWwdIiUw868aOjlL+Mpn37meE5q985St7jl07U65uTWWTSq0F+30Z6qoQn/gIc+vm3T8Mmaef/CXjhF3dM2eIdrc33bNTauAqd+nWp/S14EoZlFofyZcXLXtKdzPUVdHBD//y1C996riUa73z3n3H3LZjCN+iaWYyu/ErqVXP9qzfbxxV+EDOKoN9IKrEbzzHaolorsYH5E7LEf0zUnujdsQlYwfd/RtSgQEAAABMDirjqo9UlA57fk4gVnnY3yhYNgr1OL5Q/qerrdPPPvG7htdSJHqKjE1e5bjFPxsJL19rxuNyWkJOLyrRgj66MTwWw/RsoO78nFAgx/IZfeZSjlu8qZqweNnhcSR7aFzdrqmwPnbVNw/uLK9vv6r7O6IVr002bCperlHAs4dgbbir4v5Z30OlVYvp/+w3Da+VZ7JSk4jqy8ubkcWWcZMs8+3S0y9byiY9zPVC5lxfa2NQq6Jfxz98VM21vvvju8ONWikf9kQKtcyF8CZR2NFuj5oQ8vB0+8kiS3nXwRarvQEUGAAAAECvv8B1j+RcksWz/KTrcgdLVo9rVJs5elO1YeSHrBuF/vOP/uvGT2vS+ExquvXTj37md576Q8OHd7c3yb04RUqOoaI0VYwUpoimDGMoKw+nVS7miCIFpe+rFj9YPNxV17NRGm25G2V8Y6VrLSc/P9bh6HwF2WFU4a55V+0SVdlKVl8lNxWWWw12eK6j3IHOPHLpaFfFvXP//ZHt75vGZ3owdfpe4F8YvlN5JqsnTxSJROrnL57KKKMsT2l18KbMfZ2VZKw2h/rI1Xi5bFEz6c/hroq92UOutd8evNJDlGpl52VH2C1l8yryQ0MNnxruuMm9sq7dpsVDrVs3h/JZ3dazt7ErMAAAAIwLRpB7Kll79dVX+1rcZz/72f2VZ2Bu3br11FNPdZ6nXbR278Gd79x6+e/fKb9z75+eOPbRX38i8M+fevbYkeNWy5gQnR+HQ0Rf/vKXe8+4jrW2q+Le3WPX/uPR7/7fzDtvi0985P5v/Hf3zv1LOvbwCIp44Ho5mojoHzOffvQjx37+9r3/Nv76AZRqUHqsHQAAAACMUJfAdXzhYhRgUCb7aJrs2gEAAABMhiE+xxUAAAAAAABg/xC4AgAAAAAAgK0hcAUAAAAAAABbQ+AKAAAAAAAAtjaxgzMBAAAAAADAZEDGFQAAAAAAAGztIdPrmzdvjqQce/P000+PuggAAAAAAAAwXObAlYhOnz598OXYg52dnVEXAQAAAAAAAIbOInAdI+OVHwaAYUDLCwAAAICJN96Bq20vWG/evGnbsknsX0KV/YuKEo4Q7l4BAAAAHAYYnAkAAAAAAABsDYErAAAAAAAA2BoCVwAAAAAAALC1vQeufHqGiZW6z1eKMQzDxErKX4qZNL/n7wYAAAAAAIBDY++BqzNxldsKtYtdtSA1REVRFLNBIiKKFkVRFEWxGN3z9wIAAAAAAMCh0nPgyqdnGBMXWyXKhYwTtUSqj2uIohazEr+zNYQKdFRhHWzF8p1mJuBoFcg09Z/VvxxwsUyLtv6yZiZgXf62bwyqeJZLt/pWfcF1f1vVx7zKA5mm5bReNTMBy+3Vw4arsBYbv+8CdGVZkm7Fk8oWyFQs99AhbvahaFnTbMa6Xtr+INVQt7MNd28HAAAAgLHQV8ZVyZe2ZUikVlmXGs7GSkSNetXndknvudy+AVZC1hqMhvOUD1uGJlPxsmBSiAy+SJYq6/nIYnyq63zNa6ubkVk/kXb9H2DZgMPh8CQ3lYoNPLiusOE8KUvvFi9U1vPe8+emlL/J7ZT+9nMFd9LTWrhIoXVlW03rxVR80Z28oEbNupJY1MkYPBGn3+hqAQRBKPewXXrVPDWboqTHtBKlVWNeM9q+G6aCIAhXzp2KX0l5DWWrpbwk7w8DpV87loF2y9FjLHanncTPaeUvRIgis3H1yCtEyJuqdV31lcva3t6+mAAAAAAw2XoOXJ2JDTEbNPVTNadag1lxI+GUP6FkXKWsK5++lPPNz8pvOk9PS3FtL71ke9UajBYihiv/gYcme1JZN4TTba/BtQv2QOYUJ13nn7/IlZUQpjCEWLuZCYTrqZr6FZyftKBKFy5LwUozs6xFi02+rg+r/JxQS1HysjGoUesdznec1hv/ReUbmpnlPG0mPfJi5L/UkkrhkxoocWopK+t5b+ri4GNBIiKamvLHy0Ihkl82pk/DeTKUUIn8IgWhlvISEVVYj+dyZSp+JVUPa0lsT9Jd4AZc1mYmIIfKksXtC9r+2MwEHA55f5APKMvbEZ1JzR4qrCNcT9V05e94q0FXwOV8yzE8+oMYAAAAAA7YQ33OH8yKYtY4qRRjQrloUYtYLZRiLna6KGqzWCxnvyqsZeiTd5gnRgoC57ec35sabJEsNDPL9VRNkC+8K6xjue18ebmg8pxhKgjOyw61OmH5L0+AagO6kK9cTm4SbXocSe0rIoXaGWmdncoEPNuL8v+JiHa3N2lTm1tXJjJOUSui/qGruNW0Hk3FFyOO9QpH68lNw2LOdFkjhm2vVcCbGtSa1Pg5wU9EcSGu+/LWAvoXI44weyZFROtsmOTKxMs1Csjl0+8Ng1K5nNz0pq5oi/VzZeVFM3NBt1bVyhTIEfawzt7L4ucK6w5HWFu3+pWvrnvrVd/MXEhSqkasw5EfytYBAAAAgDGxv8fh8OkZhlmb03Vk1dE1FZ7ZcUdbesMOYVxhfctDcyNQrbmlpMnXje8fSFPh5rVV0tJMTb6utq6l1Qv6Rpe72+Stq+nNChumAudXWl7qMq4DvZr3c/KSdf/rOLthverXvhHnJykhrkY7fk64Qhccl51lgfNXWIcjkGn6uf4zaX5OkFdLv0FdS2kHvf377Jrp5wSBO0dENMuplWlmLiQ35Rnyy8NpH7u5vWs1WYppW7PR/ospL+XX++lz6udqKe/m6jWt+KYDs92q56+tUupKfEra0a7QhZ7arwMAAADABNpP4MqnF9jpolXMSkSGpsIbiUS2GNVPaXA+mj7dIUe7H81MwOFYn5WDmWYmYNEgt3ltddN75pRuyl7ipn5VLifduu6tu9ubauvaTbc7kg+rBfVz5bLSyZEjpXWxPLSN0mi378a1w9K8trrZud1n63hIWstjY9vZ3uIStddvpmnq22xqKtxlgUMcg+vcef0G7UyujsfUnfMCXdHCu8Vtz6DjNv9shCgftlhmk6+T9RadOnfeS3W+p3Wmb2UubRTTeFqmL9ZubkzFywIXj5d1h6TSFWDgeWcAAAAAsL9eA1ervq0uttqaRm3bbTW4yBG7XJKX5mKni+0i3j2SA89mJuBJbuq6TnpWz9fUkFS9Mt7d3tSSnQekwobrqYt+bfzb9bzWK9R75iIndyJsCSP8nJIsdsaH2seViIjyYSWA8qjpvs6a11Y3TdGiOWrUj9IjldubqkkZXVP6rbe4xM+pmToloKmpgxkZh/0ZUaAzNRW37OdrSVo9tZTXYqwoJXUrr8KBVsfPCbWUVz5Y9Pvd7nbHTd8mTWumbBp5gxQi3Y856/G+e7sLAQAAAAATq9fANZhtHUO4wfmsBhrWxaNauBsryQ9+nUmnY0xoi2sMOGxV6cMYKQgox6ml1WaTr1PriMPDvSyurNdTV+JTzVMXC+6kx+FwhPMtg8T6OaGWqofZSoXV5d+UNJX3DM/qE3PDybhaNRXOh7VvbQlnK5eTm95IxGuKQGspLylJbWPCdX1WCsqkbTW7PrBtYMyh24B6o0S3AkwpYS0H2by2uklE65kDHTNX2giFiLSRldKcOtOxlfieVnRlPa//XJOvt5mxtYm/dkgDAAAAwKG0vz6ubZRiTChHVda1NmeIZp2JqxyxbM7HXe00ktN+STHCBbqixkQtEWCbx+EMN+7xc+X4FNHU1JSfk5NQVgPaSrGOn9NSiLXzqxcyzd3tTbczrmbm5Iv7IbRutsq4RgqGb9Wn/ZqZ5TxFFjnuypllXUPQCutJUupKXHk+jj7sULeFNG5tXl2WN1Xbc0axcjm52ZLP69bR1NDLeAjUZ+LqMs7tHgMjDY1F+Xwy6WErWubR0H54WO2alaT+ZtLDVohoyukmQ7dUVfPa6h4bK7Q2xu96wLV7pjAAAAAAHDJDCVyDWVHq0arPqpZiDMO4VuYbYnGadQ1jaCYi6UJXCoQ2kx6HI5w3Zm+u0AV7PAKywobralTXzdS585T0hPORWWJbQpnBX9abM66GUZXMnULlsWc5P9FUvLy4LT2htMI6wvlIoSWqHuRzXPWamYAjnI+0Pi1md9vQk1npZ6nseq09nQersp7v9dGrzcxyPpJS1nt9fVe9taK/YTDcbthT8cWI8rd/NkKbFs2c2w3a1DfLGw0mTb5uEdy29o0FAAAAgEnX7+NwiJSEKhERRYu9NPgtxZhQzsc1RFHKs2ZFMSsFshRtP7jTnvg5oXYm4Fk9X1P6u24TSf1eKVUrx+PlcptPNvk6uWcPpNNrhXWEqSD0GII0M5nd+JXUqmd71u/3CwIn1Wd7cRidN5VAr/1NBf/F1LLnciUuD3x1IbkZKZR1gwWfygQcjk2v9sif4WnydaIzSnJX94X+2Ug4LD9pxZuqTSkPYNE93CUeJ6qwjiRFIqseR5IiBUF6bs0gVdbzFCn0stRm5kLSXRDO8YEkEU3Fy7OswxHWzaA9aGiQ40ibnswjJc/lAvu5WqruCTvI8FAmRzjvTdX29f1+TvA3MwFP3uv1hh0O06qfipcF9ds80uOZ5KfvyG81M3Wi2X0UAAAAAADGz14yrrr+rm2DzmBW1B7sGsyKomh+zqu0lMH2dK2wDmkwJvlSXAnDpuJloXZ+tdOgrMPOvemLGKbeH8mZD3u2nacyF+Qc65AzTZX1fOfBgZVmxMtSXtWTpFRNqoo6gOzq+Zo0xpRFs1a1xauuebDVtO6amYDDs3q+cH7VE+AvmjORuoa55fiU+lJb6VIb5XqqVua4siAIBQoPvBluM7Oc7zU3ubvtNqaLTY8a0rUbGGTO1c8pYxVr244zxJC1VF3XEzycp9axho1dxbvvoVL3bXdBKJfLgvwFph7o0r4Uzls+ksmURQcAAACAw2AvGVd7klunCoJfuvCVumd6UzX5KnsqXhbi1MwEPA62oA6YY+xfOeQcYTMT8CTdUhF7mf3a6qY3VSs7LzvC7oJQ9suRufy2moMbXAqusl5PXeGmiCi+GHGEHXmiSOGUsjK9qZogTBFJK+4yu52vp2rlc9ekt7V3iYg4QeCk+GOTdJlO9Y8K61iWZ7Wa1t3uNknVjgvnlK9pI2K8TSBt9YhxM/g5QbiYCXgCNLB0pvSIoCtTpA50rSfngyXeVK3MHewI1yo/Jwhc+7elw0bVzAQ8SW0tmd5tpVZdOhCll4Z1PxUvC/EK63CwhdqZZY9pTyOiU+e9SS3fTNLbo1pbAAAAADAijCAI+tc3b948ffr0qErTl52dnaeffnrUpbB28+ZN25ZNYv8Squxf1ENVQot21yNl/5UPAAAAAPs3ORlXADgAXVK0AAAAAABDMJRRhQEAAAAAAAAGBYErAAAAAAAA2BoCVwAAAAAAALC18R6cadRFAAAAAAAAgKEzB64nT54cVVEAAAAAAAAAWqGpMAAAAAAAANgaAlcAAAAAAACwNQSuAAAAAAAAYGsIXAEAAAAAAMDWELgCAAAAAACAre0xcN3d3R1sOQBgIHBsQo8me1dB7cYXaje+Jrt2AHDwTGcVZFwBAAAAAADA1hC4AgAAAAAAgK0hcAUAAAAAAABbQ+AKAAAAAAAAtobAFQAAAAAAAGxtsIErn55hmFhpoMsEgD3j0zO6I7IUY2bSvP5Nw3tWYiXtU6aPw6Rpt4FLsS7ndT49w4zVvtGhRqWYVBH10OHTM73v/8YD7gDw6RnLA9f6WO7hI4PdjKWYtkT93/taYqeFmDeA4fWBb53u2hbJhmXtl+Wm6nYUST9EM+mS9V465qsEAPZtgIErn55xsVWiXAgnGwCbOD3ty4UsjkA+PcO42Gm3S7mECGZFscH5KFoURVEUi1Hpr2zwoEsMdlKKMQwToqJxRzDf5XCxVaIq6xpmCHRQgnPT7LJ2uPDpBXZ6KeEkKq3loksJZ/clWPwEDm9VOBMbYosG5yPlSNboN2HLm7oP2l7LnmohsmUAAB4FSURBVNb2fstMmhJXua2Qba9A+PWVanQuSKQdVDOx2Ix8TMl70ngeSMS75jhiXaYfn2C2OM26zFXSbqaEqCiK4tVZV+Kq9nOk7pzyugKAw2tQgWspxrjqS21/QHH1CzAKzmB2Q2xwvtya4eKBTy+wxDXEbMLZw4U4HELyleTanDnkkfm4hmXoo7PRS5RnP8FskdTDpbS8Mt/IBomotJYzhqRtgyGLoPBgV0Vpma12+dW1ur+s3IGwvdZ9r9P6dSaWorlL9oz9SstKeDqTdmVFsRgl3/xidkPU7iIWo6Mu5F45ncHEhliM5i4Z06ehnOneg3wkRYvKjZNSzOVaLjkTV7mtkJavd7HTuJQEgIcGsIxSjAltcQ2x5aej7RsAMGxKEwhZiMlJf7gYVvqjqvwVLYpza0woZ5qRckyOyMdx0wdUYrAFacfxcQ1R7OHMXYoxl9wNLXAwv7YPPj3jWpk3Fc08UT1qcjkiolCIiHLMCtdYqueict6ZT8+46ku2bYxQioVy0aLYuXTRomX5+fTMwpCKtWelmHpuUqknMZWPayzVXfKMISZH0WJReS+4yF1yLZcSWdeQi9onPn0pp98QpViIiuLpZUY9BSsnY9cM2fKY6kEwKwaJKCEmlCmWp4jgUpQJxdwcEa3F1CYeiY0Gzcgbu80uCwCHzH4zrnx6hrnkbogbCad0h165C12KSW0+xvRkCzDujG0Ii1HycVyUfNGoj0xJoWyQgll1poYyvzTPRuL0qGsCB0racbqeuevLM3L6xJA90b22VwtHfn2l2trU15lYilbZBbWkUuWL8oGiHA0bp5dDOdraUevjc6sxkLHHqK6BZ/fU7DCUYqEckamFRau+M67yABa62qrVMvddHUxfVj1dirXB+VoTrsUoTZ92BrO6RKUhyHEmNsRskKhRt1VCuVEn31ZIWZulWIiK6rlYn3H1ceMYtfbZSTeYFcXsLBHRXFbdeHx6Qd0j7Zo1B4CDtd/A1ZnYUK5wnFKrEPk0fMndQP84gBFTrzPX5qQQtLrlXuJ82nWr4YJzLUfTp1sukIJZ3H+abMpeYg5CW+j3Fvfihulmh2h6bau9prTMErdo8YsUXOR8VbVXq9TRcG1O3Fh06xo0hnLRaLRabxBJEbDpMGnTY1Q88E6jpVhoiysq7S3bhI+W3WJ7anWbCy3QVWUz50bWc1S+Z6LtksFszxcbulsOIxfMbmwoHTmzpDRFj6XT+j6uLdnmsTE7H82FeryFIXfwNd/3UXY3URRFcanuOvD7QABgO4MdVbi0liP57qitrlkADp9SjGG0nufZIEk3tTcSQe26tcER61LHDA7lfJaX9jDhlEhGF3Q2TAOjdApqzBlXG+LTl3K++VnLHyVn4irnU6IwKd2VpZZBG7KLbrmneKNetVP0o+HTMyEqbiTksgWzjfkVl3kc8Q63JQwswwMfd1XeA4KLnK97Xndw1pXhwFxstbrlvqrbJaklsyfFf61l43e2Dqq8/VPzxXQ6MRF9XJ3ORFZscKQf6qwt6bhrGVRsI+HUUrdyKhr5EIDDbTCBq+6GPfU85B8ADJF8Ca7/lVcf7KHQWoVK3a2sAxPt+G537Q+TxtnrYKzy1WSvObvRKC2zpMZcrZyJpajcElHO/IRy+ta0sRIROWfnfbm1kmUI3KbhLXOAox3JnXUNx7szsSEaglerXKv1LQrr8MCiPcYBmVV3s2LUdInhUlLp0q06tmpuKqw+9qtRr46wCq1KMV2OUdlGPvdOTJ98tOedoF4pjbT1A5GbGnZoP0n8+kqViNbSuGAEgLb2G7hKZ6MFWrL+8ZN/ZwDgoFlkV1xs1aopaKyku8CQBbMN9yU5a6Fd7tovJIFhMY7pacDvbNH0aeqQvrPVvUo+fanbg2zU9sLBrDGS0z2Cw5lYiuZCLtbUU7Zzw9uDSRHx6RkXO21538mZ2BCL02xr5nWcWlzqo832d0q0nKXOeoxxrcwvKuNC2+pxKvpHkDXmVxbSfKNenT6dULOP8m44xqdd9YHJus1m6l2g1q60LA+LxrIufW9qQ/thW51ZAGAE9hu4SmcjtXESANhEywV1MUrk81kMbSJfWJsezmk1zAyuGg4TZ2KjMb9ifg4jKY1lnYmN1nxdg/PpW5TaQdverXpS0nWtpD6hJFYi5bFR6meDc1Eic+xTikkrSL1EN/QubdvRdHBKMcbFTncacjWYFcXitDYEVWmZrRpqsb/hpFxuH8n9f4dAat+rrl11hSp/8OmZ9qu4yrLKEJF8+lIvo1aNinN2nlhXKBedo9bunuN0k8Ggj3sFfPpSLspJp5MGt7XWUH+/DOeYMQ7iAWAgBtvHFQDsqRRjQrlocWNjY6nusr7QM6UyrJoQ4qrhkHEmNsSG+5LxnoWUcHWqM8ytMQwzE4vNMPJoKvbaTXocXCyYlW7hyIfB3BrDMIY0pnQIRaM5fRq6FAvlpCvz4KL6uOTgXLS6ss6TNLW3Tn57VYqFtrhG96yuthbkCE4/Rqv5OO8vR+w8Pa2LCKVhjQeGX1+pRueCwaxYpFBLBNeQnkxkuX35nS2fumZKMRdbjRaLxq1nG3w63UhclbL7plGFDyZnPxy9x618eoGdLkqjChM5Extza+1uodpx+wHAwRlk4Nqmn894d9EAGG9Kpz0q6kZoksdnxCUAWCnF9DuHlLqXdplYqSWDKQUqvmn33HyUqDrUKO1glGIMw4Ry0aIoBe2xEvHpGWlKNpttcMS6pLUj9QyXwwq5EywRkcvtUyJXZ2IpOswHefQ55rfcqFhsbT68rzI0OJ/y+782N8DeQXx6QU0OK3cW9FymHg56zsSGtGa0jRcMSlvPZhnMXMhVP+1KL8gBms1Kt2d8+lLPw/016tNFw4bU3UY13UK1120xADhwgpHYG57njRPaDPAgiq2PSwCA4VGOTbWvV/tHdaj9z6PFPq41cTRPCvNpXN0JOuwyyhN+ta6E5pm1XWm0e0rLj5Sx/2ObwhWjre8Voy3VLEblo8b8KCB5vganPQTW+tGj+2RRO1X7X2OtgOqMtjzM1drJj5Pu4fTk4xpKdVq2ibEuHS5WDoZ+2yk7im7LWFZ2fM66au20Y6DrM6H0h0rrthn5BgOAkTL93jGCIOhPICdPnuz6C0FEu7u7p06d6mVOADhIODahR5O9q6B24wu1G1+TXTsAOHimswr6uAIAAAAAAICtIXAFAAAAAAAAW0PgCgAAAAAAALaGwBUAAAAAAABszTw401tvvTWqogAAAAAAAABI9IMzPdThvQ4wcByAPd2+fbvHscHhkJvsXQW1G1+o3fi6ffu2489+POpSAMDkEF74uPH1YJ7jCgC20PtRDIfcZO8qqN34Qu3GlyAI9MJrYuoZIsK/+Bf/4t/9/vvCa6ZzJp7jCjBRJvt2PgzQZO8qqN34Qu3Gl5xx/bNfpRdeG3VZAGDM/dmv0guvCS98XH/OxOBMAAAAADAYUrYEAGA/LM8k5j6uAAAAcAidPXu2r/mvX78+pJIMw2TXzlaY5A16YdSFAIAxZ3kmQeAKAAAARP1Ea/3GgXYwebV757XS7Vev3H/3VSI6+vhnHZ+NO371d9vNzKdnXPUlMRscdqnE1DPM7WF/CQBMOMszSZfAlWEYURT38aV8esbFThcP4EQJACalGBPa4hobCeeoSwIAAIP1o7+59P7rf3Hi5NGjTzNEdP/u9Z9UIz//8Z984neWzLPy6RkXWyWi6EEUrE3G9Tj3p5+gf/86+6b1p3yBT2/89vG2C/3+j5iX3x1QAQFgDFieSbr3cWUYpvMMpRjDzKT5PZcLAIaC39kadRFg3PDpGSZWMk8txRjFuJ7sJ6AKADrCa3/9/ut/8diTDx05xtz92YNfCPfv3xMffeLo+6//xTuvGQ5hPj3DLNBVUSweSNRKbfu43lnZosS/esrX4ZPf/xGTvNH638zf3hlSUQHAtvbex3XfeVcAOHjOxIaY6GnOUowJERpGHHLtcjJy4l5MOKV5XDM0bln8CaiCbXzrW98KBoMnTpwYdUEOO+HVzImTR48cZe68d//hTy1+5Nejb/997u4byydOHr396pUnflU7m6u/BI2DKpucJ/n8L4t/8OGWN49vpD5imIBUKgBY2VcfV8SuAACTik/PuFbmG6LYiDEh0zuXchQtylGeM3GVW3GtrPOJMQr7JqAKdvH1r3/9G9/4BhH93u/93qjLcth98O73TnxUaiH84CnvF5mjD3/k16P/X+N/e/ixIx/c/N5oy6b1TLv19szXblV7/+SvfUJMfcL6re8PoGAAMEYs+7j28Ticrm2GZboWWUxsvfflA8Bg6Zrx8+kZhomV+PSMenBKjcn49AzDhHJEuRBaUR5ezsSGaJmC5NdXqhSd02VvTk9TdWV9jHaTCaiCPUhR6x/90R8harWVow8fubX59Qd3f/723+eOHpOu00acZmCSN3SvjnN/+oyYsv6vETB2akVTYQBQGM8ksv6e49o9di3FmFAuWhQlRWLZPm61AcAw5UILdFU6NKOUC8VKJEUsYjFKJB22aD8JLXxul+6Vy92pi5pNTUAVDtTbb79tmqJGrV/84hdHUiQwOf7U56T49NijR+6+sfxf/89Td99YPvbokft3xYce/+xoy2bsmXaH/doNJvmjEhHdensmeYNJv80TEd1Jp2+4ylpEWi2/3q7NcIe3AGBSDeA5rt1aC5dioZyPa6g95YLZBrflYvv6DgAYEh93VY5Lg4ucL8eulbJBdGuFThr1KtG8eWq13iAal1scE1CFg/Wtb30rl8u9+OKLv/IrvyJNQdRqQx/85P85/uQxIjpylDn+4aPSxAf3xV/cvv+E78JIi2bZM+3dUPJd+vwvi6lnpPBVl9Q4zv3ppxNP9bjs92LJH+YGVVAAsLH9Pse1ex9XfmeLaPo0rgUAbAkHJwyGMYE5liagCsMSDAa/853vPP/881LsiqjVhna+/tTjH3+YOULiAxIfiMwRIqL7d8Vf3L7/yKf/RD8y00joe6bNP/vMxq8Z337qI/rxmfi/fd31tRtShsMX+PTGR99GchUAaG/PcdU+jJGZAAAOG5fbRzlDctI6gWljE1CFg3XixImvfvWrX/rSl55//vnf+q3f+uY3v4mo1VakqPXow0eI6IM7D+jRf/aLH9WI6Ojjn3vCd2HkUSsZ8yQrL9/QN7szhKYfe6qR+ND6q2pr4ceXfpvSaUStAEC0n4xrr1Gr8/Q00dYOT0HjBcJ0z2UEAAD7cM7O+9gV3Vmd39ki3/zsGOXuJ6AKB06NXRG12o0atd6/++DOe/d/+tYHv/Y/f2vUhTKT8iS+jz5M//Re+3FOjnP/6iP0t6+zbyoTPnbcRceDiWfaPMUNjYQBDpc9Zlz7ybUG56KUYxfSs/IIL6VYCGcZgLFgvOUEIHEmlqJsSDmr8+kFthotbozVjjIBVRgBKXb9u7/7u3Pnzo26LCCrf+3JX/rUcSnXeue9+59ceLPrR/SC2QNqO8ckb9ALx+enj5f+pkP69A77NUMylt685Uresp73878s/s7drUGWEQDszjLj2mVU4X5bCAezYjFaZV3yAzfW5hocRm8EsLtgVjlu8TgcMAtmGxzJZ3UXO10Us6NvitinCajCKJw4cQJRq60c//BRKdf687fvvfvju6MuTlti6hn6/JOJp95b+4fBLDA6/WF+6108pALgUBnAqMKWTPfwgllRzOpfi21afQDAcOmOTWdiw3Akml+bj1s4pCxzMua9ZQxNQBUAJHvItR4wJnkjuvXh0n+4obW5+/wvi3/wYenP0n/o2ov18WLqE4abS7fennkZj3IFOFz2O6owAAAATLCzZ8+OughDNO61+/nb99R/7UxMPcO8fMPQU+wffsj0kX19N5TEEE0Ah92+RhUGAACACXb9+vVRF2GIJqB2n3nhJ6MuQk8s8yQAAH3ZSx9XAAAAAIAeWfZMAwDoi/WZRDASe8PzfI9zAsBB6v0ohkNusncV1G58oXbjSxAEeuE19YoT/+Jf/It/9/MvvfCa6ZzJCIJAOidPnqQe7O7unjp1qpc5AeAg3b59u8ejGA65yd5VULvxhdqNr9u3bzv+7MejLgUATA7hhY/rz5nmwPWtt97qcUFPPvnkIMsFAAAAAAAAoNAHrubBmXrMo+7u7k7wLUOA8TXZt/NhgCZ7V0HtxhdqN74mu3YAcPBu3zaMLIzBmQAAAAAAAMDWELgCAAAAAACArSFwBQAAAAAAAFsz93EFAACAQ+js2bN9zX/9+vUhlWQYJrt2AACHAQJXAAAAIOonWus3DrSDya4dAMDEQ1NhAOhbKcYwsdKoSwEAALZVijEzab5lMp+e6fT7wadn8PsCANYGELjy6RmGaT3L4NQDADA2SjFG0XKt2em9MTEBVQAYP9OnnerfXQJWZR4XWyXKhZgWuKIEgMFlXHMhnFMAAMYRn55ZmxMlDY5Yly64K8WY0BbXsHxvTExAFQDGDr+z5XO7tJfrK1X961alGOOqL4ktGpyPKFrMBodfZgCwtwEFrr5o1IfQFQBgLDkTG+pFoXN23kfVekN6xacv5Si6lHDK813lfNWV9bEK+yagCgDjpRRjGMbFVqusS2nmwK+vVKnKuhjGxVZ1GVX5LlIpxjCX3A2xJTotxRjXyrzFGwBwCA0q4+pevMp1DF3lBsWm5h5STzntvZk0b5jVuDj9MnDDHKCjUsx8mOimtD8eZ9Il6T3ts20bWbY5VPtaONgbv75SpeicdsnoPD1N4xX2TUAVAMZMMCuKxSj5uIYoisUoETXWV6rSS7HB+ShaVNKpGwknn55hLrkbyp/aD0cpxjAhKoobCWfHrwOAw2JwTYWdiQ6hK59epqtqk49cSH/ZmgstSO81OJ90M05uKVKM6hen3HRDYy+AXgTnooarc13aqdPxWGVD0gEoXSmUYozWyFIUr9K6eoCrR650qCoL6WPhYEd8eoGtGlrlGVv3udy+gy/Ufk1AFQDGC7+z5Zuflc/00/U1tqo2ezBzJjaUHwVnYkMsRpV87CV3Q0SqFQA0gxxVuEPo6kxklfOVM7EU1ZqhEZGPu6qcr5aipOvHEJyLEm3tyAmiS7locUNbCBp7AXQWXNQfJPq0U6fj0dCRSAp2tQOPnImE+qZ65FJwkdPalva8cLAVNU9ubJXXqFctZjZsVLubgCoAjJ9GvSqNzMTvbPnci9meI9DSWo5IStbiFicAGA32cTidsq5ac8NQzvCGfsw5MtwZ126L8+srVeMgcy62iisPgE6cs/Nq5Mqvr1R93KJ63dD2eDSPpWFoY2lgOnJ1elo42IszsaGm1Re6DeA5ARtyAqoAYGOltZz048Gvr1SnTzt1XUiMfVzVU40yh/SzofSO1aCVHQAM/Dmu1qFrKcYwoZzSpaEY3duytS4RCiRvADpwzs77quxySY5blVZbgzke2xjqwuEgOBMbxSjl1kpE0u1D4y1C6wSmjU1AFQDGTnAumgsxsRK/vlKNzgV1t8ZMfVyzQfle5wItGd4wwG8JABANPnDVQteFFXVSaS1HPq4hh5n8zlb/Cz09rbYaBoAeORNLUcqtSZcOSvei3o/HPRx3+z/YwVacs/M+wz7A72yR1nNtHExAFQDGTzArikUKuVjStfVpP6sobiTQCgIAuhh84KqErtWq4Z62csObTy+we7jZHVzkfFXWpSVy+fQMHr4D0E1wLkq5SwvSLW9Nj8djy3FXivVw2O3zYIeDV4oxhpNrKKc2EXcmlqJVdkEdemuhwxArNjUBVQAYR9KtyyrG0gSAQRlG4KqErqpgtsH55A4NC3R1Ty0+nIkNbSHSctBQGKCr4FyUqlX9Le9+jkfjCI8MszbX5bAbxMEOBy6YbbgvaR3Qpov6jhjBbIMjub+Zi50ujmEfjQmoAsCY4dMz0rlEFIvT6o2jrgyjmWhM4yUAwCHFCIKgf33y5MlePra7u3vq1KnhFAkA9u727ds9HsVwyE32roLa7cHZs2evX78+jJn7gtqNL6V2fHrGtTLf0EYFLsXaxp5R9U4Sn55x1Zcs7yuVYswldwOjDAMcOqZz5kMjLAoAAADYx9mzZ0ddhCGa7NrZiTOxISb0E4JZUcz28qk2bwWzIppJAAACVwAAACAaUo7RJia7dgAAh8Fw+rgCAAAAAAAADAgCVwAAAAAAALA1BK4AAAAAAABga+ZRhd96660eP/nkk08OoTwAAAAAAAAA1GlU4R4fcrO7uzvB47kDjK/JftYCDNBk7yqo3fhC7cbXZNcOAA7e7du39S/RVBgAAAAAAABsDYErAAAAAAAA2BoCVwAAAAAAALA1cx9XAAAAOITOnj3b1/zXr18fUkmGYbJrBwBwGCBwBQAAAKJ+orV+40A7mOzaAQBMPDQVBphYpRjDzKT5URcDAAAOp1KMYWKlPuY2/2aVYn18HgAm3CADVz49w+A6GcAu+J2tURcBxoV0+taZtEvFUkytGn6kAA5KaS3n4xaD3edrjVjVBVDuEo5ZACCigQau/PpKNRqNVlfWcYIBsAFnYkMUNxLO7nP2d08cJpSPa4iqbPdLzfFRijGhLbl2DY5YF2JXgANQioVyVGVdTHstx6I8uzSdT1/a4hrFaXYBhywA0CAD19IyW43OLbp9VXYZV8AAAGOkUa/S9OkebnKMIT59KUfRJfkWjjNxlfPhBivA0JVioVy0KOo0OB8Zp7TeW5VvoG0knESlZXZ6KeEMZhG6AgARDTBwLa3lKDoXdM7O+yi31hK5GpppldIzxvyOvpUa7oQDDIiujysvHXS6Q00+Avn0DMOEckS5EI4/mET8+kqVonNaBtl5epoQuQIMlxS29tR0oxRj5MysLkEbK5ViIeLcl5hYiYLZxvwKWkoAwIACV+mO9lyQSIpcDf0R+PQMo7vttlQPsVXdZ0sxxrUy3xDRigtguHKhBboqiqIoFqOUC8VKJLcnLkZJvg3eU8timDj8zpZy72ISO7iSz+3SvXK5fSMrCcChILXxzZKWtGAYhnGxVd15Rj3ZBLNiiyKFLrkb2URio0ghJlZyJjYa8yuuiTs3AUBfBhO46u9oO2fnfYa72aVlturjGuptt2C2wWlXDXz6Ui5aVK+W0YoLYGh83FX5SAsucpZNI+CQciY29K35cqEJil0b9arF1Gq9ceAlATg0nIkN+cJO33feoqlwNkhSfiNWIirFpMRFKcaszSn3UYNZeS5nYmOyOt8DQN8GEriWllldSywpclU7uvI7W9S+7xS/vmK6/+Ziq7ikABiGSe3DCAPlTGwUoxM/jqcxBwsANrNjHul8EtuCAECfBhG4ltZypG9l5mKrRP1kc0z33yZtQEsAgLEyUW1pXW6f6WaodQ4WAEaL39mqqoMwndY1A5FytT09VwcAJtr+A1c+fSlnfIyCfIqRb9g7T08Tbe3o793rrhos3gUAgFGaqDGGnbPzPsOvDL+zRb752QmpHoANKeMAmp6GY9HHVTckYGmZnS4Wp9mF9I66kJk0T8SnZ1wr8w0MwQAA+w5cpe6tS6bTiTOxFFU6ugbnolTVDWReioVy2pzBRc5XZXX97fn0DBqDABw43D86zJSuZUREfHomlJuk3IYzsRTVfoP49AJbbfnNAoABchrSpZ36uKpDAuZCISpmg8FFjlg2Jy+kOM26GMbFThcRtQIA7T9wNXZv1ZHC1eUSEQWzYjGq3XNbm9MPzkTOxIY0Fohsga6ioTDAgQpmlUMUQ3ofTsHsVVpQkyLTxQkbXjqYbXAk/wa52OkieqMA2FAupPY2kwWz0qj3GEoQAIiI6KF9fj6YFcVs93eMs/HpS4ZZnYkNMbHPggCAWTArivKf5oPM/Lr9kQyHw4Sfhie8egBjzXx8lmLMmvoimBUb7hlXrIQbTgCw38B1L+TWxTgBAQAA2MjZs2dHXYQhmuzajQFnYkPsPhdRMCsaLhF7/RwATLqDCFz59MwCXVVanpViLrbq464ibgUAALCN69evj7oIQzTZtQMAOAwOInB1JjaWYgzDKK+j6GAEAAAAAAAAvTqgpsLoQAcAAAAAAAB7s//nuAIAAAAAAAAMESMIgv71W2+91eMnn3zyySGUBwAAAAAAAIBOnjyp/m0OXPXvAQAAAAAAAIwcmgoDAAAAAACArSFwBQAAAAAAAFtD4AoAAAAAAAC2hsAVAAAAAAAAbA2BKwAAAAAAANgaAlcAAAAAAACwNQSuAAAAAAAAYGsIXAEAAAAAAMDWELgCAAAAAACArSFwBQAAAAAAAFv7/wF4J1/N191fGQAAAABJRU5ErkJggg==
