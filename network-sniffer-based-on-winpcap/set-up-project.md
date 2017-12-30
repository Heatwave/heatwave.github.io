> 本文主要参考了以下资料:[RaiChen的博客](http://www.cnblogs.com/raichen/p/4128819.html),[WinPcap官方文档](http://www.ferrisxu.com/WinPcap/html/index.html),[《网络分析技术揭秘》-吕雪峰](http://book.douban.com/subject/10830686/),[Phinecos(洞庭散人)的博客](http://www.cnblogs.com/phinecos/archive/2008/10/20/1315176.html)

下载与安装好WinPcap后，我们先创建VS2010项目与配置项目属性。
#1.创建项目

打开Microsoft Visual Studio 2010，网上一些教程用的是VS2008，配置应该也差不多，但用VS2005或者是VC++6.0什么的估计就不太一样了，建议还是用VS2010，我的VS版本是2010旗舰版10.0.40219.1 SP1。

打开VS后新建项目，选择**Win32控制台应用程序**，如图：![Sniffer_NewConsleSniffer](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_NewConsleSniffer0.jpg)

确定后在弹出的对话框勾选**空项目**，点完成：![Sniffer_NewConsleSniffer](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_NewConsleSniffer1.jpg)

完成之后打开**解决方案资源管理器**，如果右边没有的话在菜单栏的**视图**->**解决方案资源管理**中(或按Ctrl+Alt+L)打开，然后点开刚才创建的项目，右键**源文件**文件夹，选择添加->新建项：![Sniffer_NewItem](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_NewItem0.jpg)

选择**C++**文件，随便取个名：![Sniffer_NewItem](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_NewItem1.jpg)
#2.配置项目属性

新建好之后就开始针对我们的WinPcap项目进行设置，首先右键选择我们创建的项目，选择**属性**：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting0.jpg)

打开属性页之后注意上方我们配置的是**(Debug)**版本的属性，如果是**(Release)**版本的需要重新选择配置属性：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting1.jpg)

确认是Debug版本后，点开**配置属性(Configuration Properties)**，选择**C/C++**，在右边点击后面的**附加包含目录(Additional Include Directories)**后的按钮，在弹出的窗口中创建新行，选择之前下载好的WinPcap开发包中的**Include**文件夹，将WinPcap头文件添加到我们的项目中(我这里是直接将Include文件夹复制到我创建的项目文件夹下)：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting2.jpg)

接着，在**配置属性(Configuration Properties)**->**链接器(Linker)**中，选择右边的附加库目录，同样将WinPcap开发包中的Lib文件夹添加到项目中：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting3.jpg)

接下来在**C/C++**->**预处理器(Preprocessor)**中的**预处理器定义(Preprocessor Definitions)**中添加**HAVE_REMOTE**，注意加分号分隔。这个预处理在pcap.h中会用到(或者也可以在代码的开头写上#define HAVE_REMOTE，注意要写在#include "pcap.h"前，不过我也没搞懂这两种方式哪一种比较好)：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting4.jpg)

 最后，在**链接器(Linker)**->**输入(Input)**里的**附加依赖项(Additional Dependencies)**中添加**wpcap.lib;ws2_32.lib**，其中wpcap.lib为WinPcap的库文件，ws2_32.lib库文件包含Windows的一些socket函数，项目中可能会用到：![Sniffer_Setting](https://raw.githubusercontent.com/Heatwave/Blog/master/images/Sniffer_Setting5.jpg)

终于！配置终于完了！可以开始写代码了！下一篇：[获取适配器基本信息](get-the-adapter-info.md)
