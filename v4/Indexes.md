LiteDB 使用文档字段索引来改善搜索性能。每个索引存储着指定字段的值，并按字段的值 (和类型) 排序。如果没有索引，LiteDB 必须使用全文档扫描来执行一个查询。全文档扫描是毫无效率的，因为 LiteDB 必须反序列化所有文档并一个一个测试（查询条件）。

### 索引实现

LiteDB 使用了一个简单的索引方案：**跳跃列表**。跳跃列表是有序的双向链表，链接可以达到 32 级。跳跃列表非常容易实现 (只要 15 行代码) 并且统计均衡，测试结果很不错：插入和查找结果平均复杂度 O(ln n) = 1 百万文档 = 13 步。如果你想了解更多关于跳跃列表的信息，请查看[这个牛X的视频](https://www.youtube.com/watch?v=kBwUoWpeH_Q)。

文档是无模式的，即使它们在同一个集合里。因此，你可以在任何一个字段上创建索引，这个字段可以在一个文档中是这种类型，在另外一个文档中是另一种类型。当同一个字段数据类型不同时，LiteDB 只比较类型。每个类型都有一个次序：

|BSON 类型                     |次序  |
|------------------------------|-----|
|MinValue                      |1    |
|Null                          |2    |
|Int32, Int64, Double, Decimal |3    |
|String                        |4    |
|Document                      |5    |
|Array                         |6    |
|Binary                        |7    |
|ObjectId                      |8    |
|Guid                          |9    |
|Boolean                       |10   |
|DateTime                      |11   |
|MaxValue                      |12   |

- 数字 (Int32, Int64, Double 或 Decimal) 都是同样的次序。如果你在同一个文档字段上混合使用这些数字类型，当比较时，LiteDB 会将它们转换为 `Decimal`。

### EnsureIndex()

索引通过 `EnsureIndex` 创建。这个实例方法确认一个索引：如果不存在就创建索引，如果已存在就什么也不做。在 v4 中，改变定义不会再重新创建索引。要想重新创建一个索引，你首先要删除此索引，然后再执行 `EnsureIndex`。

索引用文档字段名称标识。LiteDB 只支持一个索引一个字段，但是这个字段可以是任何 BSON 类型，甚至是一个嵌入的文档。

```JS
{
    _id: 1,
    Address:
    {
        Street: "Av. Protasio Alves, 1331",
        City: "Porto Alegre",
        Country: "Brazil"
    }
}
```

- 你可以使用 `EnsureIndex("Address")` 为所有 `Address` 嵌入文档创建一个索引
- 或者使用点分表示法 `EnsureIndex("Address.Street")` 在 `Street` 上创建一个索引
- 索引按 `BsonDocument` 字段执行。如果你在属性上使用了自定义的 `ResolvePropertyName` 或 `[BsonField]` 特性，那就必须通过文档字段名称而不是属性名称来使用索引。参见[对象映射](Object-Mapping).
- 你可以通过 lambda 表达式在一个强类型集合中定义索引字段：`EnsureIndex(x => x.Name)`

### 多键值索引

当你在一个数组类型字段上创建索引时，所有的数组值都被包含在索引键中，这样可以搜索任何一个值。

```C#
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
}

var customers = db.GetCollection<Customer>("customers");

customers.Insert(new Customer { Name = "John", Phones = new string[] { "1", "2", "5" });
customers.Insert(new Customer { Name = "Doe", Phones = new string[] { "1", "8" });

customers.EnsureIndex(x => x.Phones, "$.Phones[*]");

var result = customers.Query(x => x.Phones.Contains("1")); // returns both documents
```

### 表达式

在 v4 中，通过支持多键值执行可以创建一个基于表达式的索引。这样，你可以索引那些不直接是字段值的任何重要信息，像这样：

- `db.EnsureIndex("customer", "Name", false, "LOWER($.Name)")`
- `db.EnsureIndex("customer", "Total", false, "SUM($.Items[*].Price)")`
- `db.EnsureIndex("customer", "CheapBooks", false, "LOWER($.Books[@.Price < 20].Title)")`

查看[表达式](Expressions) 获取关于表达式的更多细节。

### v4 中的变化

- 不再有自动的索引创建，你必须在数据库初始化时运行 `EnsureIndex`。
- 如果你尝试执行没有索引的查询，查询会通过全搜索进行
- 如果你使用没有解析到 `Query` 的 LINQ 表达式来查询，查询引擎会在映射到对象后才执行查询
- 如果你的查询有一个 `And` 操作，查询引擎只会在其中一个（子查询）中使用索引 (如果存在)，而其他会使用全扫描。这会优化结果，防止多索引查询。总是尝试使用左侧那个。

```C#
col.EnsureIndex(x => x.Name);
col.EnsureIndex(x= > x.Age);

var r = col.Find(x => x.Name == "John" && x.Age > 20 && x.Phones.Length > 1);
```

在这个例子中，LiteDB 使用 `Name` 索引来获得第一个结果。对于 `Age > 20` 来说，需要对所有 `Name == 'John'` 的文档使用全扫描。然后，同理于 `Phones.Length > 1`。

###  限制

- 索引值必须少于 512 字节 (序列化为 BSON 后)
- 每个集合最多 16 个索引 - 包括 `_id` 主键