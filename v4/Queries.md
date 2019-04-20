查询按三种方式过滤一个集合中的文档：

- 基于索引的搜索 (最好的选项)。参考[索引](Indexes)
- BsonDocument 上的全扫描 (更慢但更强大)
- LINQ to object (更慢但方便)

### 查询实现

`Query` 是一个生成查询条件的静态类，它的每个方法都代表一个不同的可以用于查询文档的条件操作。

- **`Query.All`** - 返回所有文档。可以指定一个索引字段，按升序或降序来读取。
- **`Query.EQ`** - 查找等于 (==) 值的文档。
- **`Query.LT/LTE`** - 查找小于 (<) 或小于等于 (<=) 值的文档。
- **`Query.GT/GTE`** - 查找大于 (>) 或大于等于 (>=) 值的文档。
- **`Query.Between`** - 查找在 start/end 值之间的文档。
- **`Query.In`** - 查找等于列出的值的文档。
- **`Query.Not`** - 查找不等于 (!=) 值的文档。（或者对其他查询取反）
- **`Query.StartsWith`** - 查找字符串值开始于值的文档。只对字符串数据类型有效。
- **`Query.Contains`** - 查找字符串值包含值的文档。只对字符串数据类型有效。此查询执行索引搜索，只是索引扫描 (对很多文档来说很慢)。
- **`Query.Where`** - 基于 `Func<BsonValue, bool>` 谓词查找文档，这里，`BsonValue` 索引键。这是一个基于全索引扫描的查询。
- **`Query.And`** - 在两个查询结果间取交集。 
- **`Query.Or`** - 在两个查询结果间取并集。 

```C#
var results = col.Find(Query.EQ("Name", "John Doe"));

var results = col.Find(Query.GTE("Age", 25));

var results = col.Find(Query.And(
    Query.EQ("FirstName", "John"), Query.EQ("LastName", "Doe")
));

var results = col.Find(Query.StartsWith("Name", "Jo"));

// 使用多键索引查询 (这里 products 是一个嵌入的文档数据)
var results = col.Find(Query.GT("Products.Price", 100))

// 在 Name 索引的每个键上执行 Func 
var results = col.Find(Query.Where("Name", name => name.AsString.Length > 20));

// 获得集合最后添加的100个对象
var results = collection.Find(Query.All(Query.Descending)), limit: 100);

// 查找20到30之间年龄最大的100个人
var results = col.Find(Query.And(Query.All("Age", Query.Descending), Query.Between("Age", 20, 30)), limit: 100);
```

在所有的查询中：

- 在索引搜索中，**Field** 必须一个索引名称或文档的字段。
- 如果没有使用索引，**Field** 可以是`路径`或一个`表达式`
- **Field** 名称在左边，**Value** (或 values) 在右边
- 查询在将 `BsonDocument` 映射为你的类对象之前执行，所以你可能需要使用 `BsonDocument` 字段名称和 BSON 类型值。如果你使用了自定义的 `ResolvePropertyName` 或 `[BsonField]` 特性，你必须使用文档字段名称，而不是类型的属性名称。参考[对象映射](Object-Mapping)

### Find(), FindById(), FindOne() 和 FindAll()

集合以4种方式返回文档：

- **`FindAll`**: 返回集合的所有文档
- **`FindOne`**: 返回 `Find()` 结果的 `FirstOrDefault`
- **`FindById`**: 返回使用主键`_id`索引 `Find()` 的结果的 `SingleOrDefault`。
- **`Find`**: 返回使用 `Query` 构建器或 LINQ 表达式查询的集合文档。

`Find()` 支持 `Skip` 和 `Limit` 参数。这些操作在索引级别执行，因此比 LINQ to Objects 更高效。

`Find()` 方法返回一个文档的 `IEnumerable` 列表。如果你想做更复杂的过滤，如，值作为表达式，排序或转换的结果，你可以使用 LINQ to Objects。

