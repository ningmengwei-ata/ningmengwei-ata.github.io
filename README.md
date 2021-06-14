





# 新闻爬取

一共选了四个新闻网站进行爬虫，分别是**人民网**、**新浪新闻**、**网易新闻**、**央视新闻**，其中主要爬取的网站是人民网和新浪新闻（爬取多天数据、总数据量达903条）并且将爬取结果存储在mysql中。

## 1.爬虫原理

1. 首先我们搜索主页面，获取我们想要的子网页的URL
   - 通过request请求，cheerio解析，each遍历
2. 搜索出我们子网页页面中我们需要的信息：标题，正文等
   - 通过request请求，cheerio解析
3. 将我们需要的信息保存下来，通过各种形式访问到这种信息
   - 建立fetch(文件对象)，输入文件信息，fs /mysql模块写入

## 2.分析网站链接格式和网页信息格式

### 爬取新浪新闻

如下所示新浪新闻的链接格式

https://cul.news.sina.com.cn/topline/2021-04-13/doc-ikmyaawa9440823.shtml

https://cul.news.sina.com.cn/stickynews/2021-04-14/doc-ikmxzfmk6721643.shtml

https://cul.news.sina.com.cn/stickynews/2021-04-14/doc-ikmxzfmk6720347.shtml

可以看到基本的格式为/年-月-日/doc-8位字符7位数字.shtml

Javascript匹配规则如下

\d 匹配一个非负整数， 等价于 [0-9]

\s 匹配一个空白字符

\w 匹配一个英文字母或数字，等价于[0-9a-zA-Z]

.  匹配除换行符以外的任意字符，等价于[\n]

所以构造正则表达式为

```js
var url_reg = /\/(\d{4})-(\d{2})-(\d{2})\/doc-(\w{8})(\d{7}).shtml/;
```

匹配时间规范化为年月日

```js
var regExp = /((\d{4}|\d{2})(\-|\/|\.)\d{1,2}\3\d{1,2})|(\d{4}年\d{1,2}月\d{1,2}日)/
```

匹配规则为年份可以用四位数字或者两位数字来表示，中间的分隔符可以是-  / .。\3是指引用前面的匹配,也即引用分隔符。也就是将如2021-04-1、2021/04/1、2021.04.1都转为2021年4月1日。

![image-20210417223155314](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210417223155314.png)

识别我们所要爬取的内容

由上图可以看到<meta name="description" content="在这里，遇见更大更好的未来！"所以获取keywords的方式为var keywords_format = " $('meta[name=\"keywords\"]').eq(0).attr(\"content\")";

```javascript
var seedURL_format = "$('a')";
var keywords_format = " $('meta[name=\"keywords\"]').eq(0).attr(\"content\")";
var source_format = " $('meta[name=\"mediaid\"]').eq(0).attr(\"content\")";
var title_format = "$('meta[property=\"og:title\"]').eq(0).attr(\"content\")";
var date_format = "$('meta[property=\"article:published_time\"]').eq(0).attr(\"content\")";
var author_format = "$('meta[property=\"article:author\"]').eq(0).attr(\"content\")";
var desc_format = " $('meta[property=\"og:description\"]').eq(0).attr(\"content\")";
var content_format = "$('.article').text()";
```

![image-20210430090153497](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210430090153497.png)

如上图可以看到class为article所以获取正文的方式为var content_format = "$('.article').text()";

### 爬取人民网新闻

网页链接格式如下

http://cpc.people.com.cn/n1/2021/0417/c164113-32080488.html

http://world.people.com.cn/n1/2021/0417/c1002-32080374.html

http://politics.people.com.cn/n1/2021/0417/c1001-32080356.html

可以看到格式为/年/月日/c至少4位数字-8位字符.html，所以构造正则表达式如下

```javascript
var url_reg = /\/(\d{4})\/(\d{4})\/(\w{1})(\d{4,})-(\d{8}).html/;
```

![image-20210418112936453](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210418112936453.png)

识别我们所要爬取的内容

由上图可以看到<meta name="author" content="105463"所以获取author的方式为var author_format = "$('meta[name=\"author\"]').eq(0).attr(\"content\")";

