# MongoDB学习记录 #

## 一，了解和安装MongoDB数据库 ##


### 1.MongoDB是什么 ###
 MongoDB是一个基于分布式文件存储的数据库，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是它支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

### 2.MongoDB的特点与不足 ###

**特点**
1. 面向文档存储(类JSON的bson格式，因此可以存储比较复杂的数据类型)。
2. 支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能。

**缺点**
1. 不支持事务
2. MongoDB没有成熟的维护工具。

### 3.MongoDB的下载与安装 ###

下载地址：https://www.mongodb.com/download-center/community

**安装注意事项：**
	
- 服务端需要安装在c盘，修改安装盘会报错，
- mongodb安装包没有数据目录，日志与配置文件，需要手动创建，
- 数据与日志需要在硬盘根目录下创建data文件，

    	c:\>cd c:\	
        c:\>mkdir data 	 
    	c:\>cd data	  
    	c:\data>mkdir db	
    	c:\data>cd db	
    	c:\data\db>	

- 配置文件需要在安装目录下创建.cfg文件，配置信息如下：

    	systemLog:
    		destination: file          
    		path: D:\data\logs\mongod.log  
    	storage:  
    		dbPath: D:\data\db


**创建admin用户：**
*注：mongodb严格区分大小写

    > use admin
    switched to db admin
    > db.createUser({user:"admin",pwd:"123456",roles:["root"]})
    Successfully added user: { "user" : "admin", "roles" : [ "root" ] }
    > db.auth("admin","123456")
    1



## 二，创建，删除数据库 ##

### 创建数据库 ###

    > use Mydatabase2   #如果数据库不存在，则创建，否则切换到指定数据库
    switched to db Mydatabase2  
    > db  
    Mydatabase2  
    > db.Mydatabase2.insert({"key2":"secondmongo"})  #向数据库插入数据
    WriteResult({ "nInserted" : 1 })  
    > show dbs    #查看所有数据库
    Mydatabase2  0.000GB  
    admin    0.000GB  
    config   0.000GB  
    local    0.000GB

### 删除数据库 ###

    > show dbs#删除前查看数据库
    MyDB2   0.000GB
    admin   0.000GB
    config  0.000GB
    local   0.000GB
    > use MyDB2   #切换到目标数据库
    switched to db MyDB2
    > db.dropDatabase()   #删除数据库
    { "dropped" : "MyDB2", "ok" : 1 }
    > show dbs#查看删除后的数据库
    admin   0.000GB
    config  0.000GB
    local   0.000GB


## 三，集合 ##

**mongodb中的集合相当于关系型数据库中的数据表**

### 1.创建集合 ###

**语法格式：db.createCollection([Listname], [options])**

参数options包含如下：
- capped：是否是固定集合。固定集合达到最大值时，它会自动覆盖最早的文档。
- size：为固定集合指定一个最大值，以千字节计（KB）。
- max：指定固定集合中包含文档的最大数量。

示例：
创建一个固定集合‘gaorylist’并通过renameCollection命令对其重命名

    > db.createCollection("grylist",{capped:true,size:1024,max:100})
    { "ok" : 1 }
    > show collections
    grylist
	> db.grylist.renameCollection("gryL")
	{ "ok" : 1 }
	
### 2.删除集合 ###

示例：

    > show collections
    gryL
    gryN
    > db.gryN.drop()
    true

### 3.新增操作 ###

使用 insert() 方法向集合中插入文档：

    > db.gryL.insert({aa:'111',bb:'222'})
    WriteResult({ "nInserted" : 1 })
    > db.gryL.insert({aa:'halou',bb:'worde'})
    WriteResult({ "nInserted" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "111", "bb" : "222" }
    { "_id" : ObjectId("5ece1493b28a1d21b81cf404"), "aa" : "halou", "bb" : "worde" }

也可以先将数据定义为一个变量在插入集合中：

    > value=({aa:'hahal',bb:'wowod'})
    { "aa" : "hahal", "bb" : "wowod" }
    > db.gryL.insert(value)
    WriteResult({ "nInserted" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "111", "bb" : "222" }
    { "_id" : ObjectId("5ece1493b28a1d21b81cf404"), "aa" : "halou", "bb" : "worde" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }


### 4.修改操作 ###

#### 1.使用Update语法 ####

db.collection.update(  
   < query>,  			#相当于sql中的where，用于筛选需要修改的数据  
   < update>,  			#相当于sql中的set，修改目标数据  
   {  
     upsert: < boolean>,	#如果不存在update的记录，是否插入新记录, 默认false，不插入  
     multi: < boolean>,  	#默认是false,只更新找到的第一条记录，如果为true,全部更新。  
     writeConcern: < document>  #抛出异常的级别。  
   }  
)


示例：

    > db.gryL.update({_id : ObjectId('5ece1465b28a1d21b81cf403')},{aa:'hl'})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "hl" }

上面的示例出现了意外的结果，要避免这种情况需要如下方式添加操作符 $set：
   
    > db.gryL.update({_id : ObjectId('5ece1465b28a1d21b81cf403')},{aa:'h2',bb:'w2'})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.update({_id : ObjectId('5ece1465b28a1d21b81cf403')},{$set:{aa:'hl'}})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "hl", "bb" : "w2" }

