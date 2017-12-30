> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

获取网络适配器基本信息后，我们继续来获取适配器的一些高级信息。

接下来，先看一段代码：

``` c++
#include "pcap.h"
#include<cstdlib>
#include<iostream>

#ifndef WIN32
    #include <sys/socket.h>
    #include <netinet/in.h>
#else
    #include <winsock.h>
#endif


//Function prototypes
void ifprint(pcap_if_t *d);
char *iptos(u_long in);
char *ip6tos(struct sockaddr *sockaddr,char *address, int addrlen);

int main()
{
    pcap_if_t *alldevs;
    pcap_if_t *d;
    char errbuf[PCAP_ERRBUF_SIZE+1];
    char source[PCAP_ERRBUF_SIZE+1];

    printf("Enter the device you want to list:\n"
        "rpcap://==>lists interfaces in the local machine\n"
        "rpcap://hostname:port==>lists interfaces in a remote machine\n"
        "           (rpcapd damon must be up and running\n"
        "file://foldername==>lists all pcap files in the give folder\n\n"
        "Enter your choice: ");
    fgets(source,PCAP_ERRBUF_SIZE,stdin);
    std::cout<<"source:"<<source<<std::endl;
    source[PCAP_ERRBUF_SIZE] = '\0';

    //Retrieve the interfaces list
    if(pcap_findalldevs_ex(source,NULL,&alldevs,errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs: %s\n",errbuf);
        exit(1);
    }

    //scan the list printing every entry
    for(d=alldevs;d;d=d->next)
    {
        ifprint(d);
    }

    pcap_freealldevs(alldevs);

    system("PAUSE");
    return 1;
}


//Print all the available information on the given interface
void ifprint(pcap_if_t *d)
{
    pcap_addr_t *a;
    char ip6str[128];

    //Name
    printf("%s\n",d->name);

    //Description
    if(d->description)
        printf("\tDescription:%s\n",d->description);

    //Loopback Address
    printf("\td->flags:%s\n",d->flags);
    printf("\tLoopback: %s\n",(d->flags & PCAP_IF_LOOPBACK)?"yes":"no");

    //IP addresses
    for(a=d->addresses;a;a=a->next)
    {
        printf("\tAddress Family: #%d\n",a->addr->sa_family);

        switch(a->addr->sa_family)
        {
        case AF_INET:
            printf("\tAddress Family Name:AF_INET\n");
            if(a->addr)
                printf("\tAddress:%s\n",iptos(((struct sockaddr_in *)a->addr)->sin_addr.s_addr));
            if(a->netmask)
                printf("\tNetmask:%s\n",iptos(((struct sockaddr_in *)a->netmask)->sin_addr.s_addr));
            if(a->broadaddr)
                printf("\tBroadcast Address:%s\n",iptos(((struct sockaddr_in *)a->broadaddr)->sin_addr.s_addr));
            if(a->dstaddr)
                printf("\tDestination Address:%s\n",iptos(((struct sockaddr_in *)a->dstaddr)->sin_addr.s_addr));
            break;

        case AF_INET6:
            printf("\tAddress Family Name:AF_INET6\n");
            if(a->addr)
                printf("\tAddress:%s\n",ip6tos(a->addr,ip6str,sizeof(ip6str)));
            break;

        default:
            printf("\tAddress Family Name:Unknow\n");
            break;
        }
    }
    printf("\n");
}


//From tcptraceroute,convert a numeric IP address to a string
#define IPTOSBUFFERS 12
char *iptos(u_long in)
{
    static char output[IPTOSBUFFERS][3*4+3+1];
    static short which;
    u_char *p;

    p = (u_char *)&in;
    which = (which + 1 == IPTOSBUFFERS ? 0 : which +1);
    _snprintf_s(output[which],sizeof(output[which]),sizeof(output[which]),"%d.%d.%d.%d",p[0],p[1],p[2],p[3]);
    return output[which];
}

char *ip6tos(struct sockaddr *sockaddr,char *address,int addrlen)
{
    socklen_t sockaddrlen;

#ifdef WIN32
    sockaddrlen = sizeof(struct sockaddr_in6);
#else
    sockaddrlen = sizeof(struct sockaddr_storage);
#endif

    if(getnameinfo(sockaddr,
        sockaddrlen,
        address,
        addrlen,
        NULL,
        0,
        NI_NUMERICHOST) != 0) address = NULL;

    return address;
}
```

首先，如果不是windows系统，则包含两个库文件:sys/socket.h与netinet/in.h，windows系统则包含winsock.h。

接下来定义了三个函数原型，**ifprint**用来打印适配器的高级信息，**iptos**与**ip6tos**分别用来将ipv4与ipv6地址转换为字符串，这些函数会在后面详细介绍。

之后便是主函数，也是先定义两个**pcap_if_t**指针，指向适配器。字符数组**errbuf**用来存储错误信息。**source**用来指定读取数据包的“源头”，根据WinPcap语法规定，source为**'rpcap://'**时表示读取本地网络适配器的数据包信息，为**'rpcap://host:port'**表示读取名为host的远程主机的port端口上的数据包信息，为**'file://c:/myfolder/'**表示读取C盘下myfolder文件夹内pcap文件的数据包信息。

之后提示用户输入想要读取数据包的方式，存入source中，然后和上节一样，调用pcap_findalldevs_ex()函数获取本机适配器列表，然后调用ifprint()函数循环输出每一个适配器的详细信息，最后用pcap_freealldevs()函数释放适配器列表，接下来详细介绍ifprint()函数的内容。

