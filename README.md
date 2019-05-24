[English](https://github.com/holdengong/EasyOffice/wiki/README-English)

[更新日志](https://github.com/holdengong/EasyOffice/wiki/EasyOffice%E6%9B%B4%E6%96%B0%E6%97%A5%E5%BF%97)

# 简介
Excel和Word操作在开发过程中经常需要使用，这类工作不涉及到核心业务，但又往往不可缺少。以往的开发方式在业务代码中直接引入NPOI、Aspose或者其他第三方库，工作繁琐，耗时多，扩展性差——比如基础库由NPOI修改为EPPlus，意味着业务代码需要全部修改。由于工作需要，我在之前版本的基础上，封装了OfficeService，目的是最大化节省导入导出这种非核心功能开发时间，专注于业务实现，并且业务端与底层基础组件完全解耦，即业务端完全不需要知道底层使用的是什么基础库，使得重构代价大大降低。

EasyOffice提供了
- Excel导入：通过对模板类标记特性自动校验数据，并将有效数据转换为指定类型，业务端只在拿到正确和错误数据后决定如何处理；
- Excel导出：通过对模板类标记特性自动渲染样式；
- Word根据模板生成：支持使用文本和图片替换，占位符只需定义模板类，制作Word模板，一行代码导出docx文档；
- Word根据Table母版生成：只需定义模板类，制作表格模板，传入数据，服务会根据数据条数自动复制表格母版，并填充数据；
- Word从空白创建等功能：特别复杂的Word导出任务，支持从空白创建；

EasyOffice底层库目前使用NPOI,因此是完全免费的。
通过IExcelImportProvider等Provider接口实现了底层库与实现的解耦，后期如果需要切换比如Excel导入的基础库为EPPlus，只需要提供IExcelImportProvider接口的EPPlus实现，并且修改依赖注入代码即可。
。

# Nuget

```
//核心包
Install-Package EasyOffice

//如果需要使用转PDF功能，不需要转PDF的不要加这个包，比较大
Install-Package EasyOffice.Extensions
```
==IMPORTANT==

如果通过nuget使用EasyOffice.Extensions,需要手动将libwkhtmltox.dll，libwkhtmltox.so
这两个文件手动拷贝到自己的项目根目录下，即与csproj项目工程文件同级，不然会报错。


# 依赖注入
支持.net core自带依赖注入

```c
// 注入Office基础服务
services.AddEasyOffice(new OfficeOptions());
```

---

# IExcelImportService - Excel通用导入
## 定义Excel模板类

```c
 public class Car
    {
        [ColName("车牌号")]  //对应Excel列名
        [Required] //校验必填
        [Regex(RegexConstant.CAR_CODE_REGEX)] //正则表达式校验,RegexConstant预置了一些常用的正则表达式，也可以自定义
        [Duplication] //校验模板类该列数据是否重复
        public string CarCode { get; set; }

        [ColName("手机号")]
        [Regex(RegexConstant.MOBILE_CHINA_REGEX)]
        public string Mobile { get; set; }

        [ColName("身份证号")]
        [Regex(RegexConstant.IDENTITY_NUMBER_REGEX)]
        public string IdentityNumber { get; set; }

        [ColName("姓名")]
        [MaxLength(10)] //最大长度校验
        public string Name { get; set; }

        [ColName("性别")] 
        [Regex(RegexConstant.GENDER_REGEX)]
        public GenderEnum Gender { get; set; }

        [ColName("注册日期")]
        [DateTime] //日期校验
        public DateTime RegisterDate { get; set; }

        [ColName("年龄")]
        [Range(0, 150)] //数值范围校验
        public int Age { get; set; }
    }
```

## 校验数据

```c

    var _rows = _excelImportService.ValidateAsync<ExcelCarTemplateDTO>(new ImportOption()
    {
        FileUrl = fileUrl, //Excel文件绝对地址
        DataRowStartIndex = 1, //数据起始行索引，默认1第二行
        HeaderRowIndex = 0,  //表头起始行索引，默认0第一行
        MappingDictionary = null, //映射字典，可以将模板类与Excel列重新映射， 默认null
        SheetIndex = 0, //页面索引，默认0第一个页签
        ValidateMode = ValidateModeEnum.Continue //校验模式，默认StopOnFirstFailure校验错误后此行停止继续校验，Continue：校验错误后继续校验
    }).Result;
    
    //得到错误行
    var errorDatas = _rows.Where(x => !x.IsValid);
    //错误行业务处理
    
    //将有效数据行转换为指定类型
    var validDatas = _rows.Where(x=>x.IsValid).FastConvert<ExcelCarTemplateDTO>();
    //正确数据业务处理
```

## 转换为DataTable

```
      var dt = _excelImportService.ToTableAsync<ExcelCarTemplateDTO> //模板类型
                (
                fileUrl,  //文件绝对地址
                0,  //页签索引，默认0
                0,  //表头行索引，默认0
                1, //数据行索引，默认1
                -1); //读取多少条数据，默认-1全部
```


---

# IExcelExportService - 通用Excel导出服务
## 定义导出模板类

```
    [Header(Color = ColorEnum.BRIGHT_GREEN, FontSize = 22, IsBold = true)] //表头样式
    [WrapText] //自动换行
    public class ExcelCarTemplateDTO
    {
        [ColName("车牌号")]
        [MergeCols] //相同数据自动合并单元格
        public string CarCode { get; set; }

        [ColName("手机号")]
        public string Mobile { get; set; }

        [ColName("身份证号")]
        public string IdentityNumber { get; set; }

        [ColName("姓名")]
        public string Name { get; set; }

        [ColName("性别")]
        public GenderEnum Gender { get; set; }

        [ColName("注册日期")]
        public DateTime RegisterDate { get; set; }

        [ColName("年龄")]
        public int Age { get; set; }
```

## 导出Excel

```
    var bytes = await _excelExportService.ExportAsync(new ExportOption<ExcelCarTemplateDTO>()
    {
        Data = list,
        DataRowStartIndex = 1, //数据行起始索引，默认1
        ExportType = XLSX, 默认导出Excel类型，默认xlsx
        HeaderRowIndex = 0, //表头行索引，默认0
        SheetName = "sheet1" //页签名称，默认sheet1
    });

    File.WriteAllBytes(@"c:\test.xls", bytes);
```

### 性能测试
通过ExcelOption中的ExportType参数，可以调整导出类型。
默认正常XLSX导出
XLS:可导出XLS
FastXLSX: 快速导出XLSX，无样式
CSV: 导出CSV文件

**100万**条数据,每行10列,无任何样式
- XLSX:0:02:12.287
- FastXLSX: 0:01:05.309
- CSV: 0:00:08.692

**10万**条数据,每行10列,无任何样式
- XLSX: 0:00:13.228
- FastXLSX: 0:00:07.011
- CSV:  0:00:01.069

**1万**条数据,每行10列,无任何样式
- XLSX: 0:00:02.062
- FastXLSX: 0:00:00.923
- CSV:  0:00:00.302
---

# IWordExportService - Word通用导出服务
## CreateFromTemplateAsync - 根据模板生成Word
```
//step1 - 定义模板类
 public class WordCarTemplateDTO
    {
        //默认占位符为{PropertyName}
        public string OwnerName { get; set; }

        [Placeholder("{Car_Type Car Type}")] //重写占位符
        public string CarType { get; set; }

        //使用Picture或IEnumerable<Picture>类型可以将占位符替换为图片
        public IEnumerable<Picture> CarPictures { get; set; }

        public Picture CarLicense { get; set; }
    }

//step2 - 制作word模板

//step3 - 导出word
string templateUrl = @"c:\template.docx";
WordCarTemplateDTO car = new WordCarTemplateDTO()
{
    OwnerName = "刘德华",
    CarType = "豪华型宾利",
    CarPictures = new List<Picture>() {
         new Picture()
         {
              PictureUrl = pic1, //图片绝对地址，如果设置了PictureData此项不生效
              FileName = "图片1",//文件名称
              Height = 10,//图片高度单位厘米默认8
              Width = 3,//图片宽度单位厘米默认14
              PictureData = null,//图片流数据，优先取这里的数据，没有则取url
              PictureType = PictureTypeEnum.JPEG //图片类型，默认jpeg
         },
         new Picture(){
              PictureUrl = pic2
         }
    },
    CarLicense = new Picture { PictureUrl = pic3 }
};

var word = await _wordExportService.CreateFromTemplateAsync(templateUrl, car);

File.WriteAllBytes(@"c:\file.docx", word.WordBytes);
```

## CreateWordFromMasterTable-根据模板表格循环生成word

```
//step1 - 定义模板类，参考上面

//step2 - 定义word模板,制作一个表格,填好占位符。

//step3 - 调用,如下示例,最终生成的word有两个用户表格
  string templateurl = @"c:\template.docx";
    var user1 = new UserInfoDTO()
    {
        Name = "张三",
        Age = 15,
        Gender = "男",
        Remarks = "简介简介"
    };
    var user2 = new UserInfoDTO()
    {
        Name = "李四",
        Age = 20,
        Gender = "女",
        Remarks = "简介简介简介"
    };
    
    var datas = new List<UserInfoDTO>() { user1, user2 };
    
    for (int i = 0; i < 10; i++)
    {
        datas.Add(user1);
        datas.Add(user2);
    }
    
    var word = await _wordExportService.CreateFromMasterTableAsync(templateurl, datas);
    
    File.WriteAllBytes(@"c:\file.docx", word.WordBytes);
```

## CreateWordAsync - 从空白生成word

CreateWordAsync方法接收IWordElement（实现类：Table,Paragraph）
可以从空白创建表格和段落 

```
  [Fact]
        public async Task 导出所有日程()
        {
            //准备数据
            .................
            var tables = new List<Table>();

            //新建一个表格
            var table = new Table()
            {
                Rows = new List<TableRow>()
            };

            foreach (var date in dates)
            {
                //新建一行
                var rowDate = new TableRow()
                {
                    Cells = new List<TableCell>()
                };
                
                //新增单元格
                rowDate.Cells.Add(new TableCell()
                {
                    Color = "lightblue", //设置单元格颜色
                    Paragraphs = new List<Paragraph>()
                    { 
                    //新增段落
                    new Paragraph()
                    {
                       //段落里面新增文本域
                        Run = new Run()
                        {
                           Text = date.DateTimeStr,//文本域文本，Run还可以
                           Color = "red", //设置文本颜色
                           FontFamily = "微软雅黑",//设置文本字体
                           FontSize = 12,//设置文本字号
                           IsBold = true,//是否粗体
                           Pictures = new List<Picture>()//也可以插入图片
                        },
                         Alignment = Alignment.CENTER //段落居中
                    }
                    }
                });
                table.Rows.Add(rowDate);
                
            tables.Add(table);

            var word = await _wordExportService.CreateWordAsync(tables);

            File.WriteAllBytes(fileUrl, word.WordBytes);
        }
```

#  扩展功能 - Extensions
## IWordConverter Word转换器
支持docx文件转换为html，或者pdf。底层库使用OpenXml和DinkToPdf，开源免费。如果你们公司没有Aspose.Word的Lisence，这是个可以考虑的选择。


```
step1: Startup注入
serviceCollection.AddEasyOfficeExtensions();

step2:
构造函数注入IWordConverter _wordConverter

step3:调用
var pdfBytes = _wordConverter.ConvertToPDF(docxBytes, "text");
```












