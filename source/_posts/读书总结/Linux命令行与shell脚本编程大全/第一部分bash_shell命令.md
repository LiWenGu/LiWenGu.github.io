---
title: 一、bash shell 命令
date: 2018-05-16 23:51:00
updated: 2018-05-16 23:51:00
comments: true
tags:
  - shell
categories: 
  - [读书总结, Linux命令行与shell脚本编程大全]
permalink: linux_cli_shell/1.html    
---

### 1. bash 手册

用于查看命令的具体详情

man xxx

## 2. ls 文件和目录列表

-a 显示隐藏文件  
文件名支持 `*？` 符号过滤  

### 3. 处理文件

-i 询问参数  
touch  
cp file1 file2：复制文件。参数 -R 用于递归复制文件  
mv file1 file2：移动文件
rm file1：删除文件，文件名支持`?*`

### 4. 处理目录

mkdir：创建目录。-p 创建多个目录和子目录  
rmdir：删除空目录
rm -rf：递归删除，-r 递归遍历，-f 删除不提示

### 5. 查看文件内容

file file1：获取文件的类型  
cat file1：显示文本所有内容，参数 -n 加上行数，参数 -b 只给有内容的行加行数  
more file1：  
less file1：  
tail file1：参数 -f，动态查看文件内容

### 6. 检测程序

ps： -ef  
top：实时检测，q 退出  
kill PID：-9 参数强制  
killall Name：关闭进程名，可以使用通配符

### 7. 检测磁盘空间

mount：挂载媒体的  
unmount：移除可移动设备  
sort file：文件排序  
grep pattern file：在 file 文件中查找 pattern 的行。-v 参数反向搜索  
gzip/gunzip：压缩解压文件
tar：-A 追加归档，-x 提取文件

### 8. 理解 shell

&：将任务置入后台模式  
which 命令：查看命令的对应路径  
history：最近的使用过的命令列表  

### 9. 使用Linux环境变量

查看环境全局变量：printenv/env  
查看环境局部变量：set  
export：将一个局部变量的key导出到全局环境中  

### 10. 管理文件系统

1. ext 文件系统：单文件不能超过2GB。
2. ext2 文件系统：保存更多信息。
3. 日志文件系统：先将数据直接写入存储设备再更新索引节点表->文件的更改写入到临时文件中，数据成功写到存储设备和索引节点表后再删除对应的日志条目。
4. ext3 文件系统：在 ext2 基础上，给每个存储设备增加了一个日志文件。
5. ext4 文件系统。

### 11. 安装软件程序

1. Debian（Ubuntu）：dpkg 命令。
2. Red Hat：rpm 命令。yum 命令。

### 12. 使用编辑器

vim  
nano  
emacs

## 13. 参考

1. 初识Linux shell：http://www.ituring.com.cn/book/tupubarticle/11430
2. 走进shell：http://www.ituring.com.cn/book/tupubarticle/11431
3. 基本的bash shell命令：http://www.ituring.com.cn/book/tupubarticle/11432
4. 更多的bash shell命令： http://www.th7.cn/system/lin/201704/210752.shtml
5. 理解shell：http://www.th7.cn/system/lin/201704/211006.shtml
6. 使用Linux环境变量：http://www.voidcn.com/article/p-vizgjbtx-bmq.html
7. 理解Linux文件权限：http://www.voidcn.com/article/p-whblgnni-bmq.html
8. 管理文件系统：https://www.aliyun.com/jiaocheng/123749.html
9. 安装软件程序：https://www.aliyun.com/jiaocheng/123748.html
10. 使用编辑器：http://www.voidcn.com/article/p-fokuslvn-bnt.html