常见的更新操作符：

- $set：修改文档中某个字段的值
- $inc：对数字字段进行加减，
- $unset：删除某个字段的值
- $push：给某个字段增加值
- $pushAll：类似push，但是可以一次增加多个值，格式{$pushAll:{cc:["A1","A2"]}}

示例：

    > db.gryL.update({_id : ObjectId('5ece1465b28a1d21b81cf403')},{$push:{cc:'bmwxl'}})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "hl", "bb" : "w2", "cc" : [ "bmwxl" ] }
    > db.gryL.update({_id : ObjectId('5ece1465b28a1d21b81cf403')},{$push:{cc:'bmwx3'}})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "hl", "bb" : "w2", "cc" : [ "bmwxl", "bmwx3" ] }


#### 2.使用save语法 ####

**save() 方法通过传入的文档来替换已有文档，_id 主键存在就更新，不存在就插入。**

示例：

    > db.gryL.save({_id : ObjectId('5ece1465b28a1d21b81cf403'),cc:'bmwx3'})
    WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "cc" : "bmwx3" }

### 5.删除操作remove ###

语法格式如下：

db.collection.remove(
   <query>,
   {
     justOne: <boolean>,		#是否只删除第一条，如果为，则删除所有符合条件的数据
     writeConcern: <document>
   }
)

示例：

    > db.gryL.find()
    { "_id" : ObjectId("5ece1493b28a1d21b81cf404"), "aa" : "hahal", "bb" : "wowod" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    > db.gryL.remove({bb:'wowod'},{justOne:true})
    WriteResult({ "nRemoved" : 1 })
    > db.gryL.find()
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }


### 5.查询操作find ###

**前面已经多次用到了最简单的find语法：db.collection.find()**

#### 1.带条件的查询示例####

    > db.gryL.find({aa:'hahal'})
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    > db.gryL.find({aa:'hahal'}).pretty()
    {
    	"_id" : ObjectId("5ece15ebb28a1d21b81cf405"),
    	"aa" : "hahal",
    	"bb" : "wowod"
    }

#### 2.查询条件and和or ####

- and使用语法：db.gryL.find({aa:'hahal',bb:'wowod'})
- or使用语法：db.gryL.find($or:[{aa:'hahal',bb:'wowod'}])
- and和or混合使用：db.gryL.find({aa:'hahal',$or:[{aa:'hahal',bb:'wowod'}]})


## 四，操作符 ##

### 1.查询操作符 ###

	“=”：{aa:'hahal'}
	“>”：{aa:{$gt:'hahal'}}
	“>=”：{aa:{$gte:'hahal'}}
    “<”：{aa:{$lt:'hahal'}}
    “<=”：{aa:{$lte:'hahal'}}
    “!=”：{aa:{$ne:'hahal'}}

### 2.$type 操作符 ###

**$type 操作符用于返回某个数据类型下的数据：{aa:{$type:'string'}}**

示例：

    > db.gryL.find({aa:{$type:'string'}})
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    { "_id" : ObjectId("5ece2876b28a1d21b81cf406"), "aa" : "bmwx5", "bb" : "bin95" }


### 3.Limit() 方法和 skip() 方法 ###

**mongodb也提供了limit方法用来读取指定数量的数据，但是它在与skip()一起使用才能达到mysql中limit函数的效果：db.collection.find().limit(number).skip(number)**

示例：
    
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "cc" : "bmwx3" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    { "_id" : ObjectId("5ece2876b28a1d21b81cf406"), "aa" : "bmwx5", "bb" : 918.5 }
    > db.gryL.find().limit(2)
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "cc" : "bmwx3" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    > db.gryL.find().limit(2).skip(1)
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    { "_id" : ObjectId("5ece2876b28a1d21b81cf406"), "aa" : "bmwx5", "bb" : 918.5 }

### 4.排序 sort() 方法 ###

**在 MongoDB 中使用 sort() 方法对数据进行排序，并使用 1（升序） 和 -1（降序） 来指定排序的方式：db.collection.find().sort({KEY:1})**

示例：

    > db.gryL.find().sort({_id:1})
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "cc" : "bmwx3" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    { "_id" : ObjectId("5ece2876b28a1d21b81cf406"), "aa" : "bmwx5", "bb" : 918.5 }


### 5.聚合 aggregate() 方法 ###

**aggregate()的语法格式：db.COLLECTION_NAME.aggregate(AGGREGATE_OPERATION)**

1.常用的聚合表达式（相当于sql中的聚合函数）：

- $sum	计算总和。	
- $avg	计算平均值	
- $min	获取集合中所有文档对应值得最小值。	
- $max	获取集合中所有文档对应值得最大值。	
- $push	在结果文档中插入值到一个数组中。	
- $addToSet	在结果文档中插入值到一个数组中，但不创建副本。	
- $first	根据资源文档的排序获取第一个文档数据。	
- $last	根据资源文档的排序获取最后一个文档数据

