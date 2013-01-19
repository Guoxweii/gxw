---
layout: post
title: 在 Ubuntu rails服务器上实现抓包流量统计功能
date: 2013-01-15
categories: ruby
tags: tcpdump packetfu rails ubuntu
---


## 需求说明

 服务器分享代理功能给服费用户，给用户分配一个域名和端口，根据端口和域名来统计用户流量
 
## 暂时需要完善的地方
 
 因为packetfu插件，解析packet没有函数可以取到host的值，所以暂时只能通过端口来映射用户
 
## 功能实现流程

 下面介绍功能实现的流程 抓包 -> 存pcap文件 -> ruby解析pcap文件
 
### 1.抓包

主要利用的内库是linux的tcpdump，

命令：

sudo tcpdump -i en0 -s 0 -G 60 -w /Users/gxw/Documents/fils/log/%F-%H-%M-%S.pcap

命令解析：抓取网卡en0上的流量，每隔60秒将抓取的包写入时间格式命名的pcap文件中

需要在后台开启一个进程，实时运行这个程序

参数解析：

1. -i 指定监听的端口，如果没有写就是监听全部的端口
2. -s 指定每次抓包文件的大小
3. -G 时间间隔，配合-w使用 
4. -w 将数据流存入文件

ps：ruby也有gem包可以实现同样地功能 packetfu pcaprub pcap

   pcap是1.8时代的，现在已经被淘汰
   
   pcaprub插件
   
   参考：https://github.com/shadowbq/pcaprub
   
   pcapfu插件
   
   参考：http://ruby-doc.org/gems/docs/p/packetfu-1.1.5/PacketFu/Capture.html
   
  上面的插件包都能够实时抓取数据，但没有深入研究
 
### 2.解析pcap文件

主要利用的是ruby的packetfu插件

用法：

参考地址：http://ruby-doc.org/gems/docs/p/packetfu-1.1.5/PacketFu/PcapFile.html

	PacketFu::PcapFile.read_packets(file_path + file_name) do |packet|
      puts packet.methods
      puts packet.protocol
    end

PacketFu::PcapFile.read_packets函数传入pcap文件路径，以及闭包内容.

闭包的参数packet是pcap文件的一个个packet数据包，查看api来取得你所需要的数据

packet的api文档： http://ruby-doc.org/gems/docs/p/packetfu-1.1.5/PacketFu/Packet.html

ps：pcaprub插件同样有这样的功能，但是没有接口可以直接查看包文件的端口号，就没有继续研究下取了 
   
   


 

