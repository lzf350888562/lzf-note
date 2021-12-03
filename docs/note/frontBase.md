# dom

**innerHTML和innerText**

```js
// 1. 更改元素里面内容
a.innerHTML = "<p>测试</p>";
// 2. 插入一个<a>元素
a.appendChild(document.createElement(`a`));
// 3. 删除第一个元素，在这里是前面的<p>测试</p>
a.removeChild(a.firstChild);
```

```js
// 1. 更改元素里面内容
a.innerHTML = "<p>测试</p>";
// 2. 插入一个<a>元素
a.innerHTML = "<p>测试</p><a></a>";
// 3. 删除第一个元素，在这里是前面的<p>测试</p>
a.innerHTML = "<a></a>";
```

```js
<form>
  Name:
  <p id="name-value"></p>
  <input type="text" name="name" id="name-input" />
  Email:
  <p id="email-value"></p>
  <input type="email" name="email" id="email-input" />
  <input type="submit" />
</form>

var nameInputEl = document.getElementById("name-input");
var emailInputEl = document.getElementById("email-input");
// 监听输入事件，此时 updateValue 函数未定义
nameInputEl.addEventListener("input", updateNameValue);
emailInputEl.addEventListener("input", updateEmailValue);

var nameValueEl = document.getElementById("name-value");
var emailValueEl = document.getElementById("email-value");
// 定义 updateValue 函数，用来更新页面内容
function updateNameValue(e) {
  nameValueEl.innerText = e.srcElement.value;
}
function updateEmailValue(e) {
  emailValueEl.innerText = e.srcElement.value;
}
```

# nodejs

## 模块化

1.js

```
exports.add=function(a,b){
	return a+b;
}
```

2.js

每个模块内部，module变量代表当前模块。这个变量是一个对象，它的exports属性（即module.exports）是对外 的接口。加载某个模块，其实是加载该模块的module.exports属性。

```
//引入模块1.js
var demo= require('./demo3_1');
console.log(demo.add(400,600));
```

内置模块

```
//http是内置模块
var http = require('http');
http.createServer(function (request, response) {
// 发送 HTTP 头部
// HTTP 状态值: 200 : OK
// 内容类型: text/plain
response.writeHead(200, {'Content-Type': 'text/plain'});
// 发送响应数据 "Hello World"
response.end('Hello World\n');
}).listen(8888);
// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8888/');
```

## 服务端渲染

```
var http = require('http');
http.createServer(function (request, response) {
	response.writeHead(200, {'Content-Type': 'text/plain'});
	for(var i=0;i<10;i++){
		response.write('Hello World\n');
	}
	response.end('');
}).listen(8888);
// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8888/')
```

运行后浏览器查看源代码发现并没有for循环语句,说明这是服务端渲染,类似jsp

## 接收参数

```
var http = require("http");
var url = require("url");
//创建服务，监听8888端口
http.createServer(function (request, response) {
	response.writeHead(200, {"Content-Type":"text/plain"});
	var params = url.parse(request.url, true).query;
	for (var key in params) {
		response.write( key + " = " + params[key]);
		response.write("\n");
	}
	response.end("");
}).listen(8888);
console.log("Server running at http://127.0.0.1:8888 ")
```

访问 http://127.0.0.1:8888?id=123&name=itcast



# 部分引入和部分获取

js**文件**

```
export const getLocal = (name) => {
  return localStorage.getItem(name)
}
```

引入

```
import { getLocal } from '@/common/js/utils'
```

**组件**

```
import { Toast } from 'vant'
```

**部分获取**

部分获取Promise的.then的回调参数

```
export function getHome(params) {
  return axios.get('/index-infos');
}
```

```
  const { data } = await getHome()
```

嵌套使用

```
      const { data, data: { list } } = await search({ pageNumber: this.page, goodsCategoryId: categoryId, keyword: this.keyword, orderBy: this.orderBy })

```

# 加密解密

## JSEncrypt

访问密钥对生成 http://web.chacuo.net/netrsakeypair

