`BsonDocument`类是LiteDB文档的一个实现。`BsonDocument`内部在一个`Dictionary<string, BsonValue>`中存储键值对。


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

- Keys 只包含字符，数字或`_`和`-`
- Keys 是区分大小写的
- 不允许重复键
- LiteDB保持原始键顺序，包括映射的类。仅有的例外是`_id`字段，它会一直是第一个字段。 

关于文档字段 **values**:

- Values 可以是任意BSON数据类型: Null, Int32, Int64, Double, String, Embedded Document, Array, Binary, ObjectId, Guid, Boolean, DateTime, MinValue, MaxValue
- 当一个字段被索引，它的值在BSON序列化后必须小于512字节。
- 无索引的值本身没有大小限制，但整个文档BSON序列化限制为1Mb。这个大小包括BSON使用的所有额外字节。
- `_id` 字段不能是: `Null`, `MinValue` or `MaxValue`
- `_id` 是唯一索引字段，因此值必须小于512字节

关于.NET类

- `BsonValue` 
    - 这个类可以保存任何BSON数据类型，包括null，数组或文档。
    - 对所有支持的.NET数据类型有一个隐式构造器
    - 值不会变化
    - `RawValue` 属性返回内部.NET对象实例
- `BsonArray` 
    - 支持 `IEnumerable<BsonValue>`
    - 每一个数组项可以有不同的BSON类型对象
- `BsonDocument`
    - 缺失字段总是返回`BsonValue.Null`值
    - `Set` 和 `Get` 方法可以配合点分表示法（dotted notation）用于访问/创建内部文档

```C#
// 测试BSON值数据类型
if(customer["Name"].IsString) { ... }

// 辅助获取.NET类型
string str = customer["Name"].AsString;
```

要使用其他.NET数据类型，你需要一个自定义`BsonMapper`类。
