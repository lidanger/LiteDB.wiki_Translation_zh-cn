`BsonDocument` 类是 LiteDB 的文档实现。在 `BsonDocument` 内部， 键-值对被存储在一个 `Dictionary<string, BsonValue>` 中。

```C#
var customer = new BsonDocument();
customer["_id"] = ObjectId.NewObjectId();
customer["Name"] = "John Doe";
customer["CreateDate"] = DateTime.Now;
customer["Phones"] = new BsonArray { "8000-0000", "9000-000" };
customer["IsActive"] = true;
customer["IsAdmin"] = new BsonValue(true);
customer.Set("Address.Street", "Av. Protasio Alves, 1331");
```

关于文档字段 **keys**:

- 只能包含字母、数字或 `_`、`-`
- 大小写敏感
- 不允许重复
- LiteDB 保持（对象字段的）原始键序，包括映射的类。只有一个例外，`_id` 字段总是第一个字段。 

关于文档字段 **values**:

- 可以是任何 BSON 值类型：Null, Int32, Int64, Double, String, 嵌入式文档, 数组, Binary, ObjectId, Guid, Boolean, DateTime, MinValue, MaxValue
- 如果要在字段上建立索引，必须保证字段值序列化为 BSON 后小于 512 字节。
- 非索引值本身没有大小限制，但是整个文档序列化为 BSON 后必须小于 1 Mb，这个大小包含所有被 BSON 使用的额外字节。
- `_id` 字段不能是：`Null`, `MinValue` 或 `MaxValue`
- `_id` 是唯一索引字段，因此值必须小于 512 字节

关于 .NET 类

- `BsonValue` 
    - 此类可以保存任何 BSON 数据类型，包括 null、数组和文档。
    - 含有针对所有支持的 .NET 数据类型的隐式构造函数（类型转换构造函数）
    - 值无法修改
    - `RawValue` 属性返回内部 .NET 对象实例
- `BsonArray` 
    - 支持通过 `IEnumerable<BsonValue>` 创建
    - 每个数组项可以是不同的 BSON 类型对象
- `BsonDocument`
    - 缺失的字段总是返回 `BsonValue.Null` 值
    - `Set` 和 `Get` 方法可以通过点分表示法（xx.yy）来获取/创建内部文档

```C#
// 测试 BSON 值的数据类型
if(customer["Name"].IsString) { ... }

// 帮助获取 .NET 类型
string str = customer["Name"].AsString;
```

要使用其他 .NET 类型，你需要自定义一个 `BsonMapper` 类。