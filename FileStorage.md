要保持内存占有不大，LiteDB限制一个文档大小为1Mb。对于文本文档，这是一个很大的尺寸。但对于很多二进制文件来说，1Mb就太小了。因此LiteDB实现了`FileStorage`，一个用于存储文件和流的自定义集合。

LiteDB使用两个特殊的集合来分裂文件内容到分块（chunks）：

- `_files` 集合只存储文件引用和元数据

```JS
{
    _id: "my-photo",
    filename: "my-photo.jpg",
    mimeType: "image/jpg",
    length: { $numberLong: "2340000" },
    uploadDate: { $date: "2015-01-01T00:00:00Z" },
    metadata: { "key1": "value1", "key2": "value2" }
}
```

- `_chunks` 集合按1MB分块存储二进制数据。

```JS
{
    _id: "my-photo\00001",
    data: { $binary: "VHlwZSAob3Igc ... GUpIGhlcmUuLi4" }
}
{
    _id: "my-photo\00002",
    data: { $binary: "pGaGhlcmUuLi4 ... VHlwZSAob3Igc" }
}
{
   ...
}
```

文件用`_id`字符串值标识，遵循下列规则：

- 以字符，数字，`_`, `-`, `$`, `@`, `!`, `+`, `%`, `;` 或 `.` 开头
- 如果包含一个 `/`，必须紧跟以上字符 

要更好地组织很多文件，你可以使用`_id`作为一个`directory/file_id`。这是一个不错的方案，可以使用`Find`方法快速查找一个文件夹下的所有文件。

示例: `$/photos/2014/picture-01.jpg`

`FileStorage`集合包含简单的方法，如:

- **`Upload`**: 发送文件或流到数据库。可以被用于文件或`Stream`。如果文件已存在，文件内容会被覆盖。
- **`Download`**: 从数据库获取你的文件，并复制到`Stream`参数。
- **`Delete`**: 删除一个文件引用和所有数据分块
- **`Find`**: 在`_files`集合中查找很多文件中的一个。返回`LiteFileInfo`类，稍后它可以用于下载数据。
- **`SetMetadata`**: 更新已存储文件的元数据。这个方法不改变已存储文件的值。它更新`_files.metadata`的值。
- **`OpenRead`**: 按`_id`查找文件，并返回一个`LiteFileStream`来将文件内容读取为流

```C#
// 从文件系统上传一个文件
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", @"C:\Temp\picture-01.jpg");

// 从一个流上传一个文件
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", stream);

// 只查找文件引用 - 如果没有找到返回null
LiteFileInfo file = db.FileStorage.FindById("$/photos/2014/picture-01.jpg");

// 现在，加载二进制数据并保存到文件系统
file.SaveAs(@"C:\Temp\new-picture.jpg");

// 或者获取二进制数据为流并复制到另一个流
file.CopyTo(Response.OutputStream);

// 查找一个"directory"中的所有文件引用
var files = db.FileStorage.Find("$/photos/2014/");
```

`FileStorage`不支持事务，以防止在保存到磁盘前把文件的所有部分都放进内存。每个数据分块使用事务。每个上传的数据分块在一个单独的事务中提交。

当上传每一个分块时，`FileStorage`在`0`保持`_files.length`。当所有分块完成上传时，`FileStorage`合计每一个分块的大小更新`_files.length`。如果你尝试下载一个0字节长度的文件，下载的文件将是毁坏的。一个毁坏的文件并不一定意味着一个毁坏的数据库。 
