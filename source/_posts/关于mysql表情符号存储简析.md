---
title: 关于mysql表情符号存储简析
date: 2018-11-20 16:28:47
tags: [mysql,表情符号,emoji,编码问题]
---
## 关键点的分析
关于mysql存储表情符号的教程，网上一大堆。但根据一些教程设置后，还是会出现乱码的现象。根本的原因在于，没有了解整个存储过程。

解决表情符号问题要从以下几点关注：

* clinet端
* connection链接（也可以理解为运输部分）
* table存储
* query查询（也可以理解为result结果）

简言之，首先你和msyql进行交互时，你的client端首先需要编码正确，然后你将编码正确的数据必须在能够运输正确的轨道上运输（connection）,最后要能把数据放在能够正确存储编码的表里。至此已经完了存储。接下来实际时读取，也是query过程，这个过程也要保证connection正确，然后就是返回的结果数据也要正确编码。

因此我在项目只设置了两项。

1. table表属性为 uft8mb4
2. [set names utf8mb4](https://dev.mysql.com/doc/refman/8.0/en/charset-connection.html) 

第二项等同于 ***character_set_client***, ***character_set_results***, ***character_set_connection*** 设置了这三项的值。

还有注意一点，就是mysql的一些属性的作用域只在当前会话中。因此，在实际使用过程中，需要在每次获取链接时，进行设置。有些三方的数据库链接池是有connectionInitSqls等属性支持的。

其实，任意编码问题都可以从这个方向进行梳理，数据元格式必须正确，传输过程必须支持，存储需要支持，读取也要按正确的编码进行读取。

