# 前端：SheetJs 赋能页面与 Excel 文件交互

## 1、认识 SheetJS <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-1&#x3001;&#x8BA4;&#x8BC6;SheetJS"></a>

### 1.1 SheetJS 可以做什么 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-1.1SheetJS&#x53EF;&#x4EE5;&#x505A;&#x4EC0;&#x4E48;"></a>

* 读文件：页面直接读取 Excel 文件中的数据
* 写文件：页面数据直接写入到 Excel 文件中
* 不止是 Excel ：SheetJS 支持读写多种表格格式文件

### 1.2 SheetJS 的数据结构 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-1.2SheetJS&#x7684;&#x6570;&#x636E;&#x7ED3;&#x6784;"></a>

![Workbook&#x8868;&#x793A;&#x6574;&#x4E2A; excel &#x6587;&#x4EF6;&#x6570;&#x636E;&#xFF0C;Sheet &#x5BF9;&#x8C61;&#x8868;&#x793A; excel &#x67D0;&#x4E00;&#x9875;&#x7684;&#x6570;&#x636E;](https://wiki.dycjr.net:1443/download/attachments/10620742/image2018-11-19_0-13-28.png?version=1&modificationDate=1542557609000&api=v2)



Workbook 对象是最大的数据对象，表示整个 excel 文件数据

Sheet 对象是 Sheet 对象，表示 excel 某一页的数据

## 2、SheetJS 读 Excel 文件数据 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-2&#x3001;SheetJS&#x8BFB;Excel&#x6587;&#x4EF6;&#x6570;&#x636E;"></a>

### 2.1 关键函数：readFile <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-2.1&#x5173;&#x952E;&#x51FD;&#x6570;&#xFF1A;readFile"></a>

基础示例

```javascript
if(typeof require !== 'undefined') XLSX = require('xlsx');
var workbook = XLSX.readFile('test.xlsx');
/* DO SOMETHING WITH workbook HERE */
```

## 3、SheetJS 写数据到 Excel 文件 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-3&#x3001;SheetJS&#x5199;&#x6570;&#x636E;&#x5230;Excel&#x6587;&#x4EF6;"></a>

### 3.1 关键函数：writeFile <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-3.1&#x5173;&#x952E;&#x51FD;&#x6570;&#xFF1A;writeFile"></a>

基础示例

```javascript
if(typeof require !== 'undefined') XLSX = require('xlsx');
/* output format determined by filename */
XLSX.writeFile(workbook, 'out.xlsb');
/* at this point, out.xlsb is a file that you can distribute */
```

## 4、SheetJS 实践——批量门店绑定 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-4&#x3001;SheetJS&#x5B9E;&#x8DF5;&#x2014;&#x2014;&#x6279;&#x91CF;&#x95E8;&#x5E97;&#x7ED1;&#x5B9A;"></a>

> 完整源码参考 finance-product-web：[http://git.dycjr.net/boss-cas/finance-product-web/blob/master/src/pages/product/dealer/batchBind.vue](http://git.dycjr.net/boss-cas/finance-product-web/blob/master/src/pages/product/dealer/batchBind.vue)

### 4.1 功能简介 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-4.1&#x529F;&#x80FD;&#x7B80;&#x4ECB;"></a>

批量绑定门店功能：

* [ ] 从页面上下载批量门店绑定的 excel 数据模板
* [ ] 从 excel 数据文件中读取某规则绑定的所有门店数据；
* [ ] 将读取到的数据 post 到服务端保存

### 4.2 使用 SheetJS 的 writeFile 实现模板下载 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-4.2&#x4F7F;&#x7528;SheetJS&#x7684;writeFile&#x5B9E;&#x73B0;&#x6A21;&#x677F;&#x4E0B;&#x8F7D;"></a>

```javascript
handleDownloadTemplate () {
  var workbook = XLSX.utils.book_new()
  var sheetName = '模板'
  /* make worksheet */
  var sheetData = [
    [ '类型', '规则ID', '规则名称', '门店编号', '门店名称', '操作(绑定/解绑)' ]
  ]
  var worksheet = XLSX.utils.aoa_to_sheet(sheetData)
  /* Add the worksheet to the workbook */
  XLSX.utils.book_append_sheet(workbook, worksheet, sheetName)
  XLSX.writeFile(workbook, '批量绑定门店-模板.xlsx')
}
```

### 4.3 使用 SheetJS 的 readFile 实现读取Excel 数据 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-4.3&#x4F7F;&#x7528;SheetJS&#x7684;readFile&#x5B9E;&#x73B0;&#x8BFB;&#x53D6;Excel&#x6570;&#x636E;"></a>

```javascript
importExcel (file) {
  // 文件格式校验 ...
 
  // 读取文件数据
  this.file2Xce(file).then(tabJson => {
    if (tabJson && tabJson.length > 0) {
      // 第一个 sheet 页的数据
      let excelData = tabJson[0].sheet
 
 
      // ...处理数据
    }
  }
}
 
 
file2Xce (file) {
  return new Promise(function (resolve, reject) {
    const reader = new FileReader()
    reader.onload = function (e) {
      const data = e.target.result
      let workbook = XLSX.read(data, {
        type: 'binary'
      })
      const result = []
      workbook.SheetNames.forEach((sheetName) => {
        result.push({
          sheetName: sheetName,
          sheet: XLSX.utils.sheet_to_json(workbook.Sheets[sheetName])
        })
      })
      resolve(result)
    }
    reader.readAsBinaryString(file.raw)
  })
}
```

### 4.4 将数据发送到服务端保存 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-4.4&#x5C06;&#x6570;&#x636E;&#x53D1;&#x9001;&#x5230;&#x670D;&#x52A1;&#x7AEF;&#x4FDD;&#x5B58;"></a>

```javascript
handleSubmit () {
  this.submitLoading = true
  this.allClear = false
  let param = {}
  param.type = this.type
  param.ruleDealerDTOList = this.dealerBindData
  this.axiosIns.post('/api/boss-finance-product/product/batch/dealer/bind', param).then(resp => {
    this.$message.success('操作成功')
    this.initData()
    this.type = null
    this.typeOfRules = {}
    this.submitLoading = false
  })
}
```

> 注意：
>
> 1）不用的前端框架发起 http 请求的方式不一样，不要滥用代码，以上为 vuejs 的实现；
>
> 2）理论上 http 的 post 请求是不限制数据大小的，但是服务端 web 容器可能对数据量大小会有默认的限制，需要自行设置，数据量过大可以考虑分批传输

## 5、小结 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-5&#x3001;&#x5C0F;&#x7ED3;"></a>

个人觉得 SheetJS 最大的优点：可以由前端页面直接读写 excel 文件，全程无需服务端的参与，对于部分类似数据导入导出的业务，可以直接前端搞定，体验很好。

使用的时候需要考虑好自己的业务场景，涉及数据量传输到服务端的需要关注数据量的大小，服务端的配置等。

## 6、更多精彩 <a id="id-&#x524D;&#x7AEF;&#xFF1A;SheetJs&#x8D4B;&#x80FD;&#x9875;&#x9762;&#x4E0E;Excel&#x6587;&#x4EF6;&#x4EA4;&#x4E92;-6&#x3001;&#x66F4;&#x591A;&#x7CBE;&#x5F69;"></a>

请多关注官方文档 [https://docs.sheetjs.com/\#sheetjs-js-xlsx](https://docs.sheetjs.com/#sheetjs-js-xlsx)

