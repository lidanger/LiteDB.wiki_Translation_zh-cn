LiteDatabase可以使用一个连接字符串初始化，语法为`key1=value1; key2=value2; ...`。如果你的连接字符串中没有`;`，LiteDB假定你的连接字符串是Filename键。键不分大小写。

### 选项

- **`Filename`** (string): 从DLL目录的完整路径或相对路径。
- **`Journal`** (bool): 启用或禁用浮点写检查来确保持久性(默认: true)
- **`Password`** (string): 使用一个密码加密(使用AES)你的数据文件(默认: null - 未加密)
- **`Cache Size`** (int): 缓存的页面最大数量。超过这个大小，将数据存在磁盘中以防止使用过多内存(默认: 5000)
- **`Timeout`** (TimeSpan): 等待解锁操作的超时时间(线程锁或锁定文件)
- **`Mode`** (Exclusive|ReadOnly|Shared): 怎样打开数据文件(默认: NET35，`Shared`，NetStandard，`Exclusive`)
- **`Initial Size`** (string|long): 如果数据库是新的，分配空间的初始长度 - 支持 KB, MB, GB (默认: null)
- **`Limit Size`** (string|long): 数据文件的最大限制 - 支持 KB, MB, GB (默认: null)
- **`Upgrade`** (bool): 如果为true，尝试从旧版本（v2)升级数据文件(默认: null)
- **`Log`** (byte): 数据库调试信息 - 使用`LiteDatabase.Log` (默认: Logger.NONE)
- **`Async`** (bool): 支持"sync over async" 文件流创建，以便在UWP中使用来访问任何磁盘文件夹(只用于NetStandard, 默认: false)




