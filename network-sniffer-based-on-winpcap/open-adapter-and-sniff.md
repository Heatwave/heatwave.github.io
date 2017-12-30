> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

终于要开始编写嗅探器最核心的代码了！——打开适配器并捕获数据包：

``` c++
#include "pcap.h"
#include <iostream>

//prototype of the packet handler
void packet_handler(u_char *param,const struct pcap_pkthdr *header,const u_char *pkt_data);

int main()
{
    pcap_if_t *alldevs;
    pcap_if_t *d;

    int inum;
    int i=0;
    pcap_t *adhandle;

    char errbuf[PCAP_ERRBUF_SIZE];

    //Retrieve the device list on the local machine
    if(pcap_findalldevs_ex(PCAP_SRC_IF_STRING,NULL,&alldevs,errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs: %s\n",errbuf);
        exit(1);
    }

    //Print the list
    for(d=alldevs;d;d=d->next)
    {
        printf("%d.%s",++i,d->name);
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

    printf("Enter the interface number (1-%d):",i);
    scanf_s("%d",&inum);

    if(inum<1||inum>i)
    {
        printf("\nInterface number out of range,\n");
        //Free the device list
        pcap_freealldevs(alldevs);
        return -1;
    }

    //Jump to the selected adapter
    for(d=alldevs,i=0;i<inum-1;d=d->next,i++);

    //Open the device
    if( (adhandle = pcap_open(d->name,      //name of the device
                            65536,          //portion of the packet to capture
                                            //65536 guarantees that the whole packet will be captured on all the link layers
                            PCAP_OPENFLAG_PROMISCUOUS,  //promiscuous mode
                            -1,         //read timeout
                            NULL,           //authentication on the remote machine
                            errbuf          //error buffer
                            ) ) == NULL)
    {
        fprintf(stderr,"\nUnable to open the adapter.%s is not supported by WinPcap\n",d->name);
        //Free the device list
        pcap_freealldevs(alldevs);
        return -1;
    }

    printf("\nlistening on %s..\n",d->description);

    //At this point,we don't need any more the device list.Free it
    pcap_freealldevs(alldevs);

    //start the capture
    pcap_loop(adhandle,0,packet_handler,NULL);

    return 0;
}


//Callback function invoked by libpcap for every incoming packet
void packet_handler(u_char *param,const struct pcap_pkthdr *header,const u_char *pkt_data)
{
    struct tm ltime;
    char timestr[16];
    time_t local_tv_sec;

    //unused variables
    (VOID)(param);
    (VOID)(pkt_data);

    //convert the timestamp to reaable format
    local_tv_sec = header->ts.tv_sec;
    localtime_s(&ltime,&local_tv_sec);
    strftime(timestr,sizeof timestr,"%H:%M:%S",&ltime);

    printf("%s,%.6d len:%d\n",timestr,header->ts.tv_usec,header->len);
}
```

首先定义了处理数据包的packet_handler()函数原型，接着打印适配器列表，让用户选择使用哪个适配器。其中有一个**pcap_t**结构的指针adhandle，官方文档是这样介绍pcap_t结构的：

> 一个已打开的捕捉实例的描述符。这个结构体对用户来说是不透明的，它通过wpcap.dll提供的函数，维护了它的内容。

我们只需要了解这个结构是给pcap_open()和pcap_compile()等函数调用的，相当于一个打开的WinPcap Session就可以了。

接下来使用pcap_open()函数打开适配器，pcap_open()函数原型定义如下：

``` C++
pcap_t* pcap_open   (   const char *    source,
                        int     snaplen,
                        int     flags,
                        int     read_timeout,
                        struct pcap_rmtauth *   auth,
                        char *  errbuf   
                     )
```

函数接受6个参数：
1. 第一个参数**source**指定网卡的**名称**，也就是pcap_if结构中name的值。
2. 第二个参数**snaplen**指定要捕获数据包中的哪些部分，我们可以设置只捕获数据包的初始化部分，以提高捕获效率。这里设置为65536已经超过了最大的**MTU(Maximum Transmission Unit)**，所以能捕获到完整的数据包。
3. 第三个参数**flags**用来将网卡设置为**混杂模式**，如果网卡工作在非混杂模式下，网卡只接受来自网络端口的目的地址指向自己的数据。当网卡工作在混杂模式下时，网卡将接受所有的数据并将所有的数据包交给WinPcap处理，比如集线器会将接受到的数据包发送给所有接口，处于混杂模式时我们就能接收到发送给其他网卡的数据包。
4. 第四个参数**read_timeout**指定读取数据的超时时间，以毫秒计(1s=1000ms)。在适配器上进行读取操作(比如用 pcap_dispatch() 或 pcap_next_ex()) 都会在read_timeout毫秒时间内响应，即使在网络上没有可用的数据包。 如果设置为一定的时间，则将会在一个操作中读取该时间段内的多个数据包。如果设置为0意味着没有超时，那么如果没有数据包到达的话，读操作将永远不会返回。 如果设置成-1，则情况恰好相反，无论有没有数据包到达，读操作都会立即返回。
5. 第五个参数**auth**用于远程主机上验证用户权限的，我们只在本地捕获，因此该参数设置为NULL。
6.第六个参数**errbuf**用于存储错误信息。
函数执行成功后返回一个已打开的捕捉实例的描述符给adhandle，用于后面的pcap_loop()函数。

