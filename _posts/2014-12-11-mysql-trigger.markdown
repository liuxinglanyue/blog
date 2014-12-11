---
layout: post
category: "sql"
title:  "MySQL的触发器(将数据按照5分钟进行归并)"
tags: [mysql,sql,trigger]
---

markdown中下划线需要通过\转义
-------------------------------------------------------------------------------
申明：
MySQL版本 mysql  Ver 14.14 Distrib 5.1.73, for redhat-linux-gnu (x86\_64) using readline 5.1
sql中的record\_time只到分钟。

首先贴一下sql
<pre class="prettyPrint">
DROP TRIGGER IF EXISTS trig\_expresslane\_service\_flow;
DELIMITER //
CREATE TRIGGER trig\_expresslane\_service\_flow
AFTER INSERT on expresslane\_service\_flow
FOR EACH ROW
BEGIN
	declare five\_len int;
	declare five\_date datetime;
	declare five\_unique varchar(255);
	declare five\_id bigint(19);
	declare five\_access\_rate bigint(20);
	declare five\_service\_flow\_rate bigint(20);
	declare five\_total\_back\_flow\_rate bigint(20);
	declare five\_back\_flow\_rate bigint(20);
	set @five\_len=(MINUTE(NEW.record\_time) % 5) * -1;
	set @five\_date=date\_add(NEW.record\_time, interval @five\_len minute);
	set @five\_unique=CONCAT\_WS(',',@five\_date, NEW.service\_type, NEW.device\_idn, NEW.app\_id, NEW.domain);
	select id, access\_rate, service\_flow\_rate, total\_back\_flow\_rate, back\_flow\_rate into @five\_id, @five\_access\_rate, @five\_service\_flow\_rate, @five\_total\_back\_flow\_rate, @five\_back\_flow\_rate from expresslane\_service\_flow\_5 where unique\_field = @five\_unique;
	if @five\_id > 0 then
		set @five\_service\_flow\_rate = @five\_service\_flow\_rate + NEW.service\_flow\_rate;
		set @five\_total\_back\_flow\_rate = @five\_total\_back\_flow\_rate + NEW.total\_back\_flow\_rate;
		set @five\_back\_flow\_rate = @five\_back\_flow\_rate + NEW.back\_flow\_rate;
		set @five\_access\_rate = @five\_access\_rate + NEW.access\_rate;
		update expresslane\_service\_flow\_5 t set t.access\_rate=@five\_access\_rate, t.service\_flow\_rate=@five\_service\_flow\_rate, t.total\_back\_flow\_rate=@five\_total\_back\_flow\_rate, t.back\_flow\_rate=@five\_back\_flow\_rate where t.id = @five\_id;
	else
		insert into expresslane\_service\_flow\_5 (record\_time,service\_type,device\_idn,domain,app\_id,access\_rate,service\_flow\_rate,total\_back\_flow\_rate,back\_flow\_rate,unique\_field)
			values(@five\_date,NEW.service\_type,NEW.device\_idn,NEW.domain,NEW.app\_id,NEW.access\_rate,NEW.service\_flow\_rate,NEW.total\_back\_flow\_rate,NEW.back\_flow\_rate,@five\_unique);
	end if;
END//
DELIMITER //
</pre>

主要说一下对时间的处理

对5分钟的处理如下：
<pre class="prettyPrint">
set @five\_len=(MINUTE(NEW.record\_time) % 5) * -1;
set @five\_date=date\_add(NEW.record\_time, interval @five\_len minute);
</pre>
先获取通过mysql函数MINUTE获取record\_time的分钟值，然后对5求余数，同时 * -1
通过date\_add函数将record\_time和five\_len相加。

举例：record\_time=2014-05-06 17:48:00 
(48 % 5) * -1 = -3
date\_add(record\_time, interval -3 minute) = 2014-05-06 17:45:00 

