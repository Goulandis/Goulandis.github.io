---
title: 【Dev】DevExpress应用
password: 
abstract: 
message: 
date: 2021-02-03 10:33:53
tags: Dev
categories: 知识记录
---

<meta name="referrer" content="no-referrer" />

## 一、大纲

最近使用DevExpress做C/S开发碰到了一些问题，在解决问题的同时在这里做一下记录，下面列出涉及到技术点

- **Dev框架下GridControl与GridView**
- **使用模板列动态替换GridView的指定列**
- **GridView分组并去掉列名的前缀**
- **FPT服务器文件预览与下载**
- **单元格添加按钮并添加自定义点击事件**
- **GridView数据导出到Excel**
- **向Word模板中写数据**

<!--more-->

## 二、Dev框架下的GridControl和GridView

### 1.GridControl和GridView的关系

Dev框架下GridControl负责操作数据，GridView负责展示数据，GridControl是GridView的容器，一个GridControl可以容纳多个GridView，在GridView中的任何数据操作都不会影响到GridControl中的源数据，即当我们将GridControl中数据重新刷入GridView时，GridView中数据操作会被清除，所以如果我们有要在本地展示的数据则需要在GridControl刷数据进入GridView的时候重新再刷入一次本地数据。

### 2.GridControl输入数据到GridView的原理

GridControl的数据刷入GridView的操作由Dev框架执行，我们需要做的只是把数据绑定到GridControl.DataSource中即可。

只有当GridView中存在与GridControl数据源对应列时，GridControl才能将对应列的数据刷入GridView的对应列中，这里的对应列指的是GridView中列的`FiledName`的值与GridControl数据源的列名相同，且大小写敏感。

在GridView的列属性中有三个极为重要的属性：

![](【Dev】DevExpress应用\Snipaste_2020-12-17_19-20-19.png)

- Name：列在程序中操作的标识符，类似变量名，对列的操作都由它来引用，如：修改colfilename列的列宽

  ```c#
  colfilename.Width = 300;
  ```

- ColumnEdit：用于挂载模板列的属性，可以将列动态的替换为其他类型的控件，例子中是将列挂载了一个多行编辑框，这样就可以在单元格中显示多行内容

  ![](【Dev】DevExpress应用\Snipaste_2020-12-17_19-29-45.png)

- FieldName：FieldName属性是列与GridControl数据源对应的标志，如果想要将GridControl数据源中某一列的数据刷入当前列，那么当前列的FieldName的取值必须和数据源中对应列的列名一致，并且FielName也是用来获取表格数据的标识，如：

  ```C#
  gridView_FileViewer.GetFocusedDataRow()["path"]//取所选行的path列单元格的数据
  ```

  

### 3.向GridView存在而GridControl中不存在的列刷入数据

有的时候为了展示需要，我们需要在GridView中增加新列刷入自己的数据，而新增列在GridControl的数据源中没有与之对应的列，即在数据源中没有对应的字段（这里的数据源通常情况下指的就是数据库中的表），此时我们就需要在GridControl.Datasource中动态地添加一列来与新增列对应。

为什么要要在GridControl.Datasource中动态地添加一列呢？

这可能是由GridControl和GridView的内部机制影响的，当一列在GridView中存在而GridControl中不存在时，我们是无法向此列写入数据的，即使数据是来自本地而不是数据库，并且编译器会报错：

![](【Dev】DevExpress应用\Snipaste_2020-12-17_19-49-11.png)

如果我们要向GridView存在而GridControl中不存在的列刷入数据，那么我们必须在GridControl的DataSource中动态的加列，下面是示例代码：

