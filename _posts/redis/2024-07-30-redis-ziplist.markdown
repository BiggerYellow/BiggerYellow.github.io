---
layout: default
redis: true
modal-id: 30000004
date: 2024-07-30
img: pexels-ahmetkurt-12661193.jpg
alt: image-alt
project-date: July 2024
client: Start Bootstrap
category: redis
subtitle: 压缩列表
description: ziplist
---

### 简单动态字符串 - sds    
> 一种用于存储二进制数据的一种结构，具有动态扩容的特点，其实现位于src/sds.h和src/sds.c中。从版本3.2开始，sds底层数据结构也发生改变。  


**字符串编码类型**  
字符串有三种 encoding 类型 int 、 raw 、 embstr 。  
- int：用于整数类型  
- embstr：用于短字符串
- raw：用于长字符串  

 


> https://pdai.tech/md/db/nosql-redis/db-redis-overview.html