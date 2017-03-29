`LiteRepository`是一个访问你的数据库的新类。LiteRepository通过`LiteDatabase`实现，只是一个不使用`LiteCollection`类快速访问你的数据并使用Fluent API查询的层

```C#
using(var db = new LiteRepository(connectionString))
{
    // 简单访问，插入/更新/更新或插入/删除
    db.Insert(new Product { ProductName = "Table", Price = 100 });

    db.Delete<Product>(x => x.Price == 100);

    // 使用fluent query查询
    var result = db.Query<Order>()
        .Include(x => x.Customer) // add dbref 1x1
        .Include(x => x.Products) // add dbref 1xN
        .Where(x => x.Date == DateTime.Today) // 使用索引查询
        .Where(x => x.Active) // 使用索引查询
        .ToList();

    var p = db.Query<Product>()
        .Where(x => x.ProductName.StartsWith("Table"))
        .Where(x => x.Price < 200)
        .Limit(10)
        .ToEnumerable();

    var c = db.Query<Customer>()
        .Where(txtName.Text != null, x => x.Name == txtName.Text) // 条件筛选器
        .ToList();

}
```

集合名称可以省略，会被新的`BsonMapper.ResolveCollectionName`函数解析（默认：`typeof(T).Name`）。

这个API是受这个牛B的项目的启发[NPoco Micro-ORM](https://github.com/schotime/NPoco)
