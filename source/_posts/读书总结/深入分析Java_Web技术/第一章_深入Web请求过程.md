---
title: 一、深入Web请求过程
date: 2017-08-03 12:08:00
updated: 2018-12-19 13:00:00
tags:
  - javaweb
categories: 
  - [读书总结, 深入分析Java_Web技术]
comments: true
permalink: deepknowjavaweb/1_web.html  
---

![][0]

<!-- more -->

# 1 DNS域名解析

使用浏览器输入网址后，浏览器会检查缓存对应的IP地址，如果没有，浏览器会查找操作系统，即host文件。 所以很多墙外比较慢的网址，可以手动编写host文件对应的IP地址以及对应的网址，可以加快访问速度。 如果实在没有就发送给LDNS，这个LDNS在不同的情况是不一样的，在学校，大部分都是学校的DNS服务器， 家庭的一般都是联通或者电信的DNS服务器，最最最后实在解析不出来，就抛给Root Server域名服务器， 它会返回给本地域名服务器的主域名服务器的地址，即域名空间提供商的域名解析服务器，就像阿里域名解析加速。

# 2 清除缓存的域名
主要在两个地方缓存：Local DNS Server, 另一个是用户的本机，当然，重启也是更好的方法。

`ipconfig /flushdns`

在java中，JVM也会缓存DNS的解析结果，分两种，即正确的解析结果，以及错误的解析结果，InetAddress，实际中 InetAddress 使用必须是单例模式，因为每次创建 InetAddress 实例都要进行一次完整的域名解析。

# 3 CDN工作机制
CDN也就是内容分布网络(Content Delivery Network)。通过在现有的Internet中增加一层新的网络架构，比镜像更智能。 比喻：CDN=镜像Mirror+缓存Cache+整体负载均衡GSLB。 目前CDN都以缓存网站中的静态数据为主，如CSS、JS、图片和静态页面等，用户在从主站服务器请求到动态内容后，再从CDN上下载这些静态数据。 
![Web请求过程][1]


# 4 负载均衡

负载均衡(Load Balance)就是对工作任务进行平衡、分摊到多个操作单元上执行，如图片服务器、应用服务器等，提高服务器响应速度，实现地理位置无关性。 通常有三种负载均衡架构：链路负载均衡、集群负载均衡、操作系统负载均衡。

链路：用户最终访问哪个Web Server是由DNS Server来控制的，优点在于用户直接访问目标服务器，不需要经过其它的代理服务器，通常访问速度更快，缺点在于DNS在用户本地和LDNS都有缓存，一旦某台Web Server挂掉，就难及时更新用户的域名解析结构。
集群：硬件负载以及软件负载均衡，前者需要贵的硬件作为中心，而软件则是成本低，但是需要多次代理服务器转发，从而增加了网络延时。 
![集群负载均衡][2]

操作系统：如设置多队列网卡。

# 5 CDN动态加速
原理在于CDN的DNS解析中通过动态的链路探测来寻找回源最好的一条路径，通过DNS的调度将所有请求到选定的路径上回源，一个简单的原则就是在每个CDN节点上从源站下载一个一定大小文件，看哪个链路的总耗时最短，这样可以构成一个链路列表，然后绑定到DNS解析上，更新到CDN的Local DNS。以及网络成本等。

# 6 总结

1. 域名的请求处理
2. CDN=镜像Mirror+缓存Cache+整体负载均衡GSLB

[0]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/background/2018-12-16%E5%85%AC%E5%8F%B8%E7%A4%BC%E7%89%A9%E7%9B%92.jpg
[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech_1_1.png
[2]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech_1_2.png