```
import JSEncrypt from 'jsencrypt/bin/jsencrypt.min'

const publicKey = '...'

const privateKey = '...'

// 加密
export function encrypt(txt) {
  const encryptor = new JSEncrypt()
  encryptor.setPublicKey(publicKey) // 设置公钥
  return encryptor.encrypt(txt) // 对数据进行加密
}

// 解密
export function decrypt(txt) {
  const encryptor = new JSEncrypt()
  encryptor.setPrivateKey(privateKey) // 设置私钥
  return encryptor.decrypt(txt) // 对数据进行解密
}
```



# cookie-localstorage-sessionstorage

| 特性           | Cookie                                                       | localStorage                                                | sessionStorage                               |
| -------------- | ------------------------------------------------------------ | ----------------------------------------------------------- | -------------------------------------------- |
| 数据的生命期   | 一般由服务器生成，可设置失效时间。如果在浏览器端生成Cookie，默认是关闭浏览器后失效 | 除非被清除，否则永久保存                                    | 仅在当前会话下有效，关闭页面或浏览器后被清除 |
| 存放数据大小   | 4K左右                                                       | 一般为5MB                                                   |                                              |
| 与服务器端通信 | 每次都会携带在HTTP头中，如果使用cookie保存过多数据会带来性能问题 | 仅在客户端（即浏览器）中保存，不参与和服务器的通信          |                                              |
| 易用性         | 需要程序员自己封装，源生的Cookie接口不友好                   | 源生接口可以接受，亦可再次封装来对Object和Array有更好的支持 |                                              |

应用场景

有了对上面这些差别的直观理解，我们就可以讨论三者的应用场景了。

因为考虑到每个 HTTP 请求都会带着 Cookie 的信息，所以 Cookie 当然是能精简就精简啦，比较常用的一个应用场景就是判断用户是否登录。针对登录过的用户，服务器端会在他登录时往 Cookie 中插入一段加密过的唯一辨识单一用户的辨识码，下次只要读取这个值就可以判断当前用户是否登录啦。曾经还使用 Cookie 来保存用户在电商网站的购物车信息，如今有了 localStorage，似乎在这个方面也可以给 Cookie 放个假了~

而另一方面 localStorage 接替了 Cookie 管理购物车的工作，同时也能胜任其他一些工作。比如HTML5游戏通常会产生一些本地数据，localStorage 也是非常适用的。如果遇到一些内容特别多的表单，为了优化用户体验，我们可能要把表单页面拆分成多个子页面，然后按步骤引导用户填写。这时候 sessionStorage 的作用就发挥出来了。

安全性的考虑

需要注意的是，不是什么数据都适合放在 Cookie、localStorage 和 sessionStorage 中的。使用它们的时候，需要时刻注意是否有代码存在 XSS 注入的风险。因为只要打开控制台，你就随意修改它们的值，也就是说如果你的网站中有 XSS 的风险，它们就能对你的 localStorage 肆意妄为。所以千万不要用它们存储你系统中的敏感数据。

**使用**

```
window.localStorage.setItem(key,value)
window.localStorage.removeItem(key)

import Cookies from "js-cookie";
Cookies.get("username");
Cookies.set(TokenKey, token)
```





# 图片

## file-saver图片下载

```
import {saveAs} from 'file-saver';

saveAs(this.picture.url, this.pictureName);
```

## vue-cropper图像处理

### html2canvas图像合成

```
<div id="previewBox" ref="previewBox">
	<div class="preview" ref="preview" />
	<div :style="textStyle">{{ textConfig.inputText }}</div>
</div>

 <vue-cropper
    ref="cropper"
    containerStyle="width:100%"
    :key="picture?.url"
    :src="picture?.url"
    :alt="picture?.name"
    :auto-crop-area="1"
    :min-container-height="200"
    preview=".preview"
    @crop="onCrop"
/>


		//合成图片并下载
      html2canvas(document.getElementById('previewBox'), {
        useCORS: true,
      }).then((canvas) => {
        this.finalImg = canvas.toDataURL("image/png");
        saveAs(this.finalImg, this.pictureName);
      });
```