```javascript
var seedURL_format = "$('a')";
var keywords_format = " $('meta[name=\"keywords\"]').eq(0).attr(\"content\")";
var source_format = " $('meta[name=\"source\"]').eq(0).attr(\"content\")";
 
var title_format =  "$('title').text()";
var date_format = "$('meta[name=\"publishdate\"]').eq(0).attr(\"content\")";
var author_format = "$('meta[name=\"author\"]').eq(0).attr(\"content\")";
var desc_format = " $('meta[name=\"description\"]').eq(0).attr(\"content\")";
 
var content_format = "$('.rm_txt_con.cf').text()";
```

![image-20210418112509331](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210418112509331.png)

html中的class用空格分割，jquery用来匹配lclass就是要加.。所以这里匹配正文部分的代码为：

```js
var content_format = "$('.rm_txt_con.cf').text()";
```

### 央视新闻爬取

网页链接格式如下

https://news.cctv.com/2021/04/23/ARTIcvjyoOUcK7qm83KjKgka210423.shtml?spm=C96370.PPDB2vhvSivD.EJHwg9t6FrM7.1

https://news.cctv.com/2021/04/18/ARTIC2yfewl0c1gNbHXWNicB210418.shtml?spm=C94212.P4YnMod9m2uD.ENPMkWvfnaiV.88

https://news.cctv.com/2021/04/18/ARTI513XcI88poP3HstaeZ2B210418.shtml?spm=C94212.P4YnMod9m2uD.ENPMkWvfnaiV.314

https://news.cctv.com/2021/04/14/ARTIkp2Di57750wzEbwuUYDf210414.shtml?spm=C94212.PZd4MuV7QTb5.Euuu2IJOvZIL.206

https://news.cctv.com/2021/04/17/ARTIpxJaxJMR9jWGctPnuIBe210417.shtml?spm=C96370.PPDB2vhvSivD.EZ4sRtXz56aB.6

可以看到格式为/年/月/日/30位字母和数组的组合.shtml，所以构造正则表达式如下

var url_reg = /\/(\d{4})\/(\d{2})\/(\d{2})\/(\w{30})\.shtml/

识别我们所要爬取的内容

![image-20210430093957968](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210430093957968.png)

![image-20210430095559356](/Users/wangwenqing/Library/Application Support/typora-user-images/image-20210430095559356.png)

```javascript
var seedURL_format = "$('a')";
var keywords_format = " $('meta[name=\"keywords\"]').eq(0).attr(\"content\")";
var source_format = " $('meta[name=\"source\"]').eq(0).attr(\"content\")";
var title_format =  " $('meta[property=\"og:title\"]').eq(0).attr(\"content\")";
var date_format = "$('.info1').text()";
var author_format = " $('meta[name=\"author\"]').eq(0).attr(\"content\")";
var desc_format = " $('meta[property=\"og:description\"]').eq(0).attr(\"content\")";
var content_format = "$('.content_area').text()";
```

其中publish_date在之后需要进行特殊处理，取出｜之后的日期部分，处理代码如下

```javascript
var arr=fetch.publish_date.split("|");
        var trimLeft = /^\s+/
        fetch.publish_date=arr[1].replace(trimLeft,"").replace("年","-").replace("月","-").replace("日","");
```

### 网易新闻

https://www.163.com/news/article/G8MVPOON000189FH.html

https://www.163.com/news/article/G8MPODL30001899O.html

由上述链接易知网易新闻的链接规律，书写正则表达式

```javascript
var url_reg = /news\/article\/(\w{16}).html/;
```

同样可以通过谷歌显示网页源代码功能分析所要获取内容的javascript表达形式，代码如下：

```javascript
var seedURL_format = "$('a')";
var keywords_format = " $('meta[name=\"keywords\"]').eq(0).attr(\"content\")";
var source_format = " $('meta[property=\"twitter:creator\"]').eq(0).attr(\"content\")";
var title_format = "$('meta[property=\"og:title\"]').eq(0).attr(\"content\")";
var date_format = "$('meta[property=\"article:published_time\"]').eq(0).attr(\"content\")";
var author_format = "$('meta[name=\"author\"]').eq(0).attr(\"content\")";
var desc_format = " $('meta[name=\"description\"]').eq(0).attr(\"content\")";
var content_format = "$('.post_body').text()";
```

## 3.数据库设计

使用mysql数据库建表语句如下

