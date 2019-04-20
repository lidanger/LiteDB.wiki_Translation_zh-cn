LiteDB 将数据存储在文档中。这是一种类似 JSON 的键值对，一种无模式的数据结构，每一个文档同时包含数据和结构。

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

- `_id` 包含文档主键 - 在集合内是唯一的
- `name` 包含一个嵌入式文档，其中有 `first` 和 `last` 两个字段
- `age` 包含一个 `Int32` 值
- `salary` 包含一个 `Double` 值
- `createDate` 包含一个 `DateTime` 值
- `phones` 包含一个 `String` 数组

LiteDB 将文档存储在集合中。集合是一组相关的文档，这些文档之间共享同一组索引。可以将集合看作关系数据库的一个表。

### BSON

LiteDB 使用 BSON (二进制 JSON) 数据格式存储文档。BSON 是 JSON 的二进制表示形式，但它比 JSON 多了一些额外的类型信息。在文档中，字段值可以是任何 BSON 数据类型，包括其他文档、数组、文档数组。BSON 是一种简单、快速地将文档序列化为二进制数据的方法。

LiteDB 只使用了 [BSON 数据类型](http://bsonspec.org/spec.html) 的一个子集。以下是 LiteDB 支持的所有 BSON 数据类型以及它们与 .NET 数据类型的对应。

|BSON 类型 |.NET 类型                                                   |
|----------|------------------------------------------------------------|
|MinValue  |-                                                           |
|Null      |任何具有 `null` 值的 .NET 对象                               |
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

### DateTime 注意事项
由于在 BSON 标准中 `DateTime` 使用 **ms** 精度而**没有存储** `DateTimeKind`。<br>
所以在存储对象时，以及检索文档并转换回本地对象前，所有 `DateTime` 值都会被转换为 UTC 时间。

### JSON 扩展

为了将文档序列化为 JSON，LiteDB 使用了 JSON 的一个扩展版本，这样就不会丢失那些在 JSON 中不存在的 BSON 类型信息。扩展数据类型使用嵌入式文档来表示，它们的键以 `$` 开头，而且值是字符串。

|BSON 数据类型  |JSON 表示法                                           |描述                               |
|--------------|------------------------------------------------------|-----------------------------------|
|ObjectId      |`{ "$oid": "507f1f55bcf96cd799438110" }`              |12 字节，十六进制格式                |
|Date          |`{ "$date": "2015-01-01T00:00:00Z" }`                 |UTC 和 ISO-8601 格式                |
|Guid          |`{ "$guid": "ebe8f677-9f27-4303-8699-5081651beb11" }` |                                   |
|Binary        |`{ "$binary": "VHlwZSgaFc3sdcGFzUpcmUuLi4=" }`        |以 base64 字符串格式存储的字节数组     |
|Int64         |`{ "$numberLong": "12200000" }`                       |                                   |
|Decimal       |`{ "$numberDecimal": "122.9991" }`                    |                                   |
|MinValue      |`{ "$minValue": "1" }`                                |                                   |
|MaxValue      |`{ "$maxValue": "1" }`                                |                                   |

LiteDB 在 `JsonSerializer` 静态类中实现 JSON 序列化。序列化和反序列化都只接受 `BsonValue` 作为输入/输出，如果要将自定义对象类型转换为 BsonValue 值，那你需要使用 `BsonMapper`。

```C#
var customer = new Customer { Id = 1, Name = "John Doe" };

var doc = BsonMapper.Global.ToDocument(customer);

var jsonString = JsonSerialize.Serialize(doc, pretty, includeBinaryData);
```

`JsonSerialize` 支持通过 `TextReader` 和 `TextWriter` 对文件或 `Stream` 进行读/写 。

### ObjectId

`ObjectId` 是一种 12 字节的 BSON 类型：

- `Timestamp`: 表示 Unix 时间的秒数 (4 字节)
- `Machine`: 机器标识 (3 字节)
- `Pid`: 处理器标识 (2 字节)
- `Increment`: 计数器，开始于一个随机值 (3 字节)

在 LiteDB 中，文档存储在集合里，需要一个唯一的 `_id` 字段来扮演主键角色。因为 `ObjectIds` 既小巧，又能在很大程度上保证唯一，生成起来又很快，所以，如果 `_id` 未指定的话，LiteDB 使用一个 `ObjectId` 作为 `_id` 字段的默认值。

与 Guid 类型不同，ObjectId 是有序的，因此更适合用于索引。ObjectId 使用的是十六进制数的字符串表示形式。

```C#

var id = ObjectId.NewObjectId();

// 你可以从 ObjectId 获取创建时间
var date = id.CreationTime;

// ObjectId 用16进制值表示
Debug.WriteLine(id);
"507h096e210a18719ea877a2"

// 基于16进制表示法创建一个实例
var nid = new ObjectId("507h096e210a18719ea877a2");
```