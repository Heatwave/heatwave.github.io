> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

我们每次捕获的都是所有流经我们网络适配器的数据包，如果不进行过滤，数以万计的数据包我们是不可能一个个看完的。因此WinPcap提供了数据包过滤引擎，使用**pcap_compile()**和**pcap_setfilter()**函数来过滤数据包。

**pcap_compile()**是一个将高层的布尔过滤表达式编译成能够被过滤引擎所解释的底层字节码，高层的布尔过滤表达式可能像这样："**ip and tcp**"，这个表达式说明我们只想保留IPv4和TCP的数据包，或者还有这样："**src 192.168.1.1 and port 20**"，这样的表达式说明我们只想保留源地址为192.168.1.1并且端口号为20的数据包。更多布尔过滤表达式的资料请参阅：[过滤串表达式的语法](http://www.ferrisxu.com/WinPcap/html/group__language.html)

**pcap_setfilter()**将一个过滤器与内核捕获会话相关联，当**pcap_setfilter()**被调用时， 这个过滤器将被应用到来自网络的所有数据包，并且，所有符合要求的数据包将会立即复制给应用程序。

以下代码展示了如何编译并设置过滤器。在编译过滤器之前我们必须从**pcap_if**结构体中获得掩码，因为一些使用**pcap_compile()**创建的过滤器需要它。

``` C++
    if (d->addresses != NULL)
        /* Retrieve the mask of the first address of the interface */
        netmask=((struct sockaddr_in *)(d->addresses->netmask))->sin_addr.S_un.S_addr;
    else
        /* If the interface is without an address we suppose to be in a C class network */
        netmask=0xffffff; 


compile the filter
    if (pcap_compile(adhandle, &fcode, "ip and tcp", 1, netmask) < 0)
    {
        fprintf(stderr,"\nUnable to compile the packet filter. Check the syntax.\n");
        /* Free the device list */
        pcap_freealldevs(alldevs);
        return -1;
    }

set the filter
    if (pcap_setfilter(adhandle, &fcode) < 0)
    {
        fprintf(stderr,"\nError setting the filter.\n");
        /* Free the device list */
        pcap_freealldevs(alldevs);
        return -1;
    }
```

我们先通过**pcap_if**结构获取适配器的掩码，然后调用**pcap_compile()**函数，函数原型如下：

``` C++
int pcap_compile    (   pcap_t *    p,
                        struct bpf_program *    fp,
                        char *  str,
                        int     optimize,
                        bpf_u_int32     netmask  
)
```
1. 第一个参数是捕获会话，通过调用pcap_open()得到的。
2. 第二个参数是**bpf_program**结构的指针，用来存放编译过后的底层字节码。
3. 第三个参数是用户指定的高层过滤表达式，规则在上面介绍了。
4. 第四个参数控制是否对所得到的字节码进行优化。
5. 第五个参数是掩码，上面的代码就获取到了，在之前获取适配器高级信息这一节里也有提到。
   如果pcap_compile()函数返回小于0的值的话，就是有错误发生，很可能是因为过滤表达式没有写对。

Compile好了过滤表达式之后，就要把编译好的底层字节码应用到适配器上，调用**pcap_setfilter()**函数，函数原型如下：

``` C++
int pcap_setfilter( pcap_t *    p,
                    struct bpf_program *    fp   
)
```

1.第一个参数和上面一个函数的一样，都是打开网络适配器后得到的捕获Session，目的就是要将编译好的底层字节码关联到这个捕获Session上。
2.第二个参数是刚才通过**pcap_compile()**函数编译过滤表达式得到的底层字节码。

如此便设置好了一个捕获会话的过滤条件，接下来调用捕获函数开始捕获，就可以得到过滤后的数据包了。

捕获到数据包后，我们只能得到一堆二进制数据，如果不进行分析的话是不知道数据包具体的细节的，所以接下来才是重中之重，下一篇：分析数据包
