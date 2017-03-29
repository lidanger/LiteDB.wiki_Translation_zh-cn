LiteDB通过在文档字段上使用索引改进了搜索性能。每个索引存储一个指定的字段的值，并按字段值（和类型）排序。如果没有索引，LiteDB必须使用一个全文档扫描来执行一个查询。全文档扫描是毫无效率的，因为LiteDB必须反序列化所有文档来一个一个测试。

### Index 实现

LiteDB使用一个简单的索引方案: **Skip Lists**。跳跃列表是排序的双向链表，最多嵌套32级。跳跃列表实现出来超级简单(只要15行代码)， 并且是统计均衡的（statistically balanced）。结果非常好: 插入和查找结果平均 O(ln n) = 1 百万文档 = 13 步。如果你想了解更多关于跳跃列表的信息，参考[this great video](https://www.youtube.com/watch?v=kBwUoWpeH_Q). 

文档是无模式的，即使它们在同一个集合中。因此，你可以在一个字段上创建一个索引，这个字段可以在一个集合中是一种类型，在另一个集合中是另一种类型。当你使用不同类型的字段时，LiteDB只比较类型。每一种类型都有一个顺序:

|BSON 类型            |顺序|
|---------------------|-----|
|MinValue             |1    |
|Null                 |2    |
|Int32, Int64, Double |3    |
|String               |4    |
|Document             |5    |
|Array                |6    |
|Binary               |7    |
|ObjectId             |8    |
|Guid                 |9    |
|Boolean              |10   |
|DateTime             |11   |
|MaxValue             |12   |

- 数字 (Int32, Int64 or Double) 有相同的序号。如果你在同一个文档字段中混合这些数字类型，LiteDB在比较时会把它们转换为Double。

### EnsureIndex()

索引通过`EnsureIndex`创建。这个实例方法建立索引：如果索引不存在则创建索引，如果索引选项不同于已存在的索引则重建索引，或者如果索引没有变化则什么也不做。

另一种确保一个索引创建的方式是使用`[BsonIndex]`类特性。在你第一次查询的时候这个特性将被读取然后运行`EnsureIndex`。

索引用文档字段名标识。LiteDB只支持一个索引一个字段，但这个字段可以是任何BSON类型，甚至是一个嵌入式文档。

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

- 你可以使用`EnsureIndex("Address")`来对所有`Address` 嵌入式文档创建一个索引
- 或者`EnsureIndex("Address.Street")`使用点分表示法为`Street`创建一个索引
- 索引被作为`BsonDocument`字段执行。如果你使用一个自定义的`ResolvePropertyName` 或 `[BsonField]` 特性，你必须使用你的文档字段名而不是属性名。参考 [Object Mapping](Object-Mapping).
- 在强类型集合中，你可以使用一个lambda表达式来定义一个索引字段: `EnsureIndex(x => x.Name)`
- 你可以使用fluent mapping api定义你的索引。参考 [Object Mapping](Object-Mapping)

### MultiKey 索引

当你在一个数组类型字段上创建索引时，所有值都被包含在索引键中，你可以搜索任何值。

```C#
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
}

var cusomters = db.GetCollection<Customer>("customers");

customers.Insert(new Customer { Name = "John", Phones = new string[] { "1", "2", "5" });
customers.Insert(new Customer { Name = "Doe", Phones = new string[] { "1", "8" });

var result = customers.Query(x => x.Phones.Contains("1")); // 两个文档都返回
```

### 限制

- 索引值必须要少于512字节(BSON序列化后)
- 每个集合可以有最多16个索引 - 包括`_id`主键
- `[BsonIndex]`在嵌入式文档中是不被支持的