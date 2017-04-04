LiteDB是一个文档数据库，因此集合间没有连接。你可以使用嵌入文档（子文档）或在集合间创建一个引用。要创建这个引用，你可以使用`[BsonRef]`属性，或从Fluent API映射器中使用`DbRef`方法。

### 在数据库初始化时映射一个引用

```C#
public class Customer
{
    public int CustomerId { get; set; }
    public string Name { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    public Customer Customer { get; set; }
}
```

如果你不做任何映射，当你保存一个`Order`，`Customer`会被保存为一个嵌入的文档（没有链接到任何其他集合）。如果你改变`Customer`集合中的顾客名称，这个更改不会影响`Order`集合。

```JS
Order => { _id: 123, Customer: { CustomerId: 99, Name: "John Doe" } }
```

如果你想顾客引用只存储在`Order`集合，你可以装饰你的类：

```C#
public class Order
{
    public int OrderId { get; set; }

    [BsonRef("customers")] // 这里"customers"是Customer集合名称
    public Customer Customer { get; set; }
}
```

或使用Fluent API:

```C#
BsonMapper.Global.Entity<Order>()
    .DbRef(x => x.Customer, "customers"); // 这里"customers"Customer集合名称
```

现在，当你存储`Order`时，你将只存储链接引用。

```JS
Order => { _id: 123, Customer: { $id: 4, $ref: "customers"} }
```

### 查询结果

当你使用一个交叉集合引用查询一个文档，你可以在查询前使用`Include`方法自动加载引用。

```C#
var orders = db.GetCollection<Order>("orders");

var order1 = orders
    .Include(x => x.Customer)
    .FindById(1);
```

DbRef也支持`List<T>` 或 `Array`，像这样：

```C#
public class Product
{
    public int ProductId { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    public DateTime OrderDate { get; set; }
    public List<Product> Products { get; set; }
}

BsonMapper.Global.Entity<Order>()
    .DbRef(x => x.Products, "products");
```

如果从数据文件恢复时你的`Products`字段是null或空列表，LiteDB将什么也不做（respect）。如果你在查询中不使用`Include`，类被加载时，只会设置`ID`（所有其他属性会一直是默认/null值）。
