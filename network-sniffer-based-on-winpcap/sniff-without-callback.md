> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

pcap_loop()函数是基于回调的原理来进行数据捕获，这是一种精妙的方法，并且在某些场合中，它是一种很好的选择。 然而，处理回调有时候并不实用 -- 它会增加程序的复杂度，特别是在拥有多线程的C++程序中。

因此这里我们使用pcap_next_ex()函数来读取数据包，源代码如下：

``` C++
#include "pcap.h"
#include <iostream>

int main()
{
    pcap_if_t *alldevs;
    pcap_if_t *d;
    int inum;
    int i=0;
    pcap_t *adhandle;
    int res;
    char errbuf[PCAP_ERRBUF_SIZE];
    struct tm ltime;
    char timestr[16];
    struct pcap_pkthdr *header;
    const u_char *pkt_data;
    time_t local_tv_sec;

    //Retrieve the device list on the local machine
    if(pcap_findalldevs_ex(PCAP_SRC_IF_STRING,NULL,&alldevs,errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs: %s\n",errbuf);
        exit(1);
    }

    //Print the list
    for(d=alldevs;d;d=d->next)
    {
        printf("%d. %s",++i,d->name);
        if(d->description)
            printf("(%s)\n",d->description);
        else
            printf("(No description available)\n");
    }

    if(i==0)
    {
        printf("\nNo interfaces found!Make sure WinPcap is installed.\n");
        return -1;
    }

    printf("Enter the interface numer(1-%d):",i);
    scanf_s("%d",&inum);

    if(inum<1 || inum>i)
    {
        printf("\nInterface number out of range.\n");
        //Free the device list
        pcap_freealldevs(alldevs);
        return -1;
    }

    //Jump to the selected adapter
    for(d=alldevs,i=0;i<inum-1;d=d->next,i++);

    //Open the device
    if( (adhandle=pcap_open(d->name,65536,PCAP_OPENFLAG_PROMISCUOUS,1000,NULL,errbuf)) == NULL)
    {
        fprintf(stderr,"\nUnable to open the adapter.%s is not supported by WinPcap\n",d->name);
        //Free the device list
        pcap_freealldevs(alldevs);
        return -1;
    }

    printf("\nlistening on %s...\n",d->description);

    //At this point,we don't need any more the device list.Free it
    pcap_freealldevs(alldevs);

    //Retrieve the packets
    while((res = pcap_next_ex(adhandle,&header,&pkt_data)) >= 0)
    {
        if(res == 0)
        {
            printf("Timeout elapsed!\n");
            continue;
        }

        //convert the timestamp to readable format
        local_tv_sec = header->ts.tv_sec;
        localtime_s(&ltime,&local_tv_sec);
        strftime(timestr,sizeof timestr,"%H:%M:%S",&ltime);

        std::cout<<"header->caplen:"<<header->caplen<<" ";
        printf("%s,%.6d len:%d\n",timestr,header->ts.tv_usec,header->len);
    }

    if(res == -1)
    {
        printf("Error reading the packets:%s\n",pcap_geterr(adhandle));
        return -1;
    }

    return 0;
}
```

大部分的代码都与上一节的一样，只有在读取数据包时使用的函数不同，上一节使用pcap_loop()函数，这里我们使用pcap_next_ex()函数。**pcap_next_ex()**函数原型如下：

``` C++
int pcap_next_ex    (   pcap_t *    p,
                        struct pcap_pkthdr **   pkt_header,
                        const u_char **     pkt_data     
)   
```

这三个参数前面基本上都介绍过了，其中**p**是捕捉实例的描述符，**pkt_header**包含数据包的头部信息，**pkt_data**包含数据包的原始数据。

pcap_next_ex()函数的返回值类型是int，这个返回值比较有用，可能是以下几个之一：
- return 1，成功，No problem！
- return 0，超时！超过**pcap_open()**或**pcap_open_live()**函数设置的时间，这种情况下pkt_header和pkt_data不会指向有效的数据包。
- return -1，错误发生！
- return -2，读取脱机文件时到达文件尾EOF。

因此这里只要pcap_next_ex()的返回值大于0时我们就循环调用pcap_next_ex()，当返回值等于0时就打印超时，否则就和上一节一样，打印每个有效数据包的时间戳与长度，代码与上一节基本一致。

运行程序，选择适配器：![CapturingThePacketsWithoutTheCallback](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_CapturingThePacketsWithoutTheCallback0.JPG)

结果也和上一节差不多：![CapturingThePacketsWithoutTheCallback](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_CapturingThePacketsWithoutTheCallback1.JPG)

这一节与上一节我们介绍了两种捕获数据包的方法，一种回调方法和一种非回调方法，具体什么场景使用哪种方法我也不太清楚，希望知道的人能留言告诉我一下.......

这两节我们捕获到的数据包是所有流经我们网卡的数据包，如果我们需要根据特定的规则只捕获某一种或几种类型的数据包该怎么办呢？请看下节：[过滤数据包](filter-network-packet.md)
