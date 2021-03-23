### 倒排索引

```
倒排索引是基于doc做的，倒排索引--根据关键词制作索引
1.分词
2.关联doc

数据结构：
1.包含关键词的doc list
2.关键词在每个doc中出现的次数TF(term frequency)
3.关键词在整个索引中
```



### ELK部署

```
1.配置虚拟机环境及网络
2.下载elasticsearch -7.6.2并解压
3.启动，查看报错，不能使用root启动Es
config 目录没有权限
4.配置ES @/comfig/elasticsearch.yml
	绑定ip需要用内网ip? 内网ip如何设置?
```

