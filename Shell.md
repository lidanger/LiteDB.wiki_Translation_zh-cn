LiteDB项目有一个简单的控制台应用程序（LiteDB.Shell.exe），可以用于你的数据库。查看，更新和测试你的数据时它会非常有用。Shell工具使用一个断开的'LiteEngine'实例，因此每条命令会连接（in read only if you are only quering）和执行然后断开。

### 参考

#### Shell控制台命令

这里的命令只适用于控制台应用程序‘LiteDB.Shell.exe’：

- **`help [full]`** - 显示基本的或全部帮助命令
- **`open <filename|connectionString>`** - 打开新数据库。如果不存在，创建一个新的。可以传递一个使用所有选项的连接字符串(如密码或超时)
- **`close`** - 关闭当前数据库
- **`pretty`** - 以好看的格式显示JSON
- **`ed`** - 在notepad.exe中打开编辑上一条命令
- **`run <filename>`** - 读filename并添加每一行为一个新命令
- **`spool [on|off]`** - 打开/关闭所有命令的重定向
- **`timer`** - 在命令提示符显示计时器
- **`--`** - 注释 (忽略行剩余部分)
- **`/<command>/`** - 支持多行命令
- **`upgrade /<filename|connectionString>/`** - 从LiteDB v2升级数据文件
- **`version`** - 显示LiteDB程序集版本
- **`quit`** - 退出shell应用程序

#### 集合命令

语法: `db.<collection>.<command>`

- **`insert <jsonDoc>`** - 使用查询过滤语法找到一个文档
    - `db.customer.insert { _id:1, Name: "John Doe", Age: 38 }`
    - `db.customer.insert { Name: "John Doe" }` - 自动创建Id作为`ObjectId`

- **`bulk <filename.json>`** - 插入一个JSON文件中的所有文档。JSON文件必须是一个JSON文档数组
    - `db.customer.bulk my-documents.json`

- **`update <jsonDoc>`** - 使用`_id`更新一个文档
    - `db.customer.update { _id:1, Name: "Joana Doe", Age: 29 }`

- **`delete <filter>`** - 使用过滤语法删除文档
    - `db.customer.delete _id = 1`
    - `db.customer.delete Name like "Jo"`

- **`ensureIndex <field> [true|<jsonOptions>]`** - 在集合field字段上创建一个新的索引。支持json选项[参考Indexes](Indexes)
    - `db.customers.ensureIndex Name true` - 创建一个唯一索引
    - `db.customers.ensureIndex Name { unique: true, removeAccents: false }`

- **`indexes`** - 列出集合上的所有索引
    - `db.customers.indexes`

- **`dropIndex <field>`** - 终止一个索引
    - `db.customers.dropIndex Name`

- **`drop`** - 删除一个集合以及所有包含的文档。如果不存在返回
    - `db.customer.drop`

- **`rename`** - 重命名一个集合。如果成功返回true，出现错误返回false
    - `db.customer.rename my_customers`

- **`count <filter>`** - 使用定义的过滤器统计文档。参考如下`<filter>`语法
    - `db.customer.count Name = "John Doe"`
    - `db.customer.count Age between [20, 40]

- **`min <field>`** - 获取字段索引最小的值
    - `db.customer.min _id`
    - `db.customer.min Age`

- **`max <field>`** -
    - `db.customer.max _id`
    - `db.customer.max Age`

- **`find [filter][skip N][limit M]`** - 使用查询过滤语法找到文档。参考如下`<filter>`语法
    - `db.customer.find`
    - `db.customer.find limit 10`
    - `db.customer.find Name = "John Doe"`
    - `db.customer.find Age between [20, 40]`
    - `db.customer.find Name LIKE "John" and Age > 30`

- `<filter>` = `<field> [=|>|>=|<|<=|!=|like|contains|in|between] <jsonValue>` 
- `<filter>` = `<filter> [and|or] <filter>`
- `<jsonDoc>`和`<jsonValue>` 是JSON 扩展格式。参考 [Data Structure](Data-Structure).

#### 文件存储

语法: `fs.<command>`

- **`upload <fileId> <filename>`** - 上传一个本地文件到数据库。如果存在fileId，覆盖数据内容
    - `fs.upload my-file picture.jpg`
    - `fs.upload $/photos/pic-001 C:\Temp\mypictures\picture-1.jpg`

- **`download <fileId> <filename>`** - 下载已存在的数据库文件为一个本地文件。
    - `fs.download my-file copy-picture.jpg`
    - `fs.download $/photos/pic-001 C:\Temp\mypictures\copy-picture-1.jpg`

- **`update <fileId> <jsonDoc>`** - 更新文件元数据
    - `fs.update my-file { user: "John", machine: "srv001" }`

- **`delete <fileId>`** - 删除一个文件
    - `fs.delete my-file`

- **`find [fileId]`** - 列出数据库中的所有文件或从fileId参数开始
    - `fs.find`
    - `fs.find $/photos/`

#### 数据库实用工具

语法: `db.<command>`

- **`userversion [N]`** - 获取/设置用户数据库文件版本

- **`shrink [password]`** - 缩小数据库，删除空页和改变密码（可选）。如果未提供密码，新数据文件不会加密。
