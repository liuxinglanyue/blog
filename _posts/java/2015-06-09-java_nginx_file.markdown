---
layout: post
category: "java"
title:  "Java配合nginx实现文件下载、显示的权限控制"
tags: [java,nginx,file,access]
---

本文参考[http://www.iitshare.com/java-download-files-with-nginx.html](http://www.iitshare.com/java-download-files-with-nginx.html)

文章标题采用的是和参考资料一样的标题（赞）

nginx配置如下：

<pre class="prettyPrint">
location /log/ {
    #禁止浏览器直接访问
    internal;
    limit_rate 200k;
    alias /root/data/;
    error_page 404 =200 @backend;
}
location @backend {
    proxy_pass http://10.222.138.16:8086/openapi/open$request_uri;
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
location /openapi/ {
    proxy_pass http://10.222.138.16:8086/$request_uri;
}
</pre>

Java代码如下：

<pre class="prettyPrint">
//httpResponse.setHeader("Content-Disposition", "attachment; filename=\""+filename+"\"");
//httpResponse.setHeader("Content-Type", "application/octet-stream");
//httpResponse.setHeader("X-Accel-Redirect", "/log/"+url);
HttpHeaders header = new HttpHeaders();
header.add("Content-Disposition", "attachment; filename=\""+filename+"\"");
header.add("Content-Type", "application/octet-stream");
header.add("X-Accel-Redirect", "/log/"+url);
</pre>

请求地址为：

http://10.222.138.180/log/download?access_token=a68cafa8b80f08c7db2af5b654435dd2

日志文件在目录/root/data下

通过wget下载时可以加上参数--content-disposition，下载的文件名就是Java代码设置的filename（对wget版本要求 >= 1.15）

![java-nginx](/img/java/java-nginx.png)


