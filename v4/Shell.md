LiteDB 项目包含一个简单的控制台应用程序 (LiteDB.Shell.exe)，可用于查看、更新以及测试你的数据，在处理你的数据库时非常有用。

在 v4 中，LiteDB 将对 shell 命令的支持退回到 `LiteDB.dll`中，因此，现在，shell 命令是 LiteDB 的一部分 (不只是一个外置工具)。

### 参考

#### Shell 控制台命令

这里的命令只在控制台应用程序 `LiteDB.Shell.exe` 中有效：

- **`help [full]`** - 显示基础或全部命令的帮助信息
- **`open <filename|connectionString>`** - 打开新数据库。如果不存在，创建一个新的。可以传递一个连接字符串，可以使用所有选项 (像 password 或 timeout)
- **`close`** - 关闭当前数据库
- **`pretty`** - 以优美的格式显示 JSON
- **`ed`** - 用 notepad.exe 打开最后一条命令编辑
- **`run <filename>`** - 读取 filename，并添加每行为一条新命令
- **`spool [on|off]`** - 打开/关闭所有命令的输出管道
- **`timer`** - 在命令提示符下显示一个定时器
- **`--`** - 注释 (忽略行的剩余部分)
- **`/<command>/`** - 支持多行命令
- **`upgrade /<filename|connectionString>/`** - 从 LiteDB v2 版本升级数据文件
- **`version`** - 显示 LiteDB 程序集版本
- **`quit`** - 退出 shell 应用程序

#### Collections 命令

语法: `db.<collection>.<command>`

- **`insert <jsonDoc>`** - 使用查询过滤器语法查找一个文档
    - `db.customer.insert { _id:1, Name: "John Doe", Age: 38 }`
    - `db.customer.insert { Name: "John Doe" }` - 自动创建 Id 为 `ObjectId`

- **`bulk <filename.json>`** - 插入一个 JSON 文件内的所有文档。JSON 文件必须是文档的一个 JSON 数组
    - `db.customer.bulk my-documents.json`

- **`update <jsonDoc>`** - 使用 `_id` 更新一个已存在的文档
    - `db.customer.update { _id:1, Name: "Joana Doe", Age: 29 }`

- **`update <field|path>=<value|expression>, [..] [where <filter>]`** - 使用过滤器查询更新多个文档字段
    - `db.customer.update Name = "John" where _id = 1`
    - `db.customer.update Age = $.Age + 1 where _id > 0`
    - `db.customer.update Addresses[*].Full = Addresses[*].Street + ", " + Addresses[*].Number`

- **`delete <filter>`** - 使用过滤器语法删除文档
    - `db.customer.delete _id = 1`
    - `db.customer.delete Name like "Jo"`

- **`ensureIndex <field|name> [unique] [using <expr>]`** - 在集合的字段上创建一个新索引。支持 json 选项，参考[索引](Indexes)
    - `db.customers.ensureIndex Name unique` - 创建一个唯一索引
    - `db.customers.ensureIndex NameLower using LOWER($.Name)`

- **`indexes`** - 列出集合中的所有索引
    - `db.customers.indexes`

- **`dropIndex <field|name>`** - 删除一个索引
    - `db.customers.dropIndex Name`

- **`drop`** - 删除一个集合以及其中的所有文档。如果不存在返回 false 
    - `db.customer.drop`

- **`rename`** - 重命名一个集合。如果成功返回 true，如果出现一个错误返回 false
    - `db.customer.rename my_customers`

- **`count <filter>`** - 使用定义的过滤器为文档计数。参考如下的 `<filter>` 语法
    - `db.customer.count Name = "John Doe"`
    - `db.customer.count Age between [20, 40]`

- **`min <field>`** - 获取字段索引的最低值
    - `db.customer.min _id`
    - `db.customer.min Age`

- **`max <field>`** - 获取字段索引的最高值
    - `db.customer.max _id`
    - `db.customer.max Age`

- **`find [filter][skip N][limit M][includes p0, p1, ..]`** - 使用查询过滤语法查找文档。参考如下的 `<filter>` 语法
    - `db.customer.find`
    - `db.customer.find limit 10`
    - `db.customer.find Name = "John Doe"`
    - `db.customer.find Age between [20, 40]`
    - `db.customer.find Name LIKE "John" and Age > 30`
    - `db.customer.find skip 10 limit 10 includes $.Orders[*]`

- **`select [path|expr] as Name1, [path|expr] [where <filter>][skip N][limit M][include p0, p1, ..]`** - 查找并转换文档
    - `db.customer.select $ where Age > 30`
    - `db.customer.select LOWER($.Addresses[*].Street)  where Age > 30`
    - `db.customer.select $.Name as Name, ARRAY($.Addresses[@.City = 'NY']) as addresses where Age > 30`

- `<filter>` = `<field> [=|>|>=|<|<=|!=|like|contains|in|between] <jsonValue>` 
- `<filter>` = `<filter> [and|or] <filter>`
- `<jsonDoc>` 和 `<jsonValue>` 是 JSON 扩展格式。参考[数据结构](Data-Structure).

#### FileStorage

语法: `fs.<command>`

- **`upload <fileId> <filename>`** - 上传一个本地文件到数据库。如果有 fileId，覆盖数据内容
    - `fs.upload my-file picture.jpg`
    - `fs.upload $/photos/pic-001 C:\Temp\mypictures\picture-1.jpg`

- **`download <fileId> <filename>`** - 下载存在的数据库文件到一个本地文件。
    - `fs.download my-file copy-picture.jpg`
    - `fs.download $/photos/pic-001 C:\Temp\mypictures\copy-picture-1.jpg`

- **`update <fileId> <jsonDoc>`** - 更新文件元数据
    - `fs.update my-file { user: "John", machine: "srv001" }`

- **`delete <fileId>`** - 删除一个文件
    - `fs.delete my-file`

- **`find [fileId]`** - 列出数据库中的所有文件，或以 fileId 参数开头的文件
    - `fs.find`
    - `fs.find $/photos/`

#### 数据库实用工具

语法: `db.<command>`

- **`userversion [N]`** - 获取/设置用户数据库文件版本

- **`shrink [password]`** - 通过删除空白页面和修改密码 (可选) 来收缩数据文件。如果未提供密码，新数据文件不再加密。
