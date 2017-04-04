LiteDB对强类型文档支持POCO类。当你从`LiteDatabase.GetCollection<T>`获得一个`LiteCollection`实例时，`<T>`将是你的文档类型。如果`<T>`不是一个`BsonDocument`，LiteDB在内部将你的类映射为`BsonDocument`。要做到这个，LiteDB使用了`BsonMapper`类：

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

var schemelessCollection = db.GetCollection("customer"); // <T> is BsonDocument
```

### 映射器约定

`BsonMapper.ToDocument()`在将一个类的每个属性自动转换为一个文档字段时遵循这些约定：

- 类必须是_**public，有一个公共的无参数构造函数**_
- 属性必须是公共的
- 属性可以是只读或读/写
- 类必须有一个`Id`属性、`<ClassName>Id`属性或任何使用`[BsonId]`特性的属性，或使用fluent api映射。
- 一个属性可以使用`[BsonIgnore]`来装饰以保证不映射到文档字段
- 一个属性可以使用`[BsonField]`来装饰以定制文档字段的名称
- 不允许循环引用
- 内部类最大深度20
- 类字段不会转换到文档
- 你可以使用`BsonMapper`的全局实例(`BsonMapper.Global`)或在`LiteDatabase`的构造函数中传递给一个自定义（BsonMapper）实例。可以在一个单独的地方保存此实例，以防止你每次使用数据库时重新创建所有映射。

除了基本的BSON类型，`BsonMapper`也映射其他.NET类型到BSON数据库：

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
|任何其他.NET类型                     |Document      |

- `Nullable<T>`是被接受的。如果值是`null`，BSON类型将是Null，否则映射器会使用`T?`。
- 对于`IDictionary<K, T>`，`K`键必须是`String`或简单类型(可以使用`Convert.ToString(..)`转换为String)。 

#### 注册一个自定义类型

你可以使用`RegisterType<T>`实例方法注册你自己的映射函数。注册时，你需要提供序列化和反序列化两个函数。

```C#
BsonMapper.Global.RegisterType<Uri>
(
    serialize: (uri) => uri.AbsoluteUri,
    deserialize: (bson) => new Uri(bson.AsString)
);
```

- `serialize` 函数传递一个`<T>`对象实例为输入参数并期望返回一个`BsonValue`
- `deserialize` 函数传递一个`BsonValue`对象为输入参数并期望返回一个`<T>`值
- `RegisterType` 通过`BsonDocument`或`BsonArray`支持复杂对象

#### 映射选项

`BsonMapper`类设置:

|名称                   |默认值   |描述                                                        |
|-----------------------|--------|-----------------------------------------------------------|
|`SerializeNullValues`  |false   |如果值是`null`，序列化此字段                                  |
|`TrimWhitespace`       |true    |在映射到文档前修剪字符串属性                                   |
|`EmptyStringToNull`    |true    |空字符串转换为`null`                                         |
|`ResolvePropertyName`  |(s) => s|一个函数来映射属性名称到文档字段名                              |

`BsonMapper`提供2个预定义的函数来解析属性名: `UseCamelCase()` 和 `UseLowerCaseDelimiter('_')`.

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

默认情况下，类型化文档在插入时将接收一个自动的**Id**值。LiteDB对这种数据类型支持auto-id： 

|.NET 数据类型    |新值                                           |
|----------------|----------------------------------------------|
|`ObjectId`      |`ObjectId.NewObjectId()`                      |
|`Guid`          |`Guid.NewGuid()`                              |
|`Int32`         |Auto-increment, per collection, starting in 1 |

但是如果你不想使用任何内置方法，或者想为**Id**使用另一个数据类型，你可以自己设置它：

```C#
var customer = new Customer { Id = "john-doe", Name = "John Doe" };
```

你也可以使用`RegisterAutoId<T>`创建你自己的auto-id函数：

```C#
BsonMapper.Global.RegisterAutoId<long>
(
    isEmpty: (value) => value == 0,
    newId: (collection) => DateTime.UtcNow.Ticks
);

public class Calendar
{
    public long CalendarId { get; set; }
}
```

- `isEmpty`返回true来指示这个类型被认为是空的。在这个例子中，0将会是一个空值。
- `newId`返回一个新id值。它将一个`LiteCollection<BsonDocument>`实例作为一个输入参数，这样你可以使用它来确定新Id。在这个例子中，集合被忽略，而返回Ticks值。

### Index 定义
    
`BsonMapper`支持直接使用`[BsonIndex]`特性在一个属性上定义索引。你可以将你的索引为定义唯一的(默认是非唯一的)。当你运行一个查询时，如果无索引，LiteDB也支持自动创建索引。

```C#
public class Customer
{
    public int Id { get; set; }
    
    [BsonIndex]
    public string Name { get; set; }
    
    [BsonIndex(true)]
    public string Email { get; set; }
}
```

不要在你的**Id**主键属性上使用`[BsonIndex]`特性。默认情况下，这个属性已经有一个唯一索引。

### Fluent Mapping

LiteDB提供了一个完整的fluent API。它可以不使用特性创建自定义映射，保持你的领域类没有外部引用。

Fluent API使用`EntityBuilder`来添加自定义映射到你的类。

```C#
var mapper = BsonMapper.Global;

mapper.Entity<MyEntity>()
    .Id(x => x.MyCustomKey) // 设置你的文档ID
    .Ignore(x => x.DoNotSerializeThis) // 忽略这个属性(不做存储)
    .Field(x => x.CustomerName, "cust_name") // 重命名文档字段
    .Index(x => x.Address.Number, true); // 创建唯一索引

```
