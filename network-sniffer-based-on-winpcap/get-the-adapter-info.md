> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

创建项目与配置好后，就可以开始使用WinPcap进行开发了！
#1.获取设备列表

接下来，先看一段代码：

``` c++
#include "pcap.h"

int main()
{
    pcap_if_t *alldevs;
    pcap_if_t *d;
    int i = 0;
    char errbuf[PCAP_ERRBUF_SIZE];

    //获取本地机器设备列表
    if(pcap_findalldevs_ex(PCAP_SRC_IF_STRING,NULL /*auth is not needed*/,&alldevs,errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs_ex:%s\n",errbuf);
        exit(1);
    }

    //打印列表
    for(d=alldevs;d!=NULL;d=d->next)
    {
        printf("%d.name:%s\n",++i,d->name);
        if(d->description)
            printf("description:(%s)\n",d->description);
        else
            printf("(No description available)\n");
    }

    if(i == 0)
    {
        printf("\nNo interfaces found!Make sure WinPcap is installed.\n");
        return 0;
    }

    //不再需要设备列表，释放它
    pcap_freealldevs(alldevs);

    system("pause");
    return 0;
}
```

首先，包含**pcap.h**头文件，这是必须的。然后定义了两个pcap_if_t的指针，用来指向每一个**网络适配器**(也就是网卡)，pcap_if_t其实是一个**pcap_if**的结构体，定义如下：

``` c++
struct pcap_if {
    struct pcap_if *next;
    char *name;     /* name to hand to "pcap_open_live()" */
    char *description;  /* textual description of interface, or NULL */
    struct pcap_addr *addresses;
    bpf_u_int32 flags;  /* PCAP_IF_ interface flags */
};
```

其中**next**用来指向下一个适配器，用来构成一个链表，每一个节点都是一个**pcap_if**的结构，表示一个适配器。**name**是适配器的名称，可以用来传递给pcap_open_live()函数。**description**是适配器的详细描述，这个成员变量可能为空。**addresses**是一个**pcap_addr**类型的结构体，定义了适配器的一些网络地址，在下面会详细介绍。**flags**是PCAP_IF_接口标志，目前唯一可能的标志为PCAP_IF_LOOPBACK，也就是在适配器为环回适配器时才设置为该值。

接下来一个char数组**errbuf**用来写入错误信息，长度为固定的PCAP_ERRBUF_SIZE。

接着就调用pcap_findalldevs_ex()函数获取本机适配器列表:

``` c++
    if(pcap_findalldevs_ex(PCAP_SRC_IF_STRING,NULL /*auth is not needed*/,&alldevs,errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs_ex:%s\n",errbuf);
        exit(1);
    }
```

pcap_findalldevs_ex()函数原型如下：

``` c++
int pcap_findalldevs_ex(
    char *  source,
    struct pcap_rmtauth *   auth,
    pcap_if_t **    alldevs,
    char *  errbuf
)
```

函数第一个参数**source**表示从哪一个源读取适配器信息，我这里设置的是常量PCAP_SRC_IF_STRING，也就是rpcap://，表示读取**本机**适配器信息。如果是rpcap://host:port则表示读取**远程主机**的适配器信息，是file://c:/myfolder/则表示读取某个目录下的**pcap文件**。

第二个参数**auth**是连接远程主机时请求授权的，我们是在本地嗅探的就用不着了，设置为NULL就行。

第三个参数**alldevs**类型就是我们上面介绍的pcap_if结构体，把我们上面定义的alldevs的地址传进去，如果函数返回0，一切正常的话，alldevs将会指向适配器列表的第一个元素，列表的每一个元素都是pcap_if结构，每一个元素表示一个适配器。

第四个参数**errbuf**用来存储错误信息。

函数的**返回值**为0表示一切正常，返回值为-1表示有错误发生，如果返回值为-1我们就输出errbuf里的错误信息，并以exit(1)退出程序。

接下来用一个相同结构的指针d指向alldevs，循环打印每一个适配器的name和description:

``` c++
    for(d=alldevs;d!=NULL;d=d->next)
    {
        printf("%d.name:%s\n",++i,d->name);
        if(d->description)
            printf("description:(%s)\n",d->description);
        else
            printf("(No description available)\n");
    }
```

注意description有可能为空，要判断一下。然后d=d->next指向下一个适配器，直到d为空。

在循环的时候每读到一个适配器上面定义的i都会加1，如果i为0的话，表示读不到任何一个适配器，于是输出错误信息然后返回0:

``` c++
    if(i == 0)
    {
        printf("\nNo interfaces found!Make sure WinPcap is installed.\n");
        return 0;
    }
```

最后调用pcap_freealldevs()函数释放pcap_findalldevs_ex()函数获得的适配器信息，也就是alldevs指向的内容:

``` c++
    pcap_freealldevs(alldevs);
```

这样就结束了！

运行一下程序，得到如下结果：![Sniffer_ObtainingTheDeviceList](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_ObtainingTheDeviceList.jpg)

获取了适配器的基本信息后，我们继续获取一些适配器的高级信息。下一篇：[获取适配器高级信息](get-the-adapter-advanced-info.md)