```c#
private void LoadFileNameColumn(object sender, EventArgs args)
{
	GridColumn col = gridView_JobPlacement.Columns["xgwj"];//取表格xgwj列的索引

	/*代码块说明：
	*   作用：向gridview的datasource动态添加filename列，使GridView中的filename列与DataSource中的filename字段对应
	*   说明：因为在GridView中添加了filename列如果在GridView的DataSource中没有与之对应的字段，
	*       则无法对filename列做任何操作
	*/
	DataTable gridViewTable = gridControl_JobPlacement.DataSource as DataTable;//取DataSource的引用并转换成DataTable
	if (!gridViewTable.Columns.Contains("filename"))//判断DataSource中是否已存在filename列
	{
		DataColumn dsFileNameCol = new DataColumn();//创建新列
		dsFileNameCol.ColumnName = "filename";//将新列命名为filename
         gridViewTable.Columns.Add(dsFileNameCol);//将新列添加到DataSource中
	}//至此，DataSource中就存在与GridView中的filename列对应的filename列了

	//遍历GridView所有行，对有文件组编码的行在filename列载入文件列表信息
	 for (int rowIndex = 0; rowIndex < gridView_JobPlacement.RowCount; rowIndex++)
	{     
		 DataRow row = gridView_JobPlacement.GetDataRow(rowIndex);//根据索引数据行
		if (row["xgwj"].ToString() != "")//如果数据行中的xgwj列单元格不为空，则向单元格刷入指定数据
		{
			string fileGroupTmp = gridView_JobPlacement.GetRowCellDisplayText(rowIndex, col);//取指定单元格显示的内容
			string fileGroup = CodingTool.GetFileGroup(fileGroupTmp);//将单元格存储的文件组编码转换成数据库可用的编码
			DataTable table = bll.SelectFromFileTableByFileGroup(fileGroup);//根据编码到数据库查询文件列表
			foreach(DataRow tableRow in table.Rows)//遍历文件列表将文件名刷入新增列
			{
				row["filename"] += tableRow["filename"]+"\n";//将数据刷入filename列单元格
			}
         }
     }
}
```

## 三、使用模板列动态替换GridView中的指定列

有的时候为了保密需要，在数据库中部分字段会用编码标识，如：人名使用编码标识，张三对应编码001，但是在表格中展示的时候应该显示人名而不是编码，此时我们就需要用到模板列的动态替换。

直接上代码：

```c#
Security.BLL.userinfo ubll = new Security.BLL.userinfo();

RepositoryItemGridLookUpEdit replaceRegistrant = new RepositoryItemGridLookUpEdit();
replaceRegistrant.DataSource = ubll.GetAllList().Tables[0];//绑定数据源到RepositoryItemGridLookUpEdit
replaceRegistrant.DisplayMember = "fullname";  //选择要替换显示的字段
replaceRegistrant.ValueMember = "ID";  //
replaceRegistrant.NullText = "";//字段为空时要显示的内容
gridView_JobPlacement.Columns["djr"].ColumnEdit = replaceRegistrant;//将RepositoryItemGridLookUpEdit绑定到GridView的“djr”列

RepositoryItemGridLookUpEdit replacePricipal = new RepositoryItemGridLookUpEdit();
replacePricipal.DataSource = ubll.GetAllList().Tables[0];//绑定数据源到RepositoryItemGridLookUpEdit
replacePricipal.DisplayMember = "fullname";  //选择要替换显示的字段
replacePricipal.ValueMember = "ID";  //
replacePricipal.NullText = "";//字段为空时要显示的内容
gridView_JobPlacement.Columns["fzr"].ColumnEdit = replacePricipal;//将RepositoryItemGridLookUpEdit绑定到GridView的“djr”列
```

这是通过代码添加动态的添加模板列，同时我们也可以在列属性中的ColumnEdit属性中静态的添加模板列。

## 四、GridView分组并去掉列名的前缀

### 1.分组

GridView分组只需要在需要分组的列的属性中将GroupIndex属性值由“-1”改为0即可，如果需要二级分组则在需要分组的列的属性中将GroupIndex属性值由“-1”改为1，以此类推需要三级分组则改为2。

![](【Dev】DevExpress应用\Snipaste_2021-01-07_20-38-53.png)

### 2.去掉列名前缀

分完组后如果不做修改我们加载数据之后表格是这样的：

![](【Dev】DevExpress应用\Snipaste_2021-01-07_20-43-04.png)

有时我们不需要显示列名前缀，这时我们需要修改GridView的GroupFormat属性修改为{1}，GroupFormat属性的默认值是`{0}: [#image]{1} {2}`，其中