处理函数

```
	flipX() {
      const dom = this.$refs.flipX;
      let scale = dom.getAttribute('data-scale');
      scale = scale ? -scale : -1;
      this.$refs.cropper.scaleX(scale);
      dom.setAttribute('data-scale', scale);
    },
    flipY() {
      const dom = this.$refs.flipY;
      let scale = dom.getAttribute('data-scale');
      scale = scale ? -scale : -1;
      this.$refs.cropper.scaleY(scale);
      dom.setAttribute('data-scale', scale);
    },
    move(offsetX, offsetY) {
      this.$refs.cropper.move(offsetX, offsetY);
    },
    reset() {
      this.$refs.cropper.reset();
    },
    rotate(deg) {
      this.$refs.cropper.rotate(deg);
    },
    zoom(percent) {
      this.$refs.cropper.relativeZoom(percent);
    },
    onCrop() {
      this.$refs.previewBox.style.width = this.$refs.preview.style.width;
    }
```



# 网页中加入播放器

**Aplayer**官网文档：https://aplayer.js.org/#/

**Metingjs**官网文档：https://github.com/metowolf/MetingJS

**Aplayer**是一个功能强大的HTML5音乐播放器，**Metingjs**基于**Aplayer**插件封装好的插件，开箱即用。

| Version | API Status | APlayer |
| ------- | ---------- | ------- |
| 1.2.x   | Supported  | 1.10.0  |
| 2.0.x   | Latest     | 1.10.0  |

```
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8" />
		<title></title>	
	<!-- require APlayer -->
	<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/aplayer/dist/APlayer.min.css">
	<script src="https://cdn.jsdelivr.net/npm/aplayer/dist/APlayer.min.js"></script>
	<!-- require MetingJS -->
	<script src="https://cdn.jsdelivr.net/npm/meting@2.0.1/dist/Meting.min.js"></script>
	</head>
	<body>

<meting-js 
	server="netease" 
	type="playlist" 
	id="60198"
	fixed="true" 
	autoplay="true"
	loop="all"
	order="random"
	preload="auto"
	list-folded="ture"
	list-max-height="500px"
	lrc-type="1">
</meting-js>

	</body>
</html>
解析:
 解析：server="netease" type="playlist" id="60198"

server指音乐平台，netease指网易云音乐， type类型，playlist列表，id指歌曲的i或者专辑或列表外链id
因此重点在于指定平台，指定外链id
```

详细参数

| 选项                        | 默认        | 描述                                                         |
| --------------------------- | ----------- | ------------------------------------------------------------ |
| id(编号)                    | **require** | 歌曲ID /播放列表ID /专辑ID /搜索关键字                       |
| server(平台)                | **require** | 音乐平台：`netease`，`tencent`，`kugou`，`xiami`，`baidu`    |
| type（类型）                | **require** | `song`，`playlist`，`album`，`search`，`artist`              |
| auto（支持类种 类）         | options     | 音乐链接，支持：`netease`，`tencent`，`xiami`                |
| fixed（固定模式）           | `false`     | 启用固定模式，默认`false`                                    |
| mini（迷你模式）            | `false`     | 启用迷你模式,默认`false`                                     |
| autoplay（自动播放）        | `false`     | 音频自动播放，默认`false`                                    |
| theme(主题颜色)             | `#2980b9`   | 默认`#2980b9`                                                |
| loop（循环）                | `all`       | 播放器循环播放，值：“all”，one”，“none”                      |
| order(顺序)                 | `list`      | 播放器播放顺序，值：“list”，“random”                         |
| preload(加载)               | `auto`      | 值：“none”，“metadata”，“'auto”                              |
| volume（声量）              | `0.7`       | 默认音量，请注意播放器会记住用户设置，用户自己设置音量后默认音量将不起作用 |
| mutex（限制）               | `true`      | 防止同时播放多个玩家，在该玩家开始播放时暂停其他玩家         |
| lrc-type（歌词）            | `0`         | 歌词显示                                                     |
| list-folded（列表折叠）     | `false`     | 指示列表是否应该首先折叠                                     |
| list-max-height（最大高度） | `340px`     | 列出最大高度                                                 |
| storage-name（储存名称）    | `metingjs`  | 存储播放器设置的localStorage键                               |

