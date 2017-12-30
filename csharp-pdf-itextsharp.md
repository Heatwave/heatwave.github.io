  最近学校有项目需要用C#创建PDF文件，输出程序中的表格。在网上搜索后发现C#确实没有原生的解决办法，大多都是用到了一个第三方的库:**iTextSharp**。这个库的核心组件可以在[这里下载](http://sourceforge.net/projects/itextsharp/)

  压缩文件itextsharp-all-core.zip中的iTestSharp.dll便是我们要用到的组件，我下载的时候组件更新到了版本5.5.0，其中删除了之前版本中的table类，但网上流传最广的iTextSharp使用手册中仍然使用的是Table来创建表格，这个方法只适用于之前的版本。因此这里使用另一个类**PdfPTable**来创建表格。

首先在visual studio 2010中引用iTextSharp.dll库，右键单击“解决方案资源管理器”中的“引用”，选择“添加引用”，用“浏览”找到你存放iTestSharp.dll的文件夹，点击确定便将iTextSharp添加到了你的解决方案中。

接下来在代码中添加命名空间如下：

``` csharp
using System.IO;
using iTextSharp.text;
using iTextSharp.text.pdf;
```

其中IO用来保存文件，后面两个是要用来创建和编辑PDF文档的类。

以下代码是在点击一个按钮之后，让用户选择保存的位置，创建一个PDF文档，将tablelayoutpanel控件中的内容做成一个表格保存到PDF文件中：

``` csharp
private void button3_Click(object sender, EventArgs e)          //“保存数据”按钮,保存为PDF文件
{
    Stream myStream;            //文件流
    SaveFileDialog savefile = new SaveFileDialog();         

        savefile.Filter = "pdf files (*.pdf)|*.pdf|All files (*.*)|*.*";    //保存文件的格式
        savefile.FilterIndex = 1;                   //默认保存文件格式索引，默认为第一种pdf格式
        savefile.RestoreDirectory = true;               //记忆上次打开目录

        if (savefile.ShowDialog() == DialogResult.OK)  //点保存之后
        {
        string localFilePath = savefile.FileName.ToString(); //获得保存文件路径 
                string fileNameExt = localFilePath.Substring(localFilePath.LastIndexOf("\\") + 1); //获取文件名，不带路径

                myStream = savefile.OpenFile();         //打开文件并赋给IO流myStream

                Document document = new Document(PageSize.A4.Rotate());         //创建A4纸、横向PDF文档
                PdfWriter writer = PdfWriter.GetInstance(document,myStream);    //将PDF文档写入创建的文件中
            document.Open();
        //要在PDF文档中写入中文必须指定中文字体，否则无法写入中文
                BaseFont bftitle = BaseFont.CreateFont("C:\\Windows\\Fonts\\SIMHEI.TTF", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);   //用系统中的字体文件SimHei.ttf创建文件字体
                iTextSharp.text.Font fonttitle = new iTextSharp.text.Font(bftitle, 30);     //标题字体，大小30
                BaseFont bf1 = BaseFont.CreateFont("C:\\Windows\\Fonts\\SIMSUN.TTC,1", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);     //用系统中的字体文件SimSun.ttc创建文件字体
                iTextSharp.text.Font CellFont = new iTextSharp.text.Font(bf1, 12);          //单元格中的字体，大小12
                iTextSharp.text.Font fonttitle2 = new iTextSharp.text.Font(bf1, 15);        //副标题字体，大小15

                //添加标题
                Paragraph Title = new Paragraph("示例文件", fonttitle);     //添加段落，第二个参数指定使用fonttitle格式的字体，写入中文必须指定字体否则无法显示中文
                Title.Alignment = iTextSharp.text.Rectangle.ALIGN_CENTER;       //设置居中
                document.Add(Title);        //将标题段加入PDF文档中

                //空一行
                Paragraph nullp = new Paragraph(" ", fonttitle2);
                nullp.Leading = 10;
                document.Add(nullp);

                PdfPTable table = new PdfPTable(int)numericUpDown2.Value);         //numericUpDown2为用户设置的列数，创建Value列的表格,行会根据写入数据自动扩展

        //不同单元格对应tablelayoutpanel添加不同文本
        for (int j = 0; j < tableLayoutPanel1.RowCount; j++)//j为行标 
        { 
            for (int i = 0; i < tableLayoutPanel1.ColumnCount; i++)//i为列表 
            { 
                if (i == 0 && j == 0) //左上角为空 
                { 
                        table.AddCell(" ");//向表格的单元格添加数据，此处为空白 
                        continue; 
                } 
                if (j == 0) //第一行标号 
                { 
                        table.AddCell(i + "#"); 
                        continue; 
                } 
                if (i == 0 && j > 0) //tablelayoutpanel第一列为textbox控件，读取用户输入的文本 
                { 
                        Control c = tableLayoutPanel1.GetControlFromPosition(i, j);//获取tablelayoutpannel容器中第i列、第j行的控件 
                        if (c is TextBox)//判定控件的类型如果为textbox则将文本内容写入PDF文档 
                        { 
                            table.AddCell(new Paragraph(c.Text, CellFont));//用CellFont字体将textbox中的内容写入PDF文档的单元格中 
                        } 
                        continue; 
                } 
                if (i > 0 && j > 0) //单元格数据 
                { 
                    Control c = tableLayoutPanel1.GetControlFromPosition(i, j); 
                    if (c is Label) 
                    { 
                            table.AddCell(c.Text); 
                    } 
                    else 
                    { 
                            table.AddCell(" "); 
                    } 
                    continue; 
                } 

                table.AddCell(" "); //如果tablelayoutpanel单元格中不存在控件则写入空单元格 
            } 
    } 

    document.Add(table); //将表格加入PDF文档中 
    document.Close(); myStream.Close();
    }
}
```

代码都有注释，应该都看得懂，有问题请提出~
