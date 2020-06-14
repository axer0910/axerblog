---
title: 树莓派cifs挂载网络NTFS文件系统，php-fpm一直报错File not found解决方法
tag: 折腾
date: 2019-12-26
updated: 2019-12-26
---

树莓派上网络挂载windows的ntfs的系统，使用php-fpm一直报错File not found。
经过检查脚本文件路径正确，应该是列出文件遇到了问题。
一开始以为是权限问题，尝试命令`mount -t cifs //192.168.1.23/M /mnt/d -o user=xxx,pass=xxx,gid=1000,uid=1000,iocharset=utf8,dir_mode=0777,file_mode=0777` 将用户挂载为和执行php-fpm相同的用户（gid和uid选项），仍然不行。
最后找到了一样的问题：
https://serverfault.com/questions/425608/using-a-mounted-ntfs-share-with-nginx
加上了参数`noserverino`解决。
noserverino：
>Client generates inode numbers itself rather than using the actual ones from the server. 
使用操作系统自身生成inode，而不是用cifs的server（windows）生成