注意:有些歌曲列表id无法获取,比如网易云的我喜欢的音乐需要登录才能获取.

可通过https://api.imjad.cn/cloudmusic测试(cloudmusicapi)

网易云测试歌曲列表例子:https://api.imjad.cn/cloudmusic/?type=playlist&id=6887284169

# 富文本编辑器

## wangEditor

```
<div ref='editor'></div>
```

```
instance = new WangEditor(editor.value)
      instance.config.showLinkImg = false
      instance.config.showLinkImgAlt = false
      instance.config.showLinkImgHref = false
      instance.config.uploadImgMaxSize = 2 * 1024 * 1024 // 2M
      instance.config.uploadFileName = 'file'
      instance.config.uploadImgHeaders = {
        token: state.token
      }
      // 图片返回格式不同，需要自定义返回格式
      instance.config.uploadImgHooks = {
        // 图片上传并返回了结果，想要自己把图片插入到编辑器中
        // 例如服务器端返回的不是 { errno: 0, data: [...] } 这种格式，可使用 customInsert
        customInsert: function(insertImgFn, result) {
          console.log('result', result)
          // result 即服务端返回的接口
          // insertImgFn 可把图片插入到编辑器，传入图片 src ，执行函数即可
          if (result.data && result.data.length) {
            result.data.forEach(item => insertImgFn(item))
          }
        }
      }
      instance.config.uploadImgServer = uploadImgsServer
      Object.assign(instance.config, {
        onchange() {
          console.log('change')
        },
      })
      instance.create()
```

手动设置编辑器内容

```
instance.txt.html(goods.goodsDetailContent)
```

摧毁

```
instance.destroy()
instance = null
```



# ajax

Ajax 是一种在无需重新加载整个网页的情况下，能够更新部分网页的技术。

原生实现方式：

```
//创建核心对象
var xmlhttp;
if (window.XMLHttpRequest){// code for IE7+, Firefox, Chrome, Opera, Safari
	xmlhttp=new XMLHttpRequest();
}else{// code for IE6, IE5
	xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
}
//建立连接
//参数：
//1. 请求方式：GET、POST
//get方式，请求参数在URL后边拼接。send方法为空参
//post方式，请求参数在send方法中定义
//2. 请求的URL：
//3. 同步或异步请求：true（异步）或 false（同步）
xmlhttp.open("GET","ajaxServlet?username=tom",true);
//发送请求
xmlhttp.send();
//接受并处理来自服务器的响应结果,获取方式 ：xmlhttp.responseText
xmlhttp.onreadystatechange=function()
{
	//判断readyState就绪状态是否为4，判断status响应状态码是否为200
	if (xmlhttp.readyState==4 && xmlhttp.status==200){
	    //获取服务器的响应结果
	    var responseText = xmlhttp.responseText;
	    alert(responseText);
	}
}
```

JQuery方式：

1.$.ajax()

```
$.ajax({
	url:"ajaxServlet1111" , 		// 请求路径
	type:"POST" , 					//请求方式
	data: "username=jack&age=23",	//请求参数
	data:{"username":"jack","age":23},
	success:function (data) {
	    alert(data);
	},						//响应成功后的回调函数
	error:function () {
	    alert("出错啦...")
	},			//表示如果请求响应出现错误，会执行的回调函数
	dataType:"text"//设置接受到的响应数据的格式
	});
```

2.$.get()

```
$.get(url, [data], [callback], [type])
```

3.$.post()

```
$.post(url, [data], [callback], [type])
```

> ajax不能做页面redirect和forward

# 网页静态化

=模板+数据

**freemarker**

模板文件静态化