接下来释放设备列表，调用pcap_loop()函数开始捕获，pcap_loop()函数原型如下：

``` c++
int pcap_loop   (   pcap_t *    p,
                    int     cnt,
                    pcap_handler    callback,
                    u_char *    user     
                )
```
1. 第一个参数为上面pcap_open()函数返回的捕获实例
2. 第二个参数**cnt**指定当读取了cnt个数据包的时候pcap_loop()函数才会返回，当cnt为负数时pcap_loop()函数会一直循环(除非有错误发生)。
3. 第三个参数**callback**是回调参数，指定一个可以接受数据包的函数，这个函数会在收到每个新的数据包并收到一个通用状态时被libpcap所调用。这个函数是处理数据包的主要函数，在下面会详细介绍，回调函数原型如下：

``` C++
typedef void(*) pcap_handler(u_char *user, const struct pcap_pkthdr *pkt_header, const u_char *pkt_data)
```

4.第四个参数**user**是一个用户自定义的参数，包含了捕获session的状态，这个参数对应了pcap_handler()回调函数的第一个参数user。

以上基本上就是一个查找设备列表——选择设备——打开设备——开始捕获的过程，但我们在捕获到数据包后如果不对其进行处理就得不到有意义的信息，因此传递给pcap_loop()的回调函数packet_handler()就是用来对数据包进行处理的，函数定义如下：

``` C++
void packet_handler(u_char *param,const struct pcap_pkthdr *header,const u_char *pkt_data)
{
    struct tm *ltime;
    char timestr[16];
    time_t local_tv_sec;

    //unused variables
    (VOID)(param);
    (VOID)(pkt_data);

    //convert the timestamp to readable format
    local_tv_sec = header->ts.tv_sec;
    localtime_s(&ltime,&local_tv_sec);
    strftime(timestr,sizeof timestr,"%H:%M:%S",&ltime);

    printf("%s,%.6d len:%d\n",timestr,header->ts.tv_usec,header->len);
}
```

每次捕获到数据包时，libpcap都会自动调用这个回调函数，并会自动填充参数。第一个参数是pcap_loop()或pcap_dispatch()的**user**参数。第二个参数是将捕获驱动与数据包关联起来的头部信息，包括时间戳与长度信息。第三个参数是数据包的真实数据，还包括协议头部。

在这里我们只用到了**header**参数，**param**和**pkt_data**参数没有用到。**header**是一个**pcap_pkthdr**结构的指针，结构体定义如下：

``` C++
struct pcap_pkthdr {
      struct timeval ts;  
      bpf_u_int32 caplen; 
      bpf_u_int32 len;    
};
```

其中ts表示时间间隔，**caplen**表示捕获到的数据包长度，如果在调用pcap_open()函数时设置**snaplen**参数比较小，就只能捕获到实际数据包的一部分，caplen就表示捕获到的长度，**len**表示数据包实际的长度，因此caplen总是小于等于len的。时间间隔的类型timeval定义如下：

``` C++
typedef struct timeval {
  long tv_sec;
  long tv_usec;
} timeval;
```

其中tv_sec表示秒，tv_usec表示微秒。

packet_handler()函数的前面几行就都是用来解析header里的时间的，其中有[tm结构](http://www.cplusplus.com/reference/ctime/tm/)，[time_t类型](http://www.cplusplus.com/reference/ctime/time_t/)，[localtime_s()函数](https://msdn.microsoft.com/en-us/library/a442x3ye.aspx)，[strftime()函数](http://www.cplusplus.com/reference/ctime/strftime/)。

然后就将时间以[时：分：秒]的格式存入timestr中，打印出这个时间，还有微妙，最后是数据包的长度。

运行一下程序，选择适配器：![OpeningAnAdapterAndCapturingThePackets0](https://raw.githubusercontent.com/Heatwave/Blog/master/images/OpeningAnAdapterAndCapturingThePackets0.JPG)

得到了数据包的时间戳和长度信息：![OpeningAnAdapterAndCapturingThePackets1](https://raw.githubusercontent.com/Heatwave/Blog/master/images/OpeningAnAdapterAndCapturingThePackets1.JPG)

其实打开适配器后，除了使用[pcap_loop()函数](http://www.ferrisxu.com/WinPcap/html/group__wpcapfunc.html#g6bcb7c5c59d76ec16b8a699da136b5de)捕获数据外，还可以使用[pcap_dispatch()函数](http://www.ferrisxu.com/WinPcap/html/group__wpcapfunc.html#g60ce104cdf28420d3361cd36d15be44c)，两个函数非常的相似，区别就是 pcap_dispatch()当超时时间到了(timeout expires)就返回 (尽管不能保证)，而 pcap_loop()不会因此而返回，只有当cnt数据包被捕获，所以，pcap_loop()会在一小段时间内，阻塞网络的利用。当然，在这个例子中，使用pcap_loop()函数已经足够了，pcap_dispatch()函数一般用于比较复杂的程序中。

还有就是使用pcap_loop()函数可能会遇到障碍，主要因为它直接由数据包捕获驱动所调用。因此，用户程序是不能直接控制它的。还有一个方法是使用[pcap_next_ex()函数](http://www.ferrisxu.com/WinPcap/html/group__wpcapfunc.html#g439439c2eae61161dc1efb1e03a81133)，同时还能提高可读性。

这些会在下一节详细说明，下一篇：[不用回调方法捕获数据包](sniff-without-callback.md)