ifprint()函数接受一个pcap_if_t类型的参数，读取各个适配器的名称、描述和地址。函数开头定义了一个类型为pcap_addr_t的指针，pcap_addr_t其实是一个pcap_addr的结构体，用来保存适配器的各种地址，定义如下：

``` c++
struct pcap_addr {
    struct pcap_addr *next;
    struct sockaddr *addr;
    struct sockaddr *netmask;
    struct sockaddr *broadaddr;
    struct sockaddr *dstaddr;
};
```

其中，**next**用来指向列表的下一个元素，如果为最后一个元素则next为空。后面四个都是**sockaddr**类型的指针变量，sockaddr是socket用来处理网络地址的结构体，还有一些其他用来处理网络地址的结构体定义如下：

``` c++
include <netinet/in.h>

// All pointers to socket address structures are often cast to pointers
// to this type before use in various functions and system calls:

struct sockaddr {
    unsigned short    sa_family;    // address family, AF_xxx
    char              sa_data[14];  // 14 bytes of protocol address
};


// IPv4 AF_INET sockets:

struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};

struct in_addr {
    unsigned long s_addr;          // load with inet_pton()
};


// IPv6 AF_INET6 sockets:

struct sockaddr_in6 {
    u_int16_t       sin6_family;   // address family, AF_INET6
    u_int16_t       sin6_port;     // port number, Network Byte Order
    u_int32_t       sin6_flowinfo; // IPv6 flow information
    struct in6_addr sin6_addr;     // IPv6 address
    u_int32_t       sin6_scope_id; // Scope ID
};

struct in6_addr {
    unsigned char   s6_addr[16];   // load with inet_pton()
};


// General socket address holding structure, big enough to hold either
// struct sockaddr_in or struct sockaddr_in6 data:

struct sockaddr_storage {
    sa_family_t  ss_family;     // address family

    // all this is padding, implementation specific, ignore it:
    char      __ss_pad1[_SS_PAD1SIZE];
    int64_t   __ss_align;
    char      __ss_pad2[_SS_PAD2SIZE];
};
```

> 这些struct的资料来自[Beej's Guide to Network Programming](http://www.retran.com/beej/index.html)

其中带有family的变量表示地址簇，通常为AF_INET(表示IPv4地址簇)或AF_INET6(表示IPv6地址簇)，其他就是一些端口、地址等的信息。

在pcap_addr结构体中，**addr**表示适配器的网络地址，**netmask**表示addr地址对应的掩码，**broadaddr**表示addr对应的广播地址，**dstaddr**表示addr对应的目的地址。在后面这些成员变量都会用到。

介绍完pcap_addr_t结构体后，定义了一个128字节的char数组，用来存储ipv6地址。接下来用d->name和d->description分别打印适配器的名称与描述，d->flags打印标志信息，用flags与PCAP_IF_LOOPBACK按位与来确定适配器是否是环回适配器。

接下来用一个for循环读取d->addresses里的适配器地址信息，a=a->next指向下一个地址信息，a为空时结束循环。循环体内，首先打印地址簇信息，如果是ipv4地址簇，则分别打印ipv4地址、掩码、广播地址和目的地址，如果是ipv6则打印ipv6地址：

``` c++
for(a=d->addresses;a;a=a->next)
    {
        printf("\tAddress Family: #%d\n",a->addr->sa_family);

        switch(a->addr->sa_family)
        {
        case AF_INET:
            printf("\tAddress Family Name:AF_INET\n");
            if(a->addr)
                printf("\tAddress:%s\n",iptos(((struct sockaddr_in *)a->addr)->sin_addr.s_addr));
            if(a->netmask)
                printf("\tNetmask:%s\n",iptos(((struct sockaddr_in *)a->netmask)->sin_addr.s_addr));
            if(a->broadaddr)
                printf("\tBroadcast Address:%s\n",iptos(((struct sockaddr_in *)a->broadaddr)->sin_addr.s_addr));
            if(a->dstaddr)
                printf("\tDestination Address:%s\n",iptos(((struct sockaddr_in *)a->dstaddr)->sin_addr.s_addr));
            break;

        case AF_INET6:
            printf("\tAddress Family Name:AF_INET6\n");
            if(a->addr)
                printf("\tAddress:%s\n",ip6tos(a->addr,ip6str,sizeof(ip6str)));
            break;

        default:
            printf("\tAddress Family Name:Unknow\n");
            break;
        }
    }
```

其中，打印ipv4地址时需要将sockaddr结构强制转换为**sockaddr_in**结构，读取到地址信息后再通过iptos()函数转换为常见的**点分十进制**表示，其中sockaddr_in结构体的定义上面已经给出。打印ipv6地址时，需通过ip6tos()函数转换为ipv6地址表示，存入ip6str字符数组后打印出来。

iptos()函数通过[_snprintf_s()函数](https://msdn.microsoft.com/en-us/library/f30dzcf6.aspx)将ipv4地址转换为点分十进制表示，ip6tos()函数通过[getnameinfo()函数](https://msdn.microsoft.com/en-us/library/windows/desktop/ms738532%28v=vs.85%29.aspx)将ipv6地址转换为主机名。

运行程序，在我的计算机上得到如下结果：![Sniffer_ObtainingAdvancedInformationAboutInstalledDevices](https://raw.githubusercontent.com/Heatwave/Blog/master/images/ObtainingAdvancedInformationAboutInstalledDevices.JPG)

获取了适配器的高级信息后，我们终于可以开始捕获数据包了！！！下一篇：[打开适配器并捕获数据包](open-adapter-and-sniff.md)
