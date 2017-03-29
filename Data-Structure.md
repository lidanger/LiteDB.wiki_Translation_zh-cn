LiteDB按文档方式存储数据，这种文档是JSON风格的键值对。文档是无模式的数据结构。每一个文档同时包含你的数据和结构。

```JS
{
    _id: 1,
    name: { first: "John", last: "Doe" },
    age: 37,
    salary: 3456.0,
    createdDate: { $date: "2014-10-30T00:00:00.00Z" },
    phones: ["8000-0000", "9000-0000"]
}
```

- `_id` 包含文档主键 - 集合中的一个唯一值
- `name` 包含一个嵌入文档，有`first`和`last`两个字段
- `age` 包含一个`Int32`值
- `salary` 包含一个`Double`值
- `createDate` 包含一个`DateTime`值
- `phones` 包含一个`String`数组

LiteDB存储在集合中存储文档。一个集合是一组相关的文档，它们有一组共享的索引。集合类似于关系数据库的一个表。

### BSON

LiteDB使用BSON（二进制JSON）数据格式存储文档。BSON是包含有额外类型信息的一个JSON二进制表示法。在文档中，字段值可以是任意BSON数据类型，包括其他的文档，数组和文档数组。BSON是一种简单、快速地将文档序列化为二进制的方式。

LiteDB使用[BSON 数据类型](http://bsonspec.org/spec.html)的一个子集。如下为所有支持的LiteDB BSON数据类型和.NET对应关系。

|BSON Type |.NET type                                                   |
|----------|------------------------------------------------------------|
|MinValue  |-                                                           |
|Null      |Any .NET object with `null` value                           |
|Int32     |`System.Int32`                                              |
|Int64     |`System.Int64`                                              |
|Double    |`System.Double`                                             |
|Decimal   |`System.Decimal`                                            |
|String    |`System.String`                                             |
|Document  |`System.Collection.Generic.Dictionary<string, BsonValue>`   |
|Array     |`System.Collection.Generic.List<BsonValue>`                 |
|Binary    |`System.Byte[]`                                             |
|ObjectId  |`LiteDB.ObjectId`                                           |
|Guid      |`System.Guid`                                               |
|Boolean   |`System.Boolean`                                            |
|DateTime  |`System.DateTime`                                           |
|MaxValue  |-                                                           |

### JSON Extended

要序列化一个文档为JSON，LiteDB使用一个扩展版本的JSON，这样不会丢失任何不存在于JSON中的BSON类型信息。扩展数据类型被表示为一个嵌入文档，用`$`作为初始键，并且值是字符串。

|BSON 数据类型 |JSON 表示法                                           |描述                               |
|--------------|------------------------------------------------------|-----------------------------------|
|ObjectId      |`{ "$oid": "507f1f55bcf96cd799438110" }`              |12 bytes in hex format             |
|Date          |`{ "$date": "2015-01-01T00:00:00Z" }`                 |UTC and ISO-8601 format            |
|Guid          |`{ "$guid": "ebe8f677-9f27-4303-8699-5081651beb11" }` |                                   |
|Binary        |`{ "$binary": "VHlwZSgaFc3sdcGFzUpcmUuLi4=" }`        |Byte array in base64 string format |
|Int64         |`{ "$numberLong": "12200000" }`                       |                                   |
|Decimal       |`{ "$numberDecimal": "122.9991" }`                    |                                   |
|MinValue      |`{ "$minValue": "1" }`                                |                                   |
|MaxValue      |`{ "$maxValue": "1" }`                                |                                   |

LiteDB在它的`JsonSerializer`静态类中实现JSON。序列化和反序列化只接受`BsonValue`作为输入/输出。如果你想转换你的对象类型为一个BsonValue，你需要使用一个`BsonMapper`。

```C#
var customer = new Customer { Id = 1, Name = "John Doe" };

var doc = BsonMapper.Global.ToDocument(customer);

var jsonString = JsonSerialize.Serialize(doc, pretty, includeBinaryData);
```

`JsonSerialize`也支持`TextReader` 和 `TextWriter`从一个文件或`Stream`中直接读/写。

### ObjectId

`ObjectId`是一个12字节BSON类型：

- `Timestamp`: 表示Unix时间以来的秒数的值
- `Machine`: 机器标识 (3字节) 
- `Pid`: 进程标识 (2字节)
- `Increment`: 一个计数器，开始于一个随机值(3字节)

在LiteDB中，文档被存储在集合中时，需要一个唯一的`_id`字段来作为主键。因为`ObjectIds`很小，唯一性很高，并且生成很快，LiteDB使用`ObjectIds`作为`_id`字段未指定时的默认值

不同于Guid数据类型，ObjectIds是有序的，因此更适合于索引。ObjectIds使用十六进制数字的字符串表示。

```C#

var id = ObjectId.NewObjectId();

// 你可以从一个ObjectId获取创建时间
var date = id.CreationTime;

// ObjectId 用十六进制值表示
Debug.WriteLine(id);
"507h096e210a18719ea877a2"

// 基于十六进制表示法创建一个实例
var nid = new ObjectId("507h096e210a18719ea877a2");
```