```mysql
CREATE TABLE `fetches2` (
  `id_fetches` int(11)  NOT NULL AUTO_INCREMENT,
  `url` varchar(200) DEFAULT NULL,
  `source_name` varchar(200) DEFAULT NULL,
  `source_encoding` varchar(45) DEFAULT NULL,
  `title` varchar(400) DEFAULT NULL,
  `keywords` varchar(200) DEFAULT NULL,
  `author` varchar(200) DEFAULT NULL,
  `publish_date` date DEFAULT NULL,
  `crawltime` datetime DEFAULT NULL,
  `content` longtext,
  `createtime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id_fetches`),
  UNIQUE KEY `id_fetches_UNIQUE` (`id_fetches`),
  UNIQUE KEY `url_UNIQUE` (`url`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

新建一个mysql.js文件，在里面写上具体操作，然后在主爬虫程序中引入。

连接池创建

在代码中引入mysql模块，并编写数据库配置：

```javascript
var mysql = require("mysql");
var pool = mysql.createPool({
    host: '127.0.0.1',
    user: 'root',
    password: 'root',
    database: 'crawl'
});
```

在本实验中，主要是对数据库进行查询和插入的操作，并无更新和删除操作。
这里的两个函数，用于操作有参数和无参数的情况。

有参数的情况：

```javascript

var query = function(sql, sqlparam, callback) {
    pool.getConnection(function(err, conn) {
        if (err) {
            console.log(err)
            callback(err, null, null);
        } else {
            conn.query(sql, sqlparam, function(qerr, vals, fields) {
                conn.release(); //释放连接 
                callback(qerr, vals, fields); //事件驱动回调 
            });
        }
    });
};
```

无参数的情况

```javascript
var query_noparam = function(sql, callback) {
    pool.getConnection(function(err, conn) {
        if (err) {
            callback(err, null, null);
        } else {
            conn.query(sql, function(qerr, vals, fields) {
                conn.release(); //释放连接 
                callback(qerr, vals, fields); //事件驱动回调 
            });
        }
    });
};
```

## 4.爬虫具体代码实现

此处以爬取新浪新闻为例

首先，定义全局常量：

```javascript
var source_name = "新浪新闻";
var domain = 'https://news.sina.com.cn';
var myEncoding = "utf-8";
var seedURL = 'https://news.sina.com.cn/';
```

encoding格式需要根据网页链接特点调整，否则会出现写入数据库的数据是乱码的情况

接着，就是我们前面讲述的链接的正则表达式匹配以及获取页面内容

为防止爬虫被屏蔽，为爬虫写一个header，在爬取网页时假装设备访问

```javascript
var headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.65 Safari/537.36'
}
```

定义**request**函数，调用该函数能够访问指定的url并且能够设置回调函数来处理得到的html页面。截取过程是异步完成的：

```javascript
function request(url, callback) {
    var options = {
        url: url,
        encoding: null,
        //proxy: 'http://x.x.x.x:8080',
        headers: headers,
        timeout: 10000 //
    }
    myRequest(options, callback)
};
```

定义**seedget**函数，从种子页面抓取二级页面的url

```javascript
function seedget() {
    request(seedURL, function(err, res, body) { //读取种子页面
        // try {
        //用iconv转换编码
        if(body==undefined)return;
        var html = myIconv.decode(body, myEncoding);
        //console.log(html);
        //准备用cheerio解析html
        var $ = myCheerio.load(html, { decodeEntities: true });
        // } catch (e) { console.log('读种子页面并转码出错：' + e) };
        var seedurl_news;
        try {
            seedurl_news = eval(seedURL_format);
        } catch (e) { console.log('url列表所处的html块识别出错：' + e) };
        seedurl_news.each(function(i, e) { //遍历种子页面里所有的a链接
            var myURL = "";
            try {
                //得到具体新闻url
                var href = "";
                href = $(e).attr("href");
                // console.log(href);
                if (href == undefined) return;
                if (href.toLowerCase().indexOf('http://') >= 0||href.toLowerCase().indexOf('https://') >= 0) myURL = href; //http://开头的以及https://
                else if (href.startsWith('//')) myURL = 'http:' + href; ////开头的
                else myURL = seedURL.substr(0, seedURL.lastIndexOf('/') + 1) + href; //其他
                // console.log(myURL);
            } catch (e) { console.log('识别种子页面中的新闻链接出错：' + e) }

            if (!url_reg.test(myURL)) return; //检验是否符合新闻url的正则表达式
            //console.log(myURL);

            var fetch_url_Sql = 'select url from fetches where url=?';
            var fetch_url_Sql_Params = [myURL];
            mysql.query(fetch_url_Sql, fetch_url_Sql_Params, function(qerr, vals, fields) {
                // if (vals!=null)return;
                // vals.length > 0
                if (vals.length>0) {
                    console.log('URL duplicate!')
                } else newsGet(myURL); //读取新闻页面
            });
        });
    });
};

