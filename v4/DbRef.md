LiteDB 是一个文档数据库，因此集合之间没有 JOIN，但可以使用嵌入文档 (子文档) 或在集合之间创建引用。你可以使用 `[BsonRef]` 特性或在fluent API 映射器中使用 `DbRef` 方法来创建引用。

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

如果不做任何映射，当你存储 `Order` 时，`Customer` 会被保存在一个嵌入文档 (没有链接到任何其他集合)中。改变 `Customer` 集合中的顾客名称，不会影响到 `Order`。

```JS
Order => { _id: 123, Customer: { CustomerId: 99, Name: "John Doe" } }
```

如果你想让 `Order` 只存储顾客的引用，你可以修饰你的类：

```C#
public class Order
{
    public int OrderId { get; set; }

    [BsonRef("customers")] // 这里 "customers" 是 Customer 集合的名称
    public Customer Customer { get; set; }
}
```

注意，`BsonRef` 修饰的是整个将要引用的对象，而不是引用其他集合对象的一个 `customerid` 整数字段。

或者使用 fluent API:

```C#
BsonMapper.Global.Entity<Order>()
    .DbRef(x => x.Customer, "customers"); // 这里 "customers" 是 Customer 集合的名称
```

**注意:** `Customer` 需要有一个 `[BsonId]` 已定义。

现在，当你存储 `Order` 时，你其实只是在存储链接性的引用。

```JS
Order => { _id: 123, Customer: { $id: 4, $ref: "customers"} }
```

### 查询结果

当你查询交叉引用其他集合的文档时，你可以使用 `Include` 方法在查询前自动加载引用。 

```C#
var orders = db.GetCollection<Order>("orders");

var order1 = orders
    .Include(x => x.Customer)
    .FindById(1);
```

DbRef 同时支持 `List<T>` 或 `Array`，例如这样：

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

从数据文件还原数据时，LiteDB 会检查你的 `Products` 字段是否为 null 或空列表。如果在查询中不使用 `Include`，加载时只有 `ID` 有值 (所有其他属性都保持 default/null 值)。

在 v4 中，此 Include 过程发生在 BsonDocument 引擎级别。当然，也支持任何级别的 Include，只要使用 `Path` 语法就行：

```C#
db.Find("orders", Query.All(), new string[] { "$.Customer", "$.Products[*]" });
```

如果是 `LiteCollection` 或 `Repository`，也可以使用 Linq 语法：

```C#
// 资源库 fluent 语法
db.Query<Order>()
    .Include(x => x.Customer)
    .Include(x => x.Products)
    .ToList();
```

