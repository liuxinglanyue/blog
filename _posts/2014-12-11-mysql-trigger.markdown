---
layout: post
category: "sql"
title:  "MySQL的触发器(将数据按照5分钟进行归并)"
tags: [mysql,sql,trigger]
---
申明：
MySQL版本 mysql  Ver 14.14 Distrib 5.1.73, for redhat-linux-gnu (x86_64) using readline 5.1
sql中的record_time只到分钟。

首先贴一下sql
<pre class="prettyPrint">
DROP TRIGGER IF EXISTS trig_expresslane_service_flow;
DELIMITER //
CREATE TRIGGER trig_expresslane_service_flow
AFTER INSERT on expresslane_service_flow
FOR EACH ROW
BEGIN
	declare five_len int;
	declare five_date datetime;
	declare five_unique varchar(255);
	declare five_id bigint(19);
	declare five_access_rate bigint(20);
	declare five_service_flow_rate bigint(20);
	declare five_total_back_flow_rate bigint(20);
	declare five_back_flow_rate bigint(20);

	set @five_len=(MINUTE(NEW.record_time) % 5) * -1;
	set @five_date=date_add(NEW.record_time, interval @five_len minute);
	
	set @five_unique=CONCAT_WS(',',@five_date, NEW.service_type, NEW.device_idn, NEW.app_id, NEW.domain);
	
	select id, access_rate, service_flow_rate, total_back_flow_rate, back_flow_rate into @five_id, @five_access_rate, @five_service_flow_rate, @five_total_back_flow_rate, @five_back_flow_rate from expresslane_service_flow_5 where unique_field = @five_unique;

	if @five_id > 0 then
		set @five_service_flow_rate = @five_service_flow_rate + NEW.service_flow_rate;
		set @five_total_back_flow_rate = @five_total_back_flow_rate + NEW.total_back_flow_rate;
		set @five_back_flow_rate = @five_back_flow_rate + NEW.back_flow_rate;
		set @five_access_rate = @five_access_rate + NEW.access_rate;
		update expresslane_service_flow_5 t set t.access_rate=@five_access_rate, t.service_flow_rate=@five_service_flow_rate, t.total_back_flow_rate=@five_total_back_flow_rate, t.back_flow_rate=@five_back_flow_rate where t.id = @five_id;
	else
		insert into expresslane_service_flow_5 (record_time,service_type,device_idn,domain,app_id,access_rate,service_flow_rate,total_back_flow_rate,back_flow_rate,unique_field)
			values(@five_date,NEW.service_type,NEW.device_idn,NEW.domain,NEW.app_id,NEW.access_rate,NEW.service_flow_rate,NEW.total_back_flow_rate,NEW.back_flow_rate,@five_unique);
	end if;
END//
DELIMITER //
</pre>

主要说一下对时间的处理

对5分钟的处理如下：
<pre class="prettyPrint">
set @five_len=(MINUTE(NEW.record_time) % 5) * -1;
set @five_date=date_add(NEW.record_time, interval @five_len minute);
</pre>
先获取通过mysql函数MINUTE获取record_time的分钟值，然后对5求余数，同时 * -1
通过date_add函数将record_time和five_len相加。

举例：record_time=2014-05-06 17:48:00 
(48 % 5) * -1 = -3
date_add(record_time, interval -3 minute) = 2014-05-06 17:45:00 

