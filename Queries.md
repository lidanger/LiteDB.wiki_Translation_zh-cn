在LiteDB中查询时，必须在所搜索字段上有一个索引才能查找文档。如果字段上没有索引，LiteDB会自动使用默认选项创建一个新索引。要做查询，你可以使用：

- `Query`静态帮助类
- 使用 LINQ 表达式

### 查询实现

`Query`是一个静态类，可以创建查询条件。每一个方法都是一个不同的条件操作，可以用于查询文档。

- **`Query.All`** - 返回所有文档。可以指定一个索引字段来按升序或降序读取。
- **`Query.EQ`** - 查找等于(==)值的文档。
- **`Query.LT/LTE`** - 查找小于(<)或小于等于(<=)值的文档。
- **`Query.GT/GTE`** - 查找大于(>)或大于等于(>=)值的文档。
- **`Query.Between`** - 查找介于start/end值的文档。
- **`Query.In`** - 查找等于列表中的值的文档。
- **`Query.Not`** - 查找不等于(!=)值的文档。
- **`Query.StartsWith`** - 查找字符串以值开始的文档。只对字符串数据类型有效。
- **`Query.Contains`** - 查找字符串包含值的文档。只对字符串数据类型有效。这项查询做索引搜索，只扫描索引（only index scan）(对很多文档来说很慢)。
- **`Query.Where`** - 基于一个`Func<BsonValue, bool>`谓词查找，这里`BsonValue`是索引中的一项。它是一个基于全索引扫描的查询。
- **`Query.And`** - 在两个查询结果间取交集。 
- **`Query.Or`** - 在两个查询结果间取并集。 

```C#
var results = col.Find(Query.EQ("Name", "John Doe"));

var results = col.Find(Query.GTE("Age", 25));

var results = col.Find(Query.And(
    Query.EQ("FirstName", "John"), Query.EQ("LastName", "Doe")
));

var results = col.Find(Query.StartsWith("Name", "Jo"));

// 使用多项（multikey）索引(这里products是一个嵌入式文档数组)
var results = col.Find(Query.GT("Products.Price", 100))

// 在Name索引的每一项上执行Func
var results = col.Find(Query.Where("Name", name => name.AsString.Length > 20));

// 获取集合中最后添加的100个对象
var results = collection.Find(Query.All(Query.Descending)), limit: 100);

// 查找年纪在20到30之间年纪最大的100个人
var results = col.Find(Query.And(Query.All("Age", Query.Descending), Query.Between("Age", 20, 30)), limit: 100);
```

在所有查询中：

- **Field** 上必须有一个索引 (使用`EnsureIndex`, `[BsonIndex]` 特性或fluent mapper创建。参考[[Indexes]])。如果没有索引，LiteDB会快速创建一个新索引。
- **Field** 名称在左侧，**Value** (或多个值)在右侧
- 查询在`BsonDocument`类中执行，不需要先映射到你的对象。你需要使用`BsonDocument`字段名称和BSON类型值。如果使用了一个自定义`ResolvePropertyName`或`[BsonField]`特性，你必须使用文档的字段名称，而不是类型的属性名称。参考[对象映射](Object-Mapping).
- LiteDB不支持表达式值，如`CreationDate == DueDate`。

### Find(), FindById(), FindOne() 和 FindAll()

集合以4种方式返回文档：

- **`FindAll`**:返回集合中的所有文档
- **`FindOne`**: 返回`Find()`结果中的`FirstOrDefault`
- **`FindById`**: 使用主键`_id`索引返回`Find()`结果中的`SingleOrDefault`。
- **`Find`**: 使用`Query`构建器或LINQ返回集合中的文档。

`Find()`支持`Skip`和`Limit`参数。这些操作可用于索引级别，因此它比LINQ to Objects更高效。

`Find()`方法返回文档的一个`IEnumerable`。如果你想做更复杂的过滤，如值为表达式，排序或转换结果等，你可以使用LINQ to Objects。

返回一个`IEnumerable`后，你的代码仍然与数据文件保持连接。只有当你消费完所有数据后，才会断开数据文件。

```C#
col.EnsureIndex(x => x.Name);

var result = col
    .Find(Query.EQ("Name", "John Doe")) // 在LiteDB中这个过滤器使用索引执行
    .Where(x => x.CreationDate >= x.DueDate.AddDays(-5)) // 这个过滤器使用LINQ to Object执行
    .OrderBy(x => x.Age)
    .Select(x => new 
    { 
        FullName = x.FirstName + " " + x.LastName, 
        DueDays = x.DueDate - x.CreationDate 
    }); // 转换
```

### Count() 和 Exists()

这两个方法很有用，因为你可以在不反序列化文档的情况下统计文档（或者检查一个文档是否存在）。

```C#
// 这种方式更高效
var count = collection.Count(Query.EQ("Name", "John Doe"));

// Than use Find + Count
var count = collection.Find(Query.EQ("Name", "John Doe")).Count();
```

- 在第一个统计中，LiteDB使用索引来搜索，并统计索引中"Name = John"出现的次数，不用反序列化和映射文档。
- 如果`Name`字段没有索引，LiteDB将反序列化文档，但不进行映射。仍然比`Find().Count()`快
- 同样的想法适用于使用`Exists()`的时候，它会比`Count() >= 1`更快。Count需要访问所有匹配的结果，而`Exists()`在找到第一个匹配时就会停止(类似于LINQ的`Any` 扩展方法)。

### Min() 和 Max()

LiteDB使用一个跳跃表实现索引(参考[索引](Indexes))。集合提供`Min`和`Max`索引值。实现是：

- **`Min`** - 读取头部索引节点(MinValue BSON 数据类型)并移动到下一个节点。这个节点是索引中的最小值。如果索引是空的，返回MinValue。最小值不是第一个值!
- **`Max`** - 读取尾部索引节点(MaxValue BSON 数据类型)并移动到前一个节点。这个节点是索引中的最大值。如果索引是空的，返回MaxValue。最大值不是最后一个值!

`Max`操作可用于`Int32`的`AutoId`。这个值获取起来很快，因为只需要读取两个索引节点(尾部+前一个)。

### LINQ 表达式

有些LiteDB方法支持谓词，这将允许你更简单地查询强类型文档。如果你使用的是`BsonDocument`，你需要使用经典的`Query`类方法。

```C#
col.Find(x => x.Name == "John Doe")
// Query.EQ("Name", "John Doe")

col.Find(x => x.Age > 30)
// Query.GT("Age", 30)

col.Find(x => x.Name.StartsWith("John") && x.Age > 30)
// Query.And(Query.StartsWith("Name", "John"), Query.GT("Age", 30))

// 这里PhoneNumbers是string[]
col.Find(x => x.PhoneNumbers.Contains("555-1234"))
// Query.EQ("PhoneNumbers", "555-1234")

col.Find(x => !(x.Age > 30))
// Query.Not(Query.GT("Age", 30))
```

- LINQ 实现: `==, !=, >, >=, <, <=, StartsWith, Contains (string and IEnumerable), Equals, &&, ||, ! (not)`
- 属性名支持内部文档字段: `x => x.Name.Last == "Doe"`
- 在后台，LINQ表达式被`QueryVisitor`类转换为`Query`实现。
