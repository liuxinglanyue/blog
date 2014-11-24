---
layout: post
category: "sql"
title:  "Oracle修改字段类型"
tags: [oracle,字段]
---

由于业务的需要，我们需要将身高、体重修改为String类型，这里简要记录下操作方法，现在主要有两种方案。

1、如果没有数据可直接采用下面语句修改即可
<pre class="prettyPrint">
alter table T_CHARACTER_CONTENT modify HEIGHT varchar2(50);
</pre>

2、但是有数据的话，就不能用上面的方法了(我采用此方案)
<pre class="prettyPrint">
alter table T_CHARACTER_CONTENT add HEIGHT_temp varchar2(50);
update T_CHARACTER_CONTENT set HEIGHT_temp=HEIGHT;
alter table T_CHARACTER_CONTENT drop column HEIGHT;
alter table T_CHARACTER_CONTENT rename column HEIGHT_temp to HEIGHT;
comment on column T_CHARACTER_CONTENT.HEIGHT is '身高';
</pre>

这种方法会使列名发生变化,而且字段顺序增加 有可能发生行迁移,对应用程序会产生影响
以下方法是比较好的方法
不用使列名发生变化 也不会发生表迁移,但这个有个缺点是表要更新两次
如果数据量较大的话 产生的undo和redo更多 ,前提也是要停机做
要是不停机的话 ,也可以采用在线重定义方式来做

以下是脚本:
<pre class="prettyPrint">
alter table T_CHARACTER_CONTENT add HEIGHT_temp varchar2(50);
--Add/modify columns
alter table T_CHARACTER_CONTENT modify HEIGHT null;
update T_CHARACTER_CONTENT set HEIGHT_temp=HEIGHT,HEIGHT=null;
commit;
alter table T_CHARACTER_CONTENT modify HEIGHT varchar2(50);
update T_CHARACTER_CONTENT set HEIGHT=HEIGHT_temp,HEIGHT_temp=null;
commit;
alter table T_CHARACTER_CONTENT drop column HEIGHT_temp;
alter table T_CHARACTER_CONTENT modify HEIGHT not null;
select * from T_CHARACTER_CONTENT;
<pre>