```

定义**newsGet**函数，传入的参数为二级页面URL，解析得到新闻的具体信息，包括title, publish_date, source_name, keywords, content等。其中fetch中存储的是在之前爬取信息的结构化存储：

```javascript
function newsGet(myURL) { //读取新闻页面
    request(myURL, function(err, res, body) { //读取新闻页面
        //try {
        var html_news = myIconv.decode(body, myEncoding); //用iconv转换编码
        //console.log(html_news);
        //准备用cheerio解析html_news
        var $ = myCheerio.load(html_news, { decodeEntities: true });
        myhtml = html_news;
        //} catch (e) {    console.log('读新闻页面并转码出错：' + e);};

        console.log("转码读取成功:" + myURL);
        //动态执行format字符串，构建json对象准备写入文件或数据库
        var fetch = {};
        fetch.title = "";
        fetch.content = "";
        fetch.publish_date = (new Date()).toFormat("YYYY-MM-DD");
        //fetch.html = myhtml;
        fetch.url = myURL;
        fetch.source_name = source_name;
        fetch.source_encoding = myEncoding; //编码
        fetch.crawltime = new Date();

        if (keywords_format == "") fetch.keywords = source_name; // eval(keywords_format);  //没有关键词就用sourcename
        else fetch.keywords = eval(keywords_format);

        if (title_format == "") fetch.title = ""
        else fetch.title = eval(title_format); //标题

        if (date_format != "") fetch.publish_date = eval(date_format); //刊登日期   
        console.log('date: ' + fetch.publish_date);
        if(fetch.publish_date == null) return;
        // fetch.publish_date =fetch.publish_date.substr(0,10);
        if(typeof(fetch.publish_time)=='string'){
            fetch.publish_date = regExp.exec(fetch.publish_date)[0];
            fetch.publish_date = fetch.publish_date.replace('年', '-')
            fetch.publish_date = fetch.publish_date.replace('月', '-')
            fetch.publish_date = fetch.publish_date.replace('日', '')
            fetch.publish_date = new Date(fetch.publish_date).toFormat("YYYY-MM-DD");
        }
        

        if (author_format == "") fetch.author = source_name; //eval(author_format);  //作者
        else fetch.author = eval(author_format);

        if (content_format == "") fetch.content = "";
        else fetch.content = eval(content_format).replace("\r\n" + fetch.author, ""); //内容,是否要去掉作者信息自行决定

        if (source_format == "") fetch.source = fetch.source_name;
        else fetch.source = eval(source_format).replace("\r\n", ""); //来源

        if (desc_format == "") fetch.desc = fetch.title;
        else{if(eval(desc_format)!=null)fetch.desc = eval(desc_format).replace("\r\n", "");}
        // else  //摘要    

        // var filename = source_name + "_" + (new Date()).toFormat("YYYY-MM-DD") +
        //     "_" + myURL.substr(myURL.lastIndexOf('/') + 1) + ".json";
        // ////存储json
        // fs.writeFileSync(filename, JSON.stringify(fetch));

        var fetchAddSql = 'INSERT INTO fetches(url,source_name,source_encoding,title,' +
            'keywords,author,publish_date,crawltime,content) VALUES(?,?,?,?,?,?,?,?,?)';
        var fetchAddSql_Params = [fetch.url, fetch.source_name, fetch.source_encoding,
            fetch.title, fetch.keywords, fetch.author, fetch.publish_date,
            fetch.crawltime.toFormat("YYYY-MM-DD HH24:MI:SS"), fetch.content
        ];

        //执行sql，数据库中fetch表里的url属性是unique的，不会把重复的url内容写入数据库
        mysql.query(fetchAddSql, fetchAddSql_Params, function(qerr, vals, fields) {
            if (qerr) {
                console.log(qerr);
            }
        }); //mysql写入
    });
}
```

## 5.存储结果展示

## 网站搭建

搭建过程分为一下几个部分：

1. 用express构建网站访问mysql
2. 用表格显示查询结果及分页
3. 前端优化
4. 时间热度分析

### 1.用express构建网站访问mysql

使用express脚手架来创建一个网站框架。
在terminal中输入：express -e search_site
将之前写好的mysql.js拷贝到该文件夹下并在文件夹内cmd运行：npm install mysql --save将mysql包安装到该项目中，并且将依赖项保存进package.json里
在search_site文件夹内cmd运行npm install。提示0vulnerability即为安装成功，完成网站搭建。若出现vulnerability不为0可考虑换源（如淘宝源）安装。

### 2.显示查询结果

#### 后端实现

以按关键词检索新闻的全文/关键词/标题中是否包含该词的后端为例。编写mysql语句判断前端勾选了全文/关键词/标题中的哪几项作为筛选项。

```javascript
router.get('/news_get', function(request, response) {
  //sql字符串和参数
  var fetchSql = "select url,source_name,title,author,publish_date " +
      "from fetches where ";
      var end=false;
      if (request.query.title=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" title like '%"+request.query.words + "%'";
      }
      if (request.query.keywords=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" keywords like '%"+request.query.words + "%'";
      }
      if (request.query.content=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" content like '%"+request.query.words + "%'";
      }
  
      fetchSql+=" order by publish_date DESC";
      fetchSql+=";";
  console.log(fetchSql);
  console.log("*****");
  mysql.query(fetchSql, function(err, result, fields) {
    if (err) throw err;
    console.log("*****");
      response.writeHead(200, {
          "Content-Type": "application/json"
      });
      response.write(JSON.stringify(result));
      response.end();
  });
});
```

#### 前端实现

实现复选框查询

```javascript
<form>
        <br> <input type="text" name="title_text" class="search">
        <input class="form1" type="checkbox" >title
        <input class="form2" type="checkbox">keywords
        <input class="form3" type="checkbox">content
        <!-- <input type="date" name="publish_date" value="" /> -->
        <input class="form-submit" type="button" value="search">
    </form>