```
//生成配置类
Configuration configuration = new Configuration(Configuration.getVersion());
//设置模板路径
String classpath = this.getClass().getResource("/").getPath();
configuration.setDirectoryForTemplateLoading(new File(classpath + "/templates/"));
//设置字符集
configuration.setDefaultEncoding("utf‐8");
//加载模板
Template template = configuration.getTemplate("test1.ftl");
//数据模型
Map<String,Object> map = new HashMap<>();
map.put("name","黑马程序员");
//静态化
String content = FreeMarkerTemplateUtils.processTemplateIntoString(template, map);
//静态化内容
System.out.println(content);
InputStream inputStream = IOUtils.toInputStream(content);
//输出文件
FileOutputStream fileOutputStream = new FileOutputStream(new File("d:/test1.html"));
int copy = IOUtils.copy(inputStream, fileOutputStream);

```

模板字符串静态化

```
//设置模板路径
String classpath = this.getClass().getResource("/").getPath();
configuration.setDirectoryForTemplateLoading(new File(classpath + "/templates/"));
//设置字符集
configuration.setDefaultEncoding("utf‐8");
//加载模板
Template template = configuration.getTemplate("test1.ftl");
//数据模型
Map<String,Object> map = new HashMap<>();
map.put("name","黑马程序员");
//静态化
String content = FreeMarkerTemplateUtils.processTemplateIntoString(template, map);
//静态化内容
System.out.println(content);
InputStream inputStream = IOUtils.toInputStream(content);
//输出文件
FileOutputStream fileOutputStream = new FileOutputStream(new File("d:/test1.html"));
int copy = IOUtils.copy(inputStream, fileOutputStream);
```



# xml

**CDATA**区

```
CDATA区：在该区域中的数据会被原样展示
格式：  <![CDATA[ 数据 ]]> 
```

**约束**

1.DTD:一种简单的约束技术

引入方式：

```
本地：<!DOCTYPE 根标签名 SYSTEM "dtd文件的位置">
网络：<!DOCTYPE 根标签名 PUBLIC "dtd文件名字" "dtd文件的位置URL">
```

如：

Student.dtd

```
<!ELEMENT students (student*) >
<!ELEMENT student (name,age,sex)>
<!ELEMENT name (#PCDATA)>
<!ELEMENT age (#PCDATA)>
<!ELEMENT sex (#PCDATA)>
<!ATTLIST student number ID #REQUIRED>
```

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE students SYSTEM "student.dtd">
<students>
	<student number="itcast_0001">
		<name>tom</name>
		<age>18</age>
		<sex>male</sex>
	</student>	
</students>
```

2.Schema:一种复杂的约束技术

引入步骤

```
1.填写xml文档的根元素
2.引入xsi前缀.  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
3.引入xsd文件命名空间.  xsi:schemaLocation="http://www.itcast.cn/xml  student.xsd"
4.为每一个xsd约束声明一个前缀,作为标识  xmlns="http://www.itcast.cn/xml" 
```

如：

Student.xsd

```
<?xml version="1.0"?>
<xsd:schema xmlns="http://www.itcast.cn/xml"
        xmlns:xsd="http://www.w3.org/2001/XMLSchema"
        targetNamespace="http://www.itcast.cn/xml" elementFormDefault="qualified">
    <xsd:element name="students" type="studentsType"/>
    <xsd:complexType name="studentsType">
        <xsd:sequence>
            <xsd:element name="student" type="studentType" minOccurs="0" maxOccurs="unbounded"/>
        </xsd:sequence>
    </xsd:complexType>
    <xsd:complexType name="studentType">
        <xsd:sequence>
            <xsd:element name="name" type="xsd:string"/>
            <xsd:element name="age" type="ageType" />
            <xsd:element name="sex" type="sexType" />
        </xsd:sequence>
        <xsd:attribute name="number" type="numberType" use="required"/>
    </xsd:complexType>
    <xsd:simpleType name="sexType">
        <xsd:restriction base="xsd:string">
            <xsd:enumeration value="male"/>
            <xsd:enumeration value="female"/>
        </xsd:restriction>
    </xsd:simpleType>
    <xsd:simpleType name="ageType">
        <xsd:restriction base="xsd:integer">
            <xsd:minInclusive value="0"/>
            <xsd:maxInclusive value="256"/>
        </xsd:restriction>
    </xsd:simpleType>
    <xsd:simpleType name="numberType">
        <xsd:restriction base="xsd:string">
            <xsd:pattern value="heima_\d{4}"/>
        </xsd:restriction>
    </xsd:simpleType>