- {0}显示列标题
- [#image]显示图片
- {1}显示列的内容值
- {2}显示列的摘要

![](【Dev】DevExpress应用\Snipaste_2021-01-07_20-45-55.png)

设置好之后，效果是这样的:

![](【Dev】DevExpress应用\Snipaste_2021-01-07_20-45-28.png)

## 四、FTP文件预览与下载

直接上代码，解释都放注释上了：

```c#
private void PreviewFile(object sender,EventArgs e)
{
    if (fileDic == null)
    {
        return;
    }
    //自定义函数，获取文件在服务器中的路径
    string serverPath = ServerFTP.CreateFilePathInServerBySQLPath(gridView_FileViewer.GetFocusedDataRow()["path"].ToString());
    //获取文件名
    string fileName = gridView_FileViewer.GetFocusedDataRow()["file"].ToString();
    //根据路径将文件下载到本地，并返回文件路径
    string savePath = ServerFTP.currentMode.RequestFile(serverPath,fileName,GeneralLib.FTPDownloadStyle.CACHE);
    if (savePath != "")
    {
        //调用系统软件打开文件
        FileIO.OpenFileInWindows(savePath);
    }  
}
```

这里挑几个比较重要的函数讲解

**RequestFile**

```c#
public string RequestFile(string serverPath,string fileName, FTPDownloadStyle style)
{
    string savePath = "";
    switch (style)
    {
        //下载文件到缓存临时文件夹，用于预览
        case FTPDownloadStyle.CACHE:
            savePath = Path.Combine(ServerFTP.cachePath, fileName);
            break;
        //下载文件到所选的文件夹，用于下载
        case FTPDownloadStyle.CHOOSEDIC:
            savePath = FileIO.ChooseSaveFile(fileName, "", ServerFTP.chooseTempDic);       
            //保存所选的文件夹，以便下次打开直接进入相应目录
            ServerFTP.chooseTempDic = Path.GetDirectoryName(savePath);
            break;
    }
    if (savePath == null)
    {
        return "";
    }

    //判断目录是否存在
    if (!Directory.Exists(Path.GetDirectoryName(savePath)))
    {
        //如果不存在则创建目录
        Directory.CreateDirectory(Path.GetDirectoryName(savePath));
    }

	//根据文件路径创建FPT连接实例
    FtpWebRequest ftp = (FtpWebRequest)WebRequest.Create(serverPath);
    //从配置文件中读取登录项
    ICredentials credentials = new NetworkCredential(Config.Get["ftp_username"], Config.Get["ftp_password"]);
    //配置FPT服务器登录项
    ftp.Credentials = credentials；
    //配置FPT操作为下载文件
    ftp.Method = WebRequestMethods.Ftp.DownloadFile;
	//向FPT服务器发出操作请求
    FtpWebResponse response = (FtpWebResponse)ftp.GetResponse();
    //创建流缓冲区接收FPT服务器反馈的字节流
    Stream responseStream = response.GetResponseStream();
    //根据存储路径在本地创建文件
    FileStream fs = File.Create(savePath);
    //创建用于批量取缓冲区字节数据的数据
    byte[] buffer = new byte[ConstLib.BUFFER_SIZE];
    int read = 0;
    do
    {
        //将缓冲区的字节数据读入字节数组
        read = responseStream.Read(buffer, 0, buffer.Length);
        //将字节数组的数据写入到文件中
        fs.Write(buffer, 0, read);
        //清楚fs的流缓冲区，这里fs的流缓冲区与responseStream流缓冲区不是同一个缓冲区，需要注意
        fs.Flush();
    }
    while (read != 0)；
        
    fs.Flush();
    //关闭文件
    fs.Close();
    
    return savePath;
}
```

**OpenFileInWindows**

```C#
public static Process OpenFileInWindows(string filePath)
{
    //创建一个新的进程
    ProcessStartInfo info = new ProcessStartInfo();
    //设置进程要打开的文档，Windows会根据文件类型的默认开打应用来启动对应应用程序来打开文件
    info.FileName = Path.GetFileName(filePath);
    //设置启动进程的初始目录
    info.WorkingDirectory = Path.GetDirectoryName(filePath);
    //设置进程启动后，窗口的状态，可以设置为最大化，最小化和正常
    info.WindowStyle = ProcessWindowStyle.Normal;
    //启动进程
    Process proc = Process.Start(info);

    return proc;
}
```

当关闭文件时清楚临时文件夹的内容

```c#
public static void CleanCacheDirectory()
{
    //判断临时文件夹是否存在
    if (!Directory.Exists(ServerFTP.cachePath))
    {
        return;
    }
    try
    {
        //等待系统将占用文件的进程杀死再清空临时文件夹
        System.Threading.Thread.Sleep(500);
        //获取临时文件夹目录信息
        DirectoryInfo dicInfo = new DirectoryInfo(ServerFTP.cachePath);
        //直接删除临时文件夹
        dicInfo.Delete(true);
    }
    catch
    {
        //如果目录被其他进程占用，则暂时不清空临时文件夹
        return;
    }
}
```

这里我使用的是最简单的直接删除临时文件夹的暴力删除法，这样做会有一个问题就是，在程序删除文件夹的时候，可能预览文件的进程还没有被系统杀死或有其他的进程占用了目录中文件，这都会导致目录删除失败而抛出异常，我的解决方案是在删除目录之前等待500ms，等待系统将预览文件的进程杀死后在删除文件夹，但是如果是其他的进程占用了目录，则需要手动结束进程才能继续删除临时文件夹，我的解决方案是，如果有其他进程占用了目录，则本次本次临时缓冲区先不删除，等下次有机会再删除。所以这里的try-catch不是用来抛出异常的，而是用来推出函数的。

当然比较理想的删除方法是遍历整个目录中文件和子文件夹，依次删除目录下文件和子文件夹，有被其他进程占用的文件暂时不删除。这样就可以只留下被占用的文件，而不是整个目录。

## 五、单元格添加按钮并添加自定义点击事件

有时我们需要向某一列的单元格添加点击事件，甚至向某一个单元格添加点击事件，这时我们就需要向单元格添加按钮了。

### 1.向单元格添加简单的点击事件

如果我们只想在某一单元格添加简单的点击事件

```C#
private void gridView_JobPlacement_RowCellClick(object sender, RowCellClickEventArgs e)
{
    if (e.RowHandle == 1 && e.Column.FieldName == "filename")
    {
        if (e.Button == MouseButtons.Left)
        {
            //todo
        }
        if (e.Button == MouseButtons.Right)
        {
			//todo
        }
        if(e.Button == MouseButtons.Moddle)
        {
            //todo
        }
    }
}
```

这时我们需要用到RowCellClick事件，RowCellClick事件在鼠标点击单元格时触发，然后我们只需要判定鼠标点击是哪一行哪一列，就可以实现某一个单元格的点击事件了。当然我们也可以通过添加按钮来实现。

### 2.向单元格添加复杂点击事件

如果我们想向单元格添加一系列复杂的点击事件，如在某一单元格内做文件的上传，预览，下载，删除等操作，这时我们就需要借助模板列了，使用模板列是无法只向某一个单元添加点击事件的，因为模板列挂载的是一整列。

我需要用到模板列`RepositoryItemButtonEdit`，我可以在列属性里静态挂载，也可以在代码中动态挂载，重要的是我们需要用到`RepositoryItemButtonEdit`属性里的`Buttons`属性，向Buttons属性里添加元素。

![](【Dev】DevExpress应用\Snipaste_2020-12-18_09-11-36.png)

光是添加按钮单元中还是看不到按钮的，我们还需要将每个按钮的Kind属性设置为Glyph，这样我们才能在单元格中看到按钮

![](【Dev】DevExpress应用\Snipaste_2020-12-18_09-13-12.png)

添加完按钮就可以向对应按钮添加点击事件了，我们可以发现在列属性里找不到事件，所以我们需要在代码中为按钮添加点击事件，这时我们需要用到`repositoryItemButtonEdit.Buttons[0].Click`，其中repositoryItemButtonEdit是模板列的名字，Buttons[0]是第一个按钮的引用，我们只需要向Click事件添加我们想要执行的函数即可。

## 六、GridView数据导出到Excel

 GridView的数据要导出到Excel有很多种方法，可以最直接的就是遍历GridView，然后将数据写入Excel，这算是比较麻烦的做法了，事实上Dev已经提供了一些便捷的方法。

### 1.GetAllFilteredAndSortedRows()方法

Dev提供了一个`GridView.DataController.GetAllFilteredAndSortedRows()`方法，可以用于提取GridView当前数据，在筛选排序等操作之后更改了的数据也可以提取。

<font color=red> 但是，GridView类中的DataController对象在VS中被隐藏了，即通过提示器是找不到GDataController对象的的，只能通过手写调用。</font>

![](【Dev】DevExpress应用\Snipaste_2020-12-22_19-13-20.png)

GetAllFilteredAndSortedRows()方法返回的是一个IList泛型列表，数据写入Excel一就要自己手动写入，写入方法：

```C#
//需要包含的引用
using DevExpress.XtraSpreadsheet;
using DevExpress.Spreadsheet;

SpreadsheetControl ss = new SpreadsheetControl();
var book = ss.Document;
Worksheet sheet = book.Worksheets[0];
sheet.Import(table, true, 0, 0);//table是DataTable类型，也是要导出到Excel的数据
```

<font color=red> 其中有一点需要格外注意，在使用Import函数时需要引用`DevExpress.Docs`程序集，因为Import函数在这个程序集里，Dev在DevExpress.Docs程序集里给Worksheet的父类ExternalWorksheet写了扩展，也就是扩展了Import函数等，其中DevExpress.Docs程序集和DevExpress.Spreadsheet程序集的命名空间是一样的，如果没有搞清楚这一点很容易产生玄学问题🥴</font>

### 2.GridView.Export()方法

最简单的方法就是使用Dev官方提供的导出方法GridView.Export().

Dev已经在GridView中添加了官方的Export方法，支持多种导出格式：

- Xls
- Xlsx
- Html
- Mht
- Pdf
- Text
- Rtf
- Csv
- Image
- Docx

同时提供三种重载：

![](【Dev】DevExpress应用\Image_20201222193848.png)

使用方法也很简单：

```C#
/// <summary>
/// 将GridView中的数据导出到Excel
/// </summary>
/// <param name="fileName"></param>
/// <param name="view"></param>
public static void ExportExcel(string fileName, GridView view)
{
    SaveFileDialog sfd = new SaveFileDialog();
    sfd.Title = "另存为";
    sfd.InitialDirectory = "C:\\";
    sfd.Filter = "Excel文件(*.xlsx) | *.xlsx";
    sfd.FileName = fileName;//fileName文件名不需要包含后缀
    if (sfd.ShowDialog() == DialogResult.OK)
    {
        view.Export(DevExpress.XtraPrinting.ExportTarget.Xlsx, sfd.FileName);
    }
}
```

- view.Export是dev自带的导出方法，在导出文件后dev会自动调用系统对此文件的默认打开应用来打开文件，当然dev也提供内置的预览方法，这在下一节导出word模板中使用。

## 七、向Word模板中写入数据

### 1.载入word模板文件

向word模板中写入数据我这里主要使用的是<font color=red>RichEditControl</font>类，RichEditControl类提供海量的富文本API接口，这里主要讲解使用到的API。

首先打开word文档，RichEditControl类提供RichEditControl.LoadDocument(string path)方法加载文档，RichEditControl类也提供多个LoadDocument函数的重载给予各种文件的加载形式，我这里使用的是直接通过文件路径加载文档。LoadDocument函数支持DOC、DOCX、RTP、HTM、HTML、MHT、XML和EPUB类型的文档，可以自动检测文档类型。

```C#
RichEditControl.LoadDocument(string path)
```

将文档载入内存之后就可以通过<font color=red>RichEditControl.Document.Text</font>属性查看文档内容了，也可以通过RichEditControl.Document.Text属性判断文档是否加载成功。

```C#
if(richEditControl.Document.Text == null)
{
	return;
}
```

### 2.向word模板的指定位置写入数据

向word模板的指定位置写入数据主要使用Word的书签和域，我这里使用的是书签，在word中想要插入数据的地方添加一个书签即可，如：

![](【Dev】DevExpress应用\Snipaste_2021-01-06_21-26-31.png)

我想要在生产号、型号和图号后面的单元格写入数据，那么我只需要在这些单元中添加书签即可，添加书签的步骤：

```mermaid
graph LR;
将光标定位到要添加的书签的位置-->插入-->书签-->添加一个书签名-->添加
```

添加完书签时候在word上是看不到的，但是把光标定位到书签所在的位置处，插入书签时会自定定位到所插入的书签名。

然后即可通过<font color=red> Document.Replace(DocumentRange range,string text)</font>函数来向书签所在位置插入数据了，其中DocumentRange类型的参数需要通过<font color=red>Document.Bookmarks[string bookmarks].Range</font>来将字符串类型的书签标志转换为DocumentRange类型的可用书签标志。

如：我要在生产号、型号和图号后面的单元格写入数据，那么我需要分别在这些单元格中插入书签`sch`、`xh`、`th`，然后通过下面代码即可向word模板中写入数据

```C#
RichEditControl richEdit = new RichEditControl();
richEdit.LoadDocument("C:/a.doc");
Document doc = rich.Document;
doc.Replace(doc.Bookmarks["sch"].Range,"01");
doc.Replace(doc.Bookmarks["xh"].Range,"02");
doc.Replace(doc.Bookmarks["th"].Range,"03");
```

原理就是书签提供了一个占位符，而dev则通过搜索匹配的占位符，将指定数据替换掉占位符。

### 3.向word模板中的表格插入新行并写入内容

向word模板中的表格插入新行则稍微复杂一些。主要步骤如下：

- 首先word文档中要有一个模板表格

- 需要在要插入表的位置添加书签table

- 遍历word文档中所有的表再遍历每一个表中所有的单元格，查找到书签所在的单元格

  ```c#
  public TableCell GetTableCell(Document document) 
  {
      //遍历文档中所有的表
      foreach (Table table in document.Tables)
      {
          int row = 0, col = 0;
          bool ok = false;
          TableCell retCell = null;
  		//遍历表格中所有的单元格
          table.ForEachCell((cell, rowIndex, columnIndex) =>
                            {
                                if (cell.Range.Contains(document.Bookmarks["table"].Range.Start))
                                {
                                    row = rowIndex;
                                    col = columnIndex;
                                    retCell = cell;
                                    ok = true;
                                }
                            });
          if (ok)
          {
              return retCell;
          }
      }
  ```

  <font color=red>Table.ForEachCell(TableCellProcessorDelegate cellProcessor)</font>函数传入的是一个委托。这里使用的是匿名方法

- 在指定单元格后新增行

  可以使用<font color=red>Document.Tables[int index].Rows.Append()</font>函数在表的最后追加行，或使用<font color=red> Document.Tables[int index].Rows.InsertAfter(int rowIndex)</font>函数在指定行之后插入行。其中Document.Tables[int index].Rows.Append()中index（表的索引）可以通过<font color=red> Document.Tables.IndexOf(Table table)</font>函数获取，而table又可以同通过上一步查找到的TableCell对象retCell.Table属性获取。

  ```c#
  RichEditControl richEdit = new RichEditControl();
  richEdit.LoadDocument("C:/a.doc");
  Document doc = rich.Document;
  TableCell cell = GetTableCell(doc);
  doc.BeginUpdate();
  int index = doc.Tables.IndexOf(cell.Table);
  doc.Tables[index].Rows.Append();//或者
  //doc.Table[index].Rows.InsertAfter(cell.Row.Index - 1);
  //获取指定单元格的占位符范围
  DocumentRange range = doc.Tables[index].Rows[cell.Row.Index].Cells[cell.Index].ContentRange;
  doc.Replace()
  doc.EndUpdate();
  ```

  