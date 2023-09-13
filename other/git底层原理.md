git 主要包含三大部分：

+ 上层命令（Porcelain Commands）
+ 底层命令（Plumbing Commands）
+ 对象数据库（Object Database）

## 上层命令
Git命令基本上都是上层命令，如：commit、add、checkout、branch、remote 等。上层命令通过组合底层命令或直接操作底层数据对象，使 Git 底层实现细节对用户透明。

## 底层命令
底层命令如：update-cache、write-tree、read-tree、commit-tree、cat-file、show-diff 等。

## 对象数据库

Git 最核心、最底层 的部分则是其所实现的一套 对象数据库（Object Database），其本质是一个基于 Key-Value 的内容寻址文件系统（Content-addressable File System）。

## 对象模式

Git 对象模型主要包括以下 4 种对象

+ 二进制对象（Blob Object）
+ 树对象（Tree Object）
+ 提交对象（Commit Object）
+ 标签对象（Tag Object）

所有对象均存储在 .git/objects/ 目录下，并采用相同格式进行表示，其可以分为两部分：

+ 头部信息：类型 + 空格 + 内容字节数 + \0
+ 存储内容

Git 使用两部分的未压缩内容的 40 位 SHA-1 值（前 2 为作为子目录，后 38 位作为文件名）作为快照文件的唯一标识，并对它们进行 zlib 压缩，然后将压缩后的结果作为快照文件的实际内容进行存储。

### Blob Object

Block Object 用于存储普通文件的内容数据，其头部信息为 “blob” + 空格 + 内容字节数 + \0，存储内容为对应文件的内容快照。

下面，我们使用底层命令 git cat-file 来查看 analyze-git 仓库的一个 Blob Object 的存储内容。

```shell
# 查看对象的类型
$ git cat-file -t fa49b077972391ad58037050f2a75f74e3671e92
blob

# 查看对象的内容长度
$ git cat-file -s fa49b077972391ad58037050f2a75f74e3671e92
9

# 查看对象的内容
$ git cat-file -p fa49b077972391ad58037050f2a75f74e3671e92
new file
```