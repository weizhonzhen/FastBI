![](https://raw.githubusercontent.com/weizhonzhen/FastApi/master/help.jpg)
![](https://raw.githubusercontent.com/weizhonzhen/FastApi/master/xml.jpg)

# Fast.Api.Framework
动态生成读api，只需配数据库连接及xml文件 net Framework
```csharp
in Global.asax Application_Start()
FastData.FastMap.InstanceMap("dbkey","db.config","SqlMap.config");
```

in web.config
```csharp
<configuration>
  <system.webServer>
    <modules>
      <add name="Fast.Api.Framework.FastApiModule" type="Fast.Api.Framework.FastApiModule,Fast.Api.Framework" />
    </modules>
  </system.webServer>
</configuration>
```

db.config
```csharp
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="DataConfig" type="FastData.Config.DataConfig,FastData" />
    <section name="RedisConfig" type="FastRedis.Config.RedisConfig,FastRedis" />
  </configSections>
  
  <DataConfig>
    <Oracle>
      <Add ConnStr="User Id=qebemr;Password=qebsoft;Data Source=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=127.0.0.1)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=emrdata)));pooling=true;Min Pool Size=5;Max Pool Size=50;" CacheType="redis" SqlErrorType="file" Key="Test" IsOutError="true" IsOutSql="false" />
    </Oracle>
  </DataConfig>
  
  <RedisConfig WriteServerList="127.0.0.1:6379" ReadServerList="127.0.0.1:6379" MaxWritePoolSize="10" MaxReadPoolSize="50" AutoStart="true" />
</configuration>

```

SqlMap.config
```csharp
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="MapConfig" type="FastData.Config.MapConfig,FastData"/>
  </configSections>

  <MapConfig>
    <SqlMap>
      <Add File="map/query.xml"/>
    </SqlMap>
  </MapConfig>
</configuration>
```
query.xml
```csharp
<?xml version="1.0" encoding="utf-8" ?><sqlMap>
  <select id="Consent" db="Test">
    select  *  from t_xt_yhb yh where 1=1
    <dynamic prepend="">
      <isNotNullOrEmpty prepend=" and " property="gh">yh.gh = :gh</isNotNullOrEmpty>
      <isNotNullOrEmpty prepend=" " property="OrderBy">order by #OrderBy#</isNotNullOrEmpty>
    </dynamic>
  </select>
</sqlMap>
```


# FastApi 
动态生成读api，只需配数据库连接及xml文件

1、ConfigureServices
```csharp
//old pagepackages
services.AddFastApi();
FastMap.InstanceMap(dbKey, "db.json", "map.json");

//new pagepackages
services.AddFastApi(new ConfigApi { mapFile = "map.json", dbKey = "dbkey", IsResource = true, dbFile = "db.json" });
    or
services.AddFastApi(a=> { a.mapFile = "map.json"; a.dbKey = "dbkey"; a.IsResource = true; a.dbFile = "db.json"; });
```

2、Configure
```csharp
FilterUrl 要过滤的url
IsAlone独立站点使用
app.UseFastApiMiddleware(a =>
{
     a.IsAlone = true; 
     a.FilterUrl.Add("help");
     a.FilterUrl.Add("xml");
     a.FilterUrl.Add("del");
});

跟业务系统（mvc或webapi）一起使用为false,FilterUrl可以不用写  
IsResource 指xml是否嵌入的资源
dbFile 是数据库的文件
app.UseFastApiMiddleware(a =>
{
     a.IsAlone = false;
     a.IsResource = true;
     a.dbFile = "db.json";
});
```
 
3、配配数据库连接 db.json
```csharp
{
  "DataConfig": [
    {
      "ProviderName": "Oracle.ManagedDataAccess",
      "DbType": "Oracle",
      "ConnStr": "User Id=test;Password=test;Data Source=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=127.0.0.1)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=test)));",
      "IsOutSql": false,
      "IsOutError": true,
      "IsPropertyCache": true,
      "IsMapSave": false,
      "Flag": ":",
      "FactoryClient": "Oracle.ManagedDataAccess.Client.OracleClientFactory",
      "Key": "Api",
      "DesignModel": "DbFirst",
      "SqlErrorType": "file",
      "CacheType": "web",
      "IsUpdateCache": true
    }
  ]
}
```

4、配置xml读取map.json
```csharp
{
  "SqlMap": { "Path": [ "map/Patient.xml" ] }
}
```

5、配置xml和成接口
```csharp
<?xml version="1.0" encoding="utf-8" ?>
<sqlMap>
   <select id="testurl" db="Api" type="param" name="备注">
    select * from table a
    <dynamic prepend=" where 1=1 ">
      <isNotNullOrEmpty prepend=" and " property="name" name="备注">a.name = :name</isNotNullOrEmpty>      
      <isNotNullOrEmpty prepend=" and " property="id" name="备注">a.id = :id</isNotNullOrEmpty>
    </dynamic>
 </select>
 
  <insert id="Write/Test" db="Api" type="write" name="备注">
    insert into aa values (
    <dynamic prepend="">
      <isPropertyAvailable prepend="" property="id" existsmap="CheckTestId" name="备注">:id,</isPropertyAvailable>
      <isPropertyAvailable prepend="" property="addTime" date="true" required="true" name="备注">:addTime,</isPropertyAvailable>
      <isPropertyAvailable prepend="" property="key" name="备注">:key,</isPropertyAvailable>
      <isPropertyAvailable prepend="" property="a" date="true" required="true" name="备注">:a,</isPropertyAvailable>
      <isPropertyAvailable prepend="" property="b" maxlength="10" name="备注">:b</isPropertyAvailable>
    </dynamic>
    )
  </insert>
  
    <select id="CheckTestId" db="Api" view="~/views/home/index.cshtml">
    select count(0) count from aa
    <dynamic prepend=" where 1=1 ">
      <isPropertyAvailable prepend=" and " property="id">id=:id</isPropertyAvailable>
    </dynamic>
    <foreach name="data" field="userId"> //子查询 name是子节点的名称，field是父节点字段,多个","分开
         select ypxh from base_role where userId=:userId
    </foreach>
  </select>
 </sqlMap>
 db对应db.json中的key
 type分五种 
 	1、"all" 查询所有可以不用传参数 
	2、"param" 根据参数查询参数必须填写 
	3、"page" 分页查询参数如果没有查询第一页
	4、"write" 写操作
	5、 "hide" 接口界面不显示
	6、 "pageall" 分页参数可以不传
 	dynamic下节点 isPropertyAvailable、isNotNullOrEmpty、isequal、isnotequal、isgreaterthan、
			islessthan、isnullorempty、isnotnullorempty、if、choose上的属性有6种
 	date 是验证时间
	required 是验证必填
	maxlength 是验证最大长度 
	existsmap 是验证是否已经存在 不存在是通过验证
	checkmap 是验证是否存在，存在通过验证

view 直接返回视图 
	type 为 page or pageall 返回 model 为FastUntility.Core.Page.PageResult
	type 不是page 和 pageall  返回 model 为 List<Dictionary<string, object>>

查看所有接口地址：http://127.0.0.1/help
配置接口地址：http://127.0.0.1/xml
访问的地址：http://127.0.0.1/testurl?name=aa&id=1
访问的分页地址：http://127.0.0.1/testurl?name=aa&id=1&pageid=1&pagesize=10
支持post get 等所有方式请求