</xsd:schema> 
```

```
<?xml version="1.0" encoding="UTF-8" ?>
 <students   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 			 xmlns="http://www.itcast.cn/xml" 
 		   xsi:schemaLocation="http://www.itcast.cn/xml  student.xsd"
 		    >
 	<student number="heima_0001">
 		<name>tom</name>
 		<age>18</age>
 		<sex>male</sex>
 	</student>	 
 </students>
```

**解析**

解析方式：

DOM：将标记语言文档一次性加载进内存，在内存中形成一颗dom树

优点：操作方便，可以对文档进行CRUD的所有操作；缺点：占内存

SAX：逐行读取，基于事件驱动的。

优点：不占内存；缺点：只能读取，不能增删改

解析器：

```
1. JAXP：sun公司提供的解析器，支持dom和sax两种思想
2. DOM4J：一款非常优秀的解析器
3. Jsoup：jsoup 是一款Java 的HTML解析器，可直接解析某个URL地址、HTML文本内容。它提供了一套非常省力的API，可通过DOM，CSS以及类似于jQuery的操作方法来取出和操作数据。
4. PULL：Android操作系统内置的解析器，sax方式的。
```

**Jsoup**

```
String path = XXX.class.getClassLoader().getResource("student.xml").getPath();
//解析xml文档，加载文档进内存，获取dom树--->Document
Document document = Jsoup.parse(new File(path), "utf-8");
//获取元素对象 Element
Elements elements = document.getElementsByTag("name");
System.out.println(elements.size());
//获取第一个name的Element对象
Element element = elements.get(0);
//获取数据
String name = element.text();
System.out.println(name);
```

Document：文档对象。代表内存中的dom树

Elements：元素Element对象的集合。可以当做 ArrayList<Element>来使用

Element：元素对象

获取方式

```
getElementById​(String id)：根据id属性值获取唯一的element对象
getElementsByTag​(String tagName)：根据标签名称获取元素对象集合
getElementsByAttribute​(String key)：根据属性名称获取元素对象集合
getElementsByAttributeValue​(String key, String value)：根据对应的属性名和属性值获取元素对象集合
```

获取属性值

```
String attr(String key)：根据属性名称获取属性值
```

获取文本内容

```
String text():获取文本内容
String html():获取标签体的所有内容(包括字标签的字符串内容)
```

selector查询方式

```
Elements	select​(String cssQuery)
```

**XPath**

XPath即为XML路径语言，它是一种用来确定XML（标准通用标记语言的子集）文档中某部分位置的语言.详细语法需要查询官方

```
String path = XXX.class.getClassLoader().getResource("student.xml").getPath();
Document document = Jsoup.parse(new File(path), "utf-8");
//根据document对象，创建JXDocument对象
JXDocument jxDocument = new JXDocument(document);
//结合xpath语法查询
Elements elements = document.getElementsByTag("name");
System.out.println(elements.size());
//查询所有student标签
List<JXNode> jxNodes = jxDocument.selN("//student");
for (JXNode jxNode : jxNodes) {
	System.out.println(jxNode);
}
//查询所有student标签下的name标签
List<JXNode> jxNodes2 = jxDocument.selN("//student/name");
//查询student标签下带有id属性的name标签
List<JXNode> jxNodes3 = jxDocument.selN("//student/name[@id]");
//查询student标签下带有id属性的name标签 并且id属性值为itcast		
List<JXNode> jxNodes4 = jxDocument.selN("//student/name[@id='itcast']");
```