返回 `IEnumerable` 的时候，你的代码仍然在连接着数据文件。只有当你用完查询到的所有数据，数据文件才会断开。

```C#
col.EnsureIndex(x => x.Name);

var result = col
    .Find(Query.EQ("Name", "John Doe")) // 此过滤器在 LiteDB 中使用索引执行
    .Where(x => x.CreationDate >= x.DueDate.AddDays(-5)) // 此过滤器通过 LINQ to Object 执行
    .OrderBy(x => x.Age)
    .Select(x => new 
    { 
        FullName = x.FirstName + " " + x.LastName, 
        DueDays = x.DueDate - x.CreationDate 
    }); // 转换
```

### Count() 和 Exists()

这两个方法非常有用，因为通过它们你可以不执行反序列化就能统计文档 (或者检查文档是否存在)。

```C#
// 这种方式更高效
var count = collection.Count(Query.EQ("Name", "John Doe"));

// 比使用 Find + Count
var count = collection.Find(Query.EQ("Name", "John Doe")).Count();
```

- 在第一个统计中，LiteDB 使用索引来搜索，并统计索引中出现 "Name = John" 的数目，而没有反序列化和映射文档。
- 如果 `Name` 字段没有索引，LiteDB 会反序列化文档，但不会运行映射器。仍然比 `Find().Count()` 快。
- 同样的道理适用于使用 `Exists()`，它比使用 `Count() >= 1` 更高效。Count 需要访问所有匹配的结果，而 `Exists()` 在第一个匹配后就停止 (类似于 LINQ 的 `Any` 扩展方法)。

### Min() 和 Max()

LiteDB 使用跳跃列表实现索引 (参考[索引](Indexes))。集合提供了 `Min` 和 `Max` 索引名称，实现如下：

- **`Min`** - 读取头索引节点 (MinValue BSON 数据类型) 并移动到下一个节点，此节点是索引中最低的一个值。如果索引为空，返回 MinValue。最低值并不是第一个值!
- **`Max`** - 读取尾索引节点 (MaxValue BSON 数据类型) 并移动到下一个节点，此节点是索引中最高的一个值。如果索引为空，返回 MaxValue。最高值并不是最后一个值!

Min/Max 需要在字段上先创建一个索引。

### LINQ 表达式

一些 LiteDB 方法支持谓词，这可以方便你简单地查询强类型文档。如果你用的是 `BsonDocument`，就要使用经典的 `Query` 类方法。 

```C#
col.Find(x => x.Name == "John Doe")
// Query.EQ("Name", "John Doe")

col.Find(x => x.Age > 30)
// Query.GT("Age", 30)

col.Find(x => x.Name.StartsWith("John") && x.Age > 30)
// Query.And(Query.StartsWith("Name", "John"), Query.GT("Age", 30))

// 这里 PhoneNumbers 是 string[]
col.Find(x => x.PhoneNumbers.Contains("555-1234"))
// Query.EQ("PhoneNumbers", "555-1234")

// 在 phone 数组里的 Number 上创建索引
col.EnsureIndex(x => x.Phones[0].Number); // 忽略 0 索引：它只是一个获取子项目的语法
// db.EnsureIndex("col", "Phones.Number", false, "$.Phones[*].Number)

col.Find(x => x.Phones.Select(z => z.Number == "555-1234")) // 另外一种获取子项目的方式
// Query.EQ("Phones.Numbers", "555-1234")

col.Find(x => !(x.Age > 30))
// Query.Not(Query.GT("Age", 30))
```

- LINQ 实现包括：`==, !=, >, >=, <, <=, StartsWith, Contains (string and IEnumerable), Equals, &&, ||, ! (not)`
- 属性名称支持内部文档字段：`x => x.Name.Last == "Doe"`
- 在内部，LINQ 表达式被 `QueryVisitor` 转换为 `Query` 实现。