```

获取复选框的勾选情况，如果没有勾选任何一个复选框给出提示。

```javascript
 $(document).ready(function() {
            $("input:button").click(function() {
                var title=$('input.form1').is(":checked");
                var keywords=$('input.form2').is(":checked");
                var content=$('input.form3').is(":checked");
              if (title.valueOf()==false && keywords.valueOf()==false && content.valueOf()==false)
                {
                    alert("请至少选择一项作为搜索范围");
                    return;
                }
```

实现直接通过html语言将后端获取的数据data以表格形式展示以及通过bootstraptable展示数据两种方式。这里展示用bootstraptable获取数据的代码。

```javascript
  $.get('/news_get?title='+ title +'&keywords='+keywords+'&content='+content+  '&words='+words, function(data) {
                    $("#record2").empty();

                var params ='/process_get?title='+title +'&keywords='+keywords+'&content='+content+  '&words='+words;
                $(function(){
                    console.log(params);
                    $("#record2").empty();
                    $('#record2').bootstrapTable('hideLoading');
                    $('#record2').bootstrapTable({
                        url:params,
                        method:'GET',
                        pagination:true,
                        sidePagination:'client',
                        pageSize:5,
                        striped : true,
                        sortable : true,
                        sortOrder:"asc",
                        showRefresh:true,
                        showLoading:false,
                        // search:true,
                        showToggle: true,
                        toolbar: '#toolbar',
                        showColumns : true,
                        columns:[{
                            field :'url',
                            title : 'url',
                            sortable : true
                        }, {
                            field:'title',
                            title:'title'
                        }, {
                            field:'source_name',
                            title:'source_name',
                            sortable : true
                        },{
                            field:'author',
                            title:'author'
                        },{
                            field:'publish_date',
                            title:'publish_date',
                            sortable : true

                        }]
                    })
                });
```

#### 对查询结果进行分页

首先引入bootstrap自带的css

```javascript
<link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">

<link rel="stylesheet" href="https://unpkg.com/bootstrap-table@1.16.0/dist/bootstrap-table.min.css">

<link rel="stylesheet" href="./stylesheets/search.css">
```

以及js

```javascript
<script src="https://code.jquery.com/jquery-3.3.1.min.js" integrity="sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8=" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.7/umd/popper.min.js" integrity="sha384-UO2eT0CpHqdSJQ6hJty5KVphtPhzWj9WO1clHTMGa3JDZwrnQq4sF86dIHNDz0W1" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" integrity="sha384-JjSmVgyd0p3pXB1rRibZUAYoIIy6OrQ6VrjIEaFf/nJGzIxFDsf4x0xIM+B07jRM" crossorigin="anonymous"></script>
    <script src="https://unpkg.com/bootstrap-table@1.16.0/dist/bootstrap-table.min.js"></script>
```

接着使用bootstraptable将输出的网页格式改为表格形式。

#### 优化页面

可用如下网址调配各按钮的颜色

https://www.bestcssbuttongenerator.com/#/20

修改 input checkbox(复选框) 默认的背景颜色

```css
 input[type=checkbox]{
     cursor: pointer;
     position: relative;
     width: 15px;
     height: 15px;
     font-size: 14px;
}

input[type=checkbox]::after{
     position: absolute;
     top: 0;
     background-color: #dbe6c4;
     color: white;
     width: 15px;
     height: 15px;
     display: inline-block;
     visibility: visible;
     text-align: center;
     content: ' ';
}
        
input[type=checkbox]:checked::after{
     content: "✓";
     font-size: 12px;
     font-weight: bold;
}
```

搜索框颜色修改

#### 时间热度分析

返回每天对应关键词的搜索数量，在前端分别用表格以及echarts图的形式展示。

##### 后端

```javascript
router.get('/timehot', function(request, response) {
  //sql字符串和参数
  var fetchSql = "select publish_date,count(*) as num " +
      "from fetches where ";
      var end=false;
      if (request.query.title=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" title like '%"+request.query.words + "%'";
      }
      if (request.query.keywords=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" keywords like '%"+request.query.words + "%'";
      }
      if (request.query.content=="true"){
          if (!end){
            end=true;
          }
          else fetchSql+=" or"
          fetchSql+=" content like '%"+request.query.words + "%'";
      }
  
      fetchSql+=" group by publish_date ";
      fetchSql+=" order by publish_date ASC";
      fetchSql+=";";
  console.log(fetchSql);
  console.log("*****");
  mysql.query(fetchSql, function(err, result, fields) {
    if (err) throw err;
    console.log("*****");
      response.writeHead(200, {
          "Content-Type": "application/json"
      });
      response.write(JSON.stringify(result));
      response.end();
  });
});
```

##### 前端

echarts图，首先构建画布。

```javascript
<div id="main" style="width: 600px;height:400px;"></div>
    <script type="text/javascript">
        // 基于准备好的dom，初始化echarts实例
        var myChart = echarts.init(document.getElementById('main'));
```

option中设置画图的数据

```javascript
myChart.resize();
var option={
        //标题
        title:{
          text:'新闻时序图'
        },
        //工具箱
        //保存图片
        toolbox:{
            show:true,
            feature:{
                saveAsImage:{
                    show:true
                }
            }
        },
        dataset: {
                    // 提供一份数据。
                    source:data
                },
        //图例-每一条数据的名字叫销量
        // legend:{
        //     data:['销量']
        // },
        //x轴
        xAxis: {type: 'category', gridIndex:0,
        axisLabel: {
                                           interval:0,
                                           rotate:15
                                        }},
       
        //y轴没有显式设置，根据值自动生成y轴
        yAxis:{type:'value', gridIndex:0},
        //数据-data是最终要显示的数据
        series:[{
            itemStyle : { normal: {label : {show: true}}},
            
            type:'line',
            encode:{
                            x:'publish_date',
                            y :'num'
                        }
        }]
    };
    myChart.setOption(option);
```



## 错误处理

### TypeError [ERR_INVALID_ARG_TYPE]: 报错

1. npm cache clean --force

2. npm install

   清除缓存并重装依赖 问题解决
   
   如还不能解决 这个报错的意思是参数类型不断 先判断是否为string类型，只有当是string类型时才进行之后的操作

### 爬虫得到的数据出现中文乱码

将myEncoding由utf-8改为gbk即可。

### MAC mysql密码输入正确的情况下报错

ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

https://blog.csdn.net/qq1808814025/article/details/112236959

最简单直接的方法 重装mysql 一切解决

mysql修改密码 mysql> alter user 'root'@'localhost' identified by 'root';

### npm audit fix 解决方法

found 4 vulnerabilities (3 **low**, 1 critical) in 126 scanned packages

 4 vulnerabilities require manual review. See the full report for details.

https://www.it610.com/article/1287669118148849664.htm



