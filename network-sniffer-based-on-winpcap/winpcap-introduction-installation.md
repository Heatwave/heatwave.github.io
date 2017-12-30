> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[hinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)
#1.WinPcap简介

WinPcap是一个基于Win32平台的，用于捕获网络数据包并进行分析的开源库.

大多数网络应用程序通过被广泛使用的操作系统元件来访问网络，比如sockets。  这是一种简单的实现方式，因为操作系统 已经妥善处理了底层具体实现细节（比如协议处理，封装数据包等等），并且提供了一个与读写文件类似的，令人熟悉的接口。

然而，有些时候，这种“简单的方式”并不能满足任务的需求，因为有些应用程序需要直接访问网 络中的数据包。也就是说，那些应用程序需要访问原始数据包，即没有被操作系统利用网络协议处理过的数据包。

WinPcap产生的目的，就是为Win32应用程序提供这种访问方式； WinPcap提供了以下功能
- 捕获原始数据包，无论它是发往某台机器的，还是在其他设备（共享媒介）上进行交换的
- 在数据包发送给某应用程序前，根据用户指定的规则过滤数据包
- 将原始数据包通过网络发送出去
- 收集并统计网络流量信息

以上这些功能需要借助安装在Win32内核中的网络设备驱动程序才能实现，再加上几个动态链接库DLL。

所有这些功能都能通过一个强大的编程接口来表现出来,易于开发，并能在不同的操作系统上使用。

WinPcap可以被用来制作许多类型的网络工具，比如具有分析，解决纷争，安全和监视功能的工具。特别地，一些基于WinPcap的典型应用有：
- 网络与协议分析器 (network and protocol analyzers)
- 网络监视器 (network monitors)
- 网络流量记录器 (traffic loggers)
- 网络流量发生器 (traffic generators)
- 用户级网桥及路由 (user-level bridges and routers)
- 网络入侵检测系统 (network intrusion detection systems (NIDS))
- 网络扫描器 (network scanners)
- 安全工具 (security tools)

WinPcap能 独立地 通过主机协议发送和接受数据，如同TCP-IP。 这就意味着WinPcap不能阻止、过滤或操纵同一机器上的其他应用程序的通讯： 它仅仅能简单地"监视"在网络上传输的数据包。所以，它不能提供类似网络流量控制、服务质量调度和个人防火墙之类的支持。
#2.安装WinPcap驱动+DLLs+开发包

如果之前安装过Wireshark，那应该已经装过WinPcap了，这步可以跳过。

接下来到[这里](http://www.winpcap.org/install/default.htm)下载Version 4.1.3 Installer for Windows
![WinPcap_Download.png](https://raw.githubusercontent.com/Heatwave/Blog/master/images/WinPcap_Download.png)

然后直接Next-Next安装就行了。接下来在[这里](http://www.winpcap.org/devel.htm)下载WinPcap 4.1.2 Developer's Pack也就是WinPcap开发包，注意这里开发包版本是4.1.2的，但和4.1.3的WinPcap库是兼容的
![WinPcap_Developer_Pack](https://raw.githubusercontent.com/Heatwave/Blog/master/images/WinPcap%20Developer%20Resources.png)

下载解压之后的目录结构如下：
- docs
- Example-pcap
- Examples-remote
- Inlcude
- Lib

其中docs为官方英文文档，[这里](http://www.ferrisxu.com/WinPcap/html/index.html)是中文文档。

Example-pcap和Examples-remote目录下为示例代码，两个目录的区别在于前者采用的是与libpcap库接口兼容的示例代码，后者采用的是wpcap.dll库接口的示例代码。

Include目录下为基于WinPcap库进行开发所需的头文件。

Lib目录下为基于WinPcap库进行开发所需的库文件。

下载安装好后，就可以开始创建项目了！下一篇： [创建VS2010项目与配置](set-up-project.md)
