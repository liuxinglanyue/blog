---
layout: post
category: "sql"
title:  "MySQL的触发器(将数据按照5分钟进行归并)"
tags: [mysql,sql,trigger]
---
申明：
MySQL版本 mysql  Ver 14.14 Distrib 5.1.73, for redhat-linux-gnu (x86_64) using readline 5.1
sql中的record_time只到分钟。



举例：record_time=2014-05-06 17:48:00 
(48 % 5) * -1 = -3
date_add(record_time, interval -3 minute) = 2014-05-06 17:45:00 

