内存IO, 磁盘IO ， 网络IO

 Linux 系统

各类文件标识符

### page cache 是什么

​    用户在写文件时，内核会分配pageCache到内存中，先写到pageCache中，达到一定阈值，再写到磁盘中。

​	内核为了优化IO性能使用了将文件存储到pageCache中进行临时缓存的方式，但却会在异常断电的时候丢失数据 

​	内核维护，一个中间层

​	使用多大内存？

​	是否淘汰？ 如果内存达到阈值，则将最老的pageCache进行判断是否dirty，如果是，则写入磁盘并进行淘汰。如果不是，则不需要写入，直接淘汰

​	是否延时，还是立刻持久化？是否丢数据？

### 如何看centOs的内核配置项？

​	sysctl -a | dirty    查看几个项，配置各项阈值

 	free 查看内存 

### NIO

​	randomAccessFile介绍

​		不同改的申请内存方式 allocated byteBuffer allocatedDirect 方法分配内存到linux堆中创建off heap

​	seek方法寻址

​	堆外

​	结论： 性能	onHeap < offHeap < mapped (仅限文件系统)

​	操作系统没有绝对的可靠性，为什么设计PageCahce? 较少IO调用，优先使用内存。即便想要可靠性，使用性能最低的方式，但是单点问题，会让这些前功尽弃。因此，还要使用HA，主从复制。

## 网络IO

​	命令 

lsof  -p 查看文件描述符;

netstat -natp 

tcpdump 抓包命令



BIO,NIO,AIO 整理

 好他妈的困啊，好像睡一会儿。。但是不太敢睡。玛德

tcp是什么。面向连接的可靠的传输协议 

socket 是什么？是一个四元组（CIP CPort + S IP S Port）