示例：

    > db.gryL.aggregate([{$group:{_id:"$aa",cou:{$sum:1}}}])
    { "_id" : "hahal", "cou" : 1 }
    { "_id" : "bmwx5", "cou" : 2 }
    > db.gryL.aggregate([{$group:{_id:"$aa",cou:{$first:'$bb'}}}])
    { "_id" : "hahal", "cou" : "wowod" }
    { "_id" : "bmwx5", "cou" : 911 }

**2.聚合管道：一般用于将当前命令的输出结果作为下一个命令的参数，管道的构件有：筛选、投射、分组、排序、限制和跳过。**

- $project：修改输入文档的结构。可以用来重命名、增加或删除域，也可以用于创建计算结果以及嵌套文档。
- $match：用于过滤数据，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit：用来限制MongoDB聚合管道返回的文档数。
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档。
- $unwind：将文档中的某一个数组类型字段拆分成多条，每条包含数组中的一个值。
- $group：将集合中的文档分组，可用于统计结果。
- $sort：将输入文档排序后输出。
- $geoNear：输出接近某一地理位置的有序文档。

示例1：

    >  db.gryL.aggregate({$project:{'ssd':'$aa',bb:1}})
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "bb" : 911, "ssd" : "bmwx5" }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "bb" : "wowod", "ssd" : "hahal" }
    { "_id" : ObjectId("5ece2876b28a1d21b81cf406"), "bb" : 918.5, "ssd" : "bmwx5" }


示例2：

    > db.gryL.aggregate([{$match:{aa:'bmwx5'}},{$group:{_id:"$aa",cou:{$first:'$bb'}}}])
    { "_id" : "bmwx5", "cou" : 911 }

获取aa:'bmwx5'的记录，然后将符合条件的记录送到下一阶段$group管道操作符进行处理。


### 6.索引 createIndex() 方法 ###

**mongodb索引的作用与sql类似**

索引的简单操作：

    > db.gryL.createIndex({'aa':1})        #在列'aa'创建索引
    {
    "createdCollectionAutomatically" : false,
    "numIndexesBefore" : 1,
    "numIndexesAfter" : 2,
    "ok" : 1
    }
    > db.gryL.getIndexes()        #查看表中左右的索引，有两个
    [
    {
    "v" : 2,
    "key" : {
    "_id" : 1
    },
    "name" : "_id_",
    "ns" : "mdb1.gryL"
    },
    {
    "v" : 2,
    "key" : {
    "aa" : 1
    },
    "name" : "aa_1",
    "ns" : "mdb1.gryL"
    }
    ]
    > db.gryL.dropIndexes()    #删除索引
    {
    "nIndexesWas" : 2,
    "msg" : "non-_id indexes dropped for collection",
    "ok" : 1
    }
    > db.gryL.getIndexes()
    [
    {
    "v" : 2,
    "key" : {
    "_id" : 1
    },
    "name" : "_id_",
    "ns" : "mdb1.gryL"
    }
    ]
    
[https://blog.51cto.com/1937519/2299640](https://blog.51cto.com/1937519/2299640)


## 五，备份与恢复 ##

### 备份mongodb ###

**语法格式：mongodump -h [服务器地址：端口号] -d [数据库名] -o [备份文件存放路径]**

运行如下脚本，会在d：\data\log路径下创建数据库mdb1的备份文件：

    C:\Program Files\MongoDB\Server\4.2\bin>mongodump -h 127.0.0.1:27017 -d mdb1 -o D:\data\log
    2020-05-28T14:40:06.870+0800writing mdb1.gryL to
    2020-05-28T14:40:06.889+0800done dumping mdb1.gryL (5 documents)

### 恢复mongodb ###

**语法格式：mongodump -h [服务器地址：端口号] -d [数据库名] -o [备份文件存放路径]**

示例：

删除原有的mdb1

    > show dbs
    mdb1	0.000GB
    > use mdb1
    switched to db mdb1
    > db.dropDatabase()
    { "dropped" : "mdb1", "ok" : 1 }
    > show dbs

    > exit
    bye

执行恢复命令

    C:\Program Files\MongoDB\Server\4.2\bin>mongorestore -h 127.0.0.1:27017 -d mdb1 D:\data\log\mdb1
    2020-05-28T14:56:17.088+0800the --db and --collection args should only be used when restoring from a BSON file. Other uses are deprecated and will not exist in the future; use ......

重新连接mongodb查看数据已经恢复：
    
    C:\Program Files\MongoDB\Server\4.2\bin>mongo
    MongoDB shell version v4.2.7
    ......
    > show dbs
    mdb1	0.000GB
    > use mdb1
    switched to db mdb1
    > db.gryL.find()
    { "_id" : ObjectId("5ece1465b28a1d21b81cf403"), "aa" : "bmwx5", "bb" : 911 }
    { "_id" : ObjectId("5ece15ebb28a1d21b81cf405"), "aa" : "hahal", "bb" : "wowod" }
    ......