---
layout: default
title: "开源：open source"
---


1. ####MySQL数据迁移工具(经过现网验证)

	[migration-tool](https://github.com/liuxinglanyue/migration-tool)

	支持指定表名、列名，多线程+多进程。

	保证高可用，数据一致性。
	
2. ####java class热替换(实验室)
	
	[hot2hot](https://github.com/liuxinglanyue/hot2hot)

	阿里现有开源项目hotcode2(可应用于生产环境）
	
	hot2hot与hotcode2实现方式不同，hot2hot采用的是整个替换class

3. ####计算Portal数据获取最大带宽值以及对应时间(现网验证)(go语言)
	
	[analyze_portal](https://github.com/liuxinglanyue/analyze_portal)

	为了更好地做流量调度，需要查看每个省市的各个ISP的带宽使用最大值
	
	第一次用go做，界面比较粗糙

4. ####分布式session管理(未完成)
	
	[distributed-session-manager](https://github.com/liuxinglanyue/distributed-session-manager)

	基于Tomcat7，支持多种session持久化方案

5. ####将java线程绑定到具体的cpu上执行
	
	[threadBandCpu](https://github.com/liuxinglanyue/threadBandCpu)
	
	通过JNI调用，使用pthread实现cpu的绑定
	
6. ####SSH免密码自动登录(fork后添加功能)

	[ssh-auto-login-manage](https://github.com/liuxinglanyue/ssh-auto-login-manage)
	
	在原先的基础上加了两个功能，1.自定义端口 2.添加描述信息，帮助记忆

7. ####基于预测模型的缓存(未开动)
	
	[PreCache](https://github.com/liuxinglanyue/PreCache)

	目的就是提前缓存用户查询的数据

8. ####爬取阿里巴巴国际站上商家信息(给老婆用的小工具)
	
	[Carry](https://github.com/liuxinglanyue/Carry)

	采用httpclient获取并分析商家信息，生成excel文件
	
9. ####Mtr、Traceroute工具(不需要安装mtr、traceroute)

	[mtr](https://github.com/liuxinglanyue/mtr)
	
	采用Go语言开发，协议有UDP、ICMP。支持实时返回一次探测的数据

<!-- Blog Comments -->
<div class="media">
  <!-- UY BEGIN -->
  <div id="uyan_frame">
  </div>
  <script type="text/javascript" src="http://v2.uyan.cc/code/uyan.js?uid=1988228">
  </script>
  <!-- UY END -->
</div>