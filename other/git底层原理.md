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