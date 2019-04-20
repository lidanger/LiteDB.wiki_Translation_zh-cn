表达式一般是路径或公式，可用于访问和修改你的文档数据。LiteDB 路径基于 [JSON 路径](http://goessner.net/articles/JsonPath/)，支持近似的语法在一个单独的文档中导航。路径在任何情况下总是返回一个 `IEnumerable<BsonValue>`。

`BsonExpression` 是解析字符串表达式 (或路径) 并将其编译为 LINQ 表达式以便被 LiteDB 快速求值的一个类。解析器使用

- 开头是 `$` 的路径： `$.Address.Street`
- 开始于 `[0-9]*` 的 Int 值： `123`
- 开始于 `[0-9].[0-9]`的 Double 值： `123.45`
- 字符串使用单 `'` 表示： `'Hello World'`
- Null 就是 `null`
- Bool 使用 `true` 或 `false` 关键字表示。
- 函数使用 `FUNCTION_NAME(par1, par2, ...)` 表示： `LOWER($.Name)`

示例：
- `$.Price`
- `$.Price + 100`
- `SUM($.Items[*].Price)`

```C#
var expr = new BsonExpression("SUM($.Items[*].Unity * $.Items[*].Price)");
var total = expr.Execute(doc, true).First().AsDecimal;
```

表达式也可以用于：

- 创建一个基于表达式的索引：
    - `collection.EnsureIndex("Name", true, "LOWER($.Name)")`
    - `collection.EnsureIndex(x => x.Name, true, "LOWER($.Name)")`
- 基于表达式查询一个集合中的文档 (全扫描搜索)
    - `Query.EQ("SUBSTRING($.Name, 0, 1)", "T")`
- shell 命令 Update 
    - `db.customers.update $.Name = LOWER($.Name) where _id = 1`
- 为 shell 命令 SELECT 生成一个文档结果
    - `db.customers.select $._id, ARRAY(UPPER($.Books[*].Title)) AS titles where $.Name startswith "John"`

# 路径

- `$` - Root
- `$.Name` - 名称字段
- `$.Name.First` - 名称子文档的 First 字段  
- `$.Books` - 返回 book 数组值 
- `$.Books[0]` - 返回 books 数组中的第一本书
- `$.Books[*]` - 返回 books 数组中的所有书
- `$.Books[*].Title` 返回所有 books 的标题
- `$.Books[-1]` - 返回 books 数组中的最后一本书

路径同时支持使用表达式来过滤子节点

- `$.Books[@.Title = 'John Doe']` - 返回所有标题是 John Doe 的书
- `$.Books[@.Price > 100].Title` - 返回所价格大于 100 的书的标题

在数组中，`@` 表示当前子文档。同时，也可以在表达式内部使用函数：

- `$.Books[SUBSTRING(LOWER(@.Title), 0, 1) = 'j']` - 返回所有标题开头是 J 或 j 的书。

# 函数

函数总是使用 `IEnumerable<BsonValue>` 作为输入输入/输出参数来运行。

- `LOWER($.Name)` - 返回只有一个元素的 `IEnumerable`
- `LOWER($.Books[*].Title)` - 返回所有值都是小写的 `IEnumerable` 

## 运算符

运算符是与数学语法同样实现的函数。支持

- `ADD(<left>, <right>)`: `<left> + <right>` (如果任何一边是字符串，合并值并返回字符串。)
- `MINUS(<left>, <right>)`: `<left> - <right>`
- `MULTIPLY(<left>, <right>)`: `<left> * <right>`
- `DIVIDE(<left>, <right>)`: `<left> / <right>`
- `MOD(<left>, <right>)`: `<left> % <right>`
- `EQ(<left>, <right>)`: `<left> = <right>`
- `NEQ(<left>, <right>)`: `<left> != <right>`
- `GT(<left>, <right>)`: `<left> > <right>`
- `GTE(<left>, <right>)`: `<left> >= <right>`
- `LT(<left>, <right>)`: `<left> < <right>`
- `LTE(<left>, <right>)`: `<left> <= <right>`
- `AND(<left>, <right>)`: `<left> && <right>` (左边和右边必须是 boolean)
- `OR(<left>, <right>)`: `<left> || <right>`: (左右和右边必须是 boolean)
- `IIF(<condition>, <ifTrue>, <ifFalse>)`: (条件必须是一个 boolean 值)

### 示例

- `db.dummy.select ((2 + 11 % 7)-2)/3` => `1.33333`
- `db.dummy.select 'My Id is ' + $._id` => `"My Id is 1"`
- `db.customers.select $._id, IIF($._id < 20, 'Less', 'Greater') + ' than twenty' as Info`
    - => `{ _id: 1, Info: "Less than twenty" }`

## 字符串

字符串函数只工作于你的 `<values>` 是字符串的情况下。任何其他数据类型都会跳过

| 函数                                          | 描述                                             |
| ---                                           | ---                                             |
| `LOWER(<values>)`                             | 与 `String` 的 `ToLower()` 相同。返回一个字符串   |
| `UPPER(<values>)`                             | 与 `String` 的 `ToUpper()` 相同。返回一个字符串   |
| `SUBSTRING(<values>, <index>, <length>)`      | 与 `String` 的 `Substring()` 相同。返回一个字符串 |
| `LPAD(<values>, <totalWidth>, <paddingChar>)` | 与 `String` 的 `PadLeft()` 相同。返回一个字符串   |
| `RPAD(<values>, <totalWidth>, <paddingChar>)` | 与 `String` 的 `PadRight()` 相同。返回一个字符串  |
| `FORMAT(<values>, <format>)`                  | 与 `String` 的 `Format()` 相同。任何数据类型即可运行 (使用 `RawValue`). 返回一个字符串 |

## 聚合

| 函数               | 描述                    |
| ---               | ---                     |
| `SUM(<values>)`   | 所有数值取总和返回一个数值 |
| `COUNT(<values>)` | 为所有值计数返回一个整数   |
| `MIN(<values>)`   | 获得值列表的最小值        |
| `MAX(<values>)`   | 获得值列表的最大值        |
| `AVG(<values>)`   | 计算值列表的平均值        |

所有聚合函数在单独的文档中有效，并且总是使用一个数组的 IEnumerable 选择器： (`SUM($.Items[*].Price)`)

## 数据类型

| 函数                  | 描述                                                  |
| ---                   | ---                                                   |
| `ARRAY(<values>)`     | 转换所有值到一个单独的数组中
| `JSON(<strings>)`     | 反序列化字符串值到一个 `BsonDocument` 值
| `KEYS(<docs>)`        | 返回所有文档的所有键
| `IS_DATE(<values>)`   | 为每个是日期的值返回 true 
| `IS_NUMBER(<values>)` | 为每个是数字的值返回 true 
| `IS_STRING(<values>)` | 为每个是字符串的值返回 true 

所有函数都是 `BsonExpression` 的静态方法。
