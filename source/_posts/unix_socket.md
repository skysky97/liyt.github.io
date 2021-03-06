---
title: Socket连接数问题
date: 2018-02-27 15:04:33
tags: [socket, TCP/IP, unix]
categories: unix
---
# Socket连接数问题
终于理清楚了Socket连接数与端口号的问题，特此记录，以后不要再犯浑。  
  
先说**结论**：  
1. 一个socket连接由客户端IP，客户端PORT，服务端IP，服务端PORT和协议版本5各参数决定，任意参数不同就不是同一个连接。  
2. 一个端口只能被一个socket监听；
  
下面针对我的愚蠢问题做说明。  
  
Q1. Socket连接数是不是受到端口数（最大65536）限制？ 
对于Server来说，创建socket对象，绑定**一个**端口，开始监听，等待客户端连接。看到加粗部分了吗，一个socket对象总是监听一个端口，无论后面接受多少个连接，监听的端口都不变，变的是客户端的IP和端口。所以实际可以接受的连接数仅仅受到客户端IP和客户端端口的限制（忽略系统资源等限制），考虑到IP地址的数量级，这个连接数可以是非常大的。  
  
Q2. 一个端口只能被一个socket监听，那多个浏览器是怎么同时使用80端口的？  
这里又犯了个很明显的错误，一个端口只能被一个socket监听是对于服务端而言的，而浏览器作为客户端其源端口号是动态分配的，只有远程端口才是80，所以不存在冲突问题。只有当两个服务器程序需要同时监听80端口时才会出现冲突。  

