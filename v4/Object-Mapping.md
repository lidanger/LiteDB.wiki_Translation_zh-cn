LiteDB 支持通过 POCO 类创建强类型文档。当你从 `LiteDatabase.GetCollection<T>` 获取 `LiteCollection` 时， `<T>` 就是你的文档类型。如果 `<T>` 不是 `BsonDocument`，LiteDB 内部会将你的类映射为 `BsonDocument`。为了做到这些，LiteDB 使用了`BsonMapper` 类：

```C#
// 简单的强类型文档
public class Customer
{
    public ObjectId CustomerId { get; set; }
    public string Name { get; set; }
    public DateTime CreateDate { get; set; }
    public List<Phone> Phones { get; set; }
    public bool IsActive { get; set; }
}

var typedCustomerCollection = db.GetCollection<Customer>("customer");

var schemelessCollection = db.GetCollection("customer"); // <T> 是 BsonDocument
```

### 映射约定

`BsonMapper.ToDocument()` 自动转换类（实例）的每个**属性**为文档字段，并遵循这些约定：

- 类必须是 _**公有的，而且要有一个公有的无参数构造方法**_
- 属性必须是公有的
- 属性可以是只读的或支持读/写的
- 类必须有一个 `Id` 属性，`<ClassName>Id` 属性，或任何使用 `[BsonId]` 特性修饰或用 fluent api Id 函数映射的属性.
- 属性可以使用 `[BsonIgnore]` 修饰，确保不被映射到文档字段
- 属性可以使用 `[BsonField]` 修饰，定制文档字段的名称
- 不允许循环引用
- 最大深度为 20 层内部（嵌套）类
- 类字段不会被转换到文档
- 可以将 `BsonMapper` 的全局实例 (`BsonMapper.Global`) 或自定义实例传递给 `LiteDatabase` 的构造函数，可以单独保存此实例，以避免每次使用数据库时都要重新创建映射。

除基础 BSON 类型外，`BsonMapper` 还可以将其他 .NET 类型映射为 BSON 数据类型：

|.NET 类型                          |BSON 类型     |
|-----------------------------------|--------------|
|`Int16`, `UInt16`, `Byte`, `SByte` |Int32         |
|`UInt32` , `UInt64`                |Int64         |
|`Single`                           |Double        |
|`Char`, `Enum`                     |String        |
|`IList<T>`                         |Array         |
|`T[]`                              |Array         |
|`NameValueCollection`              |Document      |
|`IDictionary<K,T>`                 |Document      |
|任何其他 .NET type                 |Document      |

- `Nullable<T>` 是允许的。如果值是 `null`，那么 BSON 类型就是 Null，否则映射器会使用 `T?`。
- 对于 `IDictionary<K, T>`，`K` 键必须是 `String` 或简单类型 (可以通过 `Convert.ToString(..)` 转换为字符串)。

#### 注册自定义类型

可以使用 `RegisterType<T>` 实例方法注册自己的映射函数。在注册时，你需要提供序列化和反序列化两个函数。

```C#
BsonMapper.Global.RegisterType<Uri>
(
    serialize: (uri) => uri.AbsoluteUri,
    deserialize: (bson) => new Uri(bson.AsString)
);
```

- `serialize` 函数传递 `<T>` 实例为输入参数，并期待返回 `BsonValue`
- `deserialize` 函数传递 `BsonValue` 对象为输入参数，并期待返回 `<T>` 值
- `RegisterType` 通过 `BsonDocument` 或 `BsonArray` 支持复杂对象

#### 映射选项

`BsonMapper` 类设置：

|名称                   |默认值  |描述                                                        |
|-----------------------|--------|-----------------------------------------------------------|
|`SerializeNullValues`  |false   |如果值是 `null`，序列化字段                                  |
|`TrimWhitespace`       |true    |在映射到文档前，修剪字符串属性                               |
|`EmptyStringToNull`    |true    |空字符串转换为 `null`                                       |
|`ResolvePropertyName`  |(s) => s|映射属性名称为字段名称的函数                                 |

`BsonMapper` 提供2个预定义的函数来解析属性名称：`UseCamelCase()` 和 `UseLowerCaseDelimiter('_')`。

```C#
BsonMapper.Global.UseLowerCaseDelimiter('_');

public class Customer
{
    public int CustomerId { get; set; }

    public string FirstName { get; set; }

    [BsonField("customerLastName")]
    public string LastName { get; set; }
}

var doc = BsonMapper.Global.ToDocument(new Customer { FirstName = "John", LastName = "Doe" });

var id = doc["_id"].AsInt;
var john = doc["first_name"].AsString;
var doe = doc["customerLastName"].AsString;
```    

### AutoId

在 v4 中，AutoId 被上升到引擎级别 (BsonDocument)。有 4 种内置的 auto-id 实现：

- `ObjectId`: `ObjectId.NewObjectId()`
- `Guid`: `Guid.NewGuid()` 方法
- `Int32/Int64`: 新集合序列
- `DateTime`: `DateTime.Now`

AutoId 只在插入文档 `_id` 缺失的情况下使用。在强类型文档中，`BsonMapper` 从 "empty" (像 `Int` 中的 `0` 或 `Guid` 中的 `Guid.Empty`) 值中移除了 `_id` 字段。

### Fluent 映射

LiteDB 提供了完整的 fluent API，不用使用特性就可以创建自定义映射。要使用它们，请保证你的领域类没有外部引用。

Fluent API 使用 `EntityBuilder` 来为你的类添加自定义映射。

```C#
var mapper = BsonMapper.Global;

mapper.Entity<MyEntity>()
    .Id(x => x.MyCustomKey) // 设置你的文档 ID
    .Ignore(x => x.DoNotSerializeThis) // 忽略此属性 (不存储)
    .Field(x => x.CustomerName, "cust_name"); // 重命名文档字段
```