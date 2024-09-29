
## 为什么要写


为什么要写，大概就是沉没成本吧


只是从Source Generators出来开始，就打算以其研究是否能做 aop （现在已经有内置功能了），本来当年就想尝试能否在 orm 做一些尝试，可惜种种原因，自己都忘了这个打算了


直到今年7月份，才又想起了这个打算，现在精力不行了，本来研究一下原理和功能限制也算完了，


可是都写了蛮久了，不写完整点，感觉有点浪费，又花了不少时间才把 类似 [DapperAOT](https://github.com) 功能做的差不多


写完了又觉得光复制复制功能 没意思， 所以又把以前为了在公司内少写一些比较重复性代码做的 查询定制功能 在 sv.db 基础上搞一把， 反正成本已经花了那么多了


虽然这也不算多大创新， 类似想法老早就不少人搞了，比如最夸张的 graphql 这一套甚至让大家的基础设施都得搭一套，


不过越搞越多，都到国庆了，目前还只能算主体完成了，细节还有很多没搞


很多转换 解析都是手写的，不是因为写的nb，只是引入其他库兼容可能有点麻烦，场景不多，递归遍历就够了


nuget 什么的也没空搞了，bug 估计不少，等后面再完整点再搞，反正都是自娱自乐


真是年纪大了，做什么都越来越慢了


## sv.db 能做什么


### 1\. db映射到实体


像 [DapperAOT](https://github.com) 一样，使用 [Source Generators](https://github.com):[westworld加速](https://tianchuang88.com) 在构建期间生成必要的代码，以帮助您更轻松地使用 sql。


理论上，您还可以进行 [Native AOT 部署](https://github.com)


其实没什么，举个栗子吧



```
public async Task<object> OldWay()
{
    var a = factory.GetConnection(StaticInfo.Demo);
    using var dd = await a.ExecuteReaderAsync("""
SELECT count(1)
FROM Weather;
SELECT *
FROM Weather;
""");
    var t = await dd.QueryFirstOrDefaultAsync<int>();
    var r = await dd.QueryAsync<string>().ToListAsync();
    return new { TotalCount = t, Rows = r };
}

```

### 2\. 让查询编码简单，并支持更复杂的条件


通过定义一些简单的查询规则，我们可以将 查询转换为 db / api / es 查询语句 ....


目前只搞了 db 转换，并且还没空完整适配测试， es 、mongodb 什么等后面有空把


不过理论上，我们可以这样做：



```
http query string / body  |------>  select statement    |------>  db (sqlite / mysql/ sqlserver / PostgreSQL)
Expression code           |------>                      |------>  es
                                                        |------>  mongodb
                                                        |------>  more .....

```

中间 `select statement` 这一层已经定义好了，前面的转换也有了， 后面的理论加上适配，什么都可以做，


#### 2\.1 举个 api 的栗子


##### Code exmples:


首先定义一个 实体配置， 列明哪些字段可以查，可以排序，可以筛选



```
[Db("Demo")]
[Table(nameof(Weather))]
public class Weather
{ 
 [Select, Where, OrderBy]
 public string Name { get; set; }

 [Select(Field = "Value as v"), Where, OrderBy]
 public string V { get; set; }

 [Select(NotAllow = true)]
 public string Test { get; set; }
}

```

然后定义 查询接口



```
[HttpGet] 
public async Task<object> Selects() //  你可以自己做一些字段，授权检查 [FromQuery, Required] string name) 
{
    return await this.QueryByParamsAsync();
}

```

接着你就可以让用户自己拼写各种条件，让她们自己满足自己的场景，这样自己可以多摸一会鱼了



```
curl --location 'http://localhost:5259/weather?where=not (name like '%e%')&TotalCount=true'

```

Response



```
{
    "totalCount": 1,
    "rows": [
        {
            "name": "H",
            "v": "mery!"
        }
    ]
}

```

##### 2\.2 同样可以让查询代码更简单


##### Code exmples:


其实很多 orm 都提供用Expression 达到类似或更复杂效果的


这里考虑工作量，restful api 和其他数据查询实现 支持度不一，目前只做基础filter 支持， join 什么都不搞了，就算搞了多半又会被骂，直接写sql 不更好吗？


比如下面 query Weather which name no Contains 'e'



```
public async Task<object> DoSelects()
{
    return await factory.ExecuteQueryAsync(From.Of().Where(i => !i.Name.Like("e")).WithTotalCount());
}


```

这里就非常简单介绍一下，复杂的，怎么实现的不写，累了，写不动了，要有感兴趣的 可以在 gayhub 看源码 [https://github.com/fs7744/sv.db](https://github.com)


## 下面再列举一下过滤操作符支持情况


### Query in api


Both has func support use query string or body to query


body or query string will map to `Dictionary` to handle


#### operater


such filter operater just make api more restful (`Where=urlencode(complex condition)` will be more better)


* `{{nl}}` is null
	+ query string `?name={{nl}}`
	+ body `{"name":"{{nl}}"}`
* `{{eq}}` Equal \=
	+ query string `?name=xxx`
	+ body `{"name":"xxx"}`
* `{{lt}}` LessThan or Equal \<\=
	+ query string `?age={{lt}}30`
	+ body `{"age":"{{lt}}30"}`
* `{{le}}` LessThan \<
	+ query string `?age={{le}}30`
	+ body `{"age":"{{le}}30"}`
* `{{gt}}` GreaterThan or Equal \>\=
	+ query string `?age={{gt}}30`
	+ body `{"age":"{{gt}}30"}`
* `{{gr}}` GreaterThan \>
	+ query string `?age={{gr}}30`
	+ body `{"age":"{{gr}}30"}`
* `{{nq}}` Not Equal !\=
	+ query string `?age={{nq}}30`
	+ body `{"age":"{{nq}}30"}`
* `{{lk}}` Prefix Like 'e%'
	+ query string `?name={{lk}}e`
	+ body `{"name":"{{lk}}e"}`
* `{{rk}}` Suffix Like '%e'
	+ query string `?name={{rk}}e`
	+ body `{"name":"{{rk}}e"}`
* `{{kk}}` Like '%e%'
	+ query string `?name={{kk}}e`
	+ body `{"name":"{{kk}}e"}`
* `{{in}}` in array (bool/number/string)
	+ query string `?name={{in}}[true,false]`
	+ body `{"name":"{{in}}[\"s\",\"sky\"]"}`
* `{{no}}` not
	+ query string `?age={{no}}{{lt}}30`
	+ body `{"age":"{{no}}{{lt}}30"}`


#### Func Fields:


* `Fields` return some Fields , no Fields or `Fields=*` is return all
	+ query string `?Fields=name,age`
	+ body `{"Fields":"name,age"}`
* `TotalCount` return total count
	+ query string `?TotalCount=true`
	+ body `{"TotalCount":"true"}`
* `NoRows` no return rows
	+ query string `?NoRows=true`
	+ body `{"NoRows":"true"}`
* `Offset` Offset Rows index
	+ query string `?Offset=10`
	+ body `{"Offset":10}`
* `Rows` Take Rows count, default is 10
	+ query string `?Rows=100`
	+ body `{"Rows":100}`
* `OrderBy` sort result
	+ query string `?OrderBy=name:asc,age:desc`
	+ body `{"OrderBy":"name:asc,age:desc"}`
* `Where` complex condition filter
	+ query string `?Where=urlencode( not(name like 'H%') or name like '%v%' )`
	+ body `{"Where":"not(name like 'H%') or name like '%v%'"}`
	+ operaters
		- bool
			* example `true` or `false`
		- number
			* example `12323` or `1.324` or `-44.4`
		- string
			* example `'sdsdfa'` or `'sds\'dfa'` or `"dsdsdsd"` or `"fs\"dsf"`
		- `= null` is null
			* example  `name = null`
		- `=` Equal
			* example  `name = 'sky'`
		- `<=` LessThan or Equal
			* example  `age <= 30`
		- `<` LessThan
			* example  `age < 30`
		- `>=` GreaterThan or Equal
			* example  `age >= 30`
		- `>` GreaterThan
			* example  `age > 30`
		- `!=` Not Equal
			* example  `age != 30`
		- `like 'e%'` Prefix Like
			* example  `name like 'xx%'`
		- `like '%e'` Suffix Like
			* example  `name like '%xx'`
		- `like '%e%'` Like
			* example  `name like '%xx%'`
		- `in ()` in array (bool/number/string)
			* example `in (1,2,3)` or `in ('sdsdfa','sdfa')` or `in (true,false)`
		- `not`
			* example  `not( age <= 30 )`
		- `and`
			* example  `age <= 30 and age > 60`
		- `or`
			* example  `age <= 30 or age > 60`
		- `()`
			* example  `(age <= 30 or age > 60) and name = 'killer'`


