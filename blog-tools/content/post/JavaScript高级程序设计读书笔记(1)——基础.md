---
title: "JavaScript高级程序设计读书笔记(1)——基础" 
date: 2022-06-22T20:32:39+08:00
draft: false
tags: ["web","javascript"]
categories:
  - "web"
  - "javascript"
---





# JavaScript概述

`JavaScript`和 `ECMAScript` 基本上是同义词，但 JavaScript远远不限于`ECMA-262` 所定义的那样。 没错，完整的 JavaScript 实现包含以下几个部分：

1. 核心(`ECMAScript`)
2. 文档对象模型(`DOM`)
3. 浏览器对象模型(`BOM`)



## ECMAScript

`ECMAScript`，即 `ECMA-262` 定义的语言，并不局限于 Web 浏览器。事实上，这门语言没有输入和 输出之类的方法。`ECMA-262` 将这门语言作为一个基准来定义，以便在它之上再构建更稳健的脚本语言。 Web 浏览器只是 `ECMAScript` 实现可能存在的一种宿主环境（host environment）。宿主环境提供 `ECMAScript `的基准实现和与环境自身交互必需的扩展。扩展（比如 `DOM`）使用 `ECMAScript` 核心类型 和语法，提供特定于环境的额外功能。其他宿主环境还有服务器端 `JavaScript` 平台 `Node.js` 和即将被淘汰 的 `Adobe Flash`。



如果不涉及浏览器的话，`ECMA-262` 到底定义了什么？在基本的层面，它描述这门语言的如下部分：

- 语法
- 类型
- 语句
- 关键字
- 保留字
- 操作符
- 全局对象



`ECMAScript` 只是对实现这个规范描述的所有方面的一门语言的称呼。`JavaScript` 实现了 `ECMAScript`，而 `Adobe ActionScript` 同样也实现了 `ECMAScript`。



## DOM

文档对象模型（DOM，Document Object Model）是一个应用编程接口（API），用于在 HTML 中使 用扩展的 XML。DOM 将整个页面抽象为一组分层节点。HTML 或 XML 页面的每个组成部分都是一种 节点，包含不同的数据。比如下面的 HTML 页面：

```html
<html> 
 <head> 
 <title>Sample Page</title> 
 </head> 
 <body> 
 <p> Hello World!</p> 
 </body> 
</html> 
```

这些代码通过 DOM 可以表示为一组分层节点，如图所示。

![](https://www.punklu.tech/assets/DOM-1.png)



`DOM` 通过创建表示文档的树，让开发者可以随心所欲地控制网页的内容和结构。使用 `DOM API`， 可以轻松地删除、添加、替换、修改节点。



## BOM

IE3 和 Netscape Navigator 3 提供了浏览器对象模型（BOM） API，用于支持访问和操作浏览器的窗 口。使用 BOM，开发者可以操控浏览器显示页面之外的部分。而 BOM 真正独一无二的地方，当然也是 问题最多的地方，就是它是唯一一个没有相关标准的 JavaScript 实现。HTML5 改变了这个局面，这个版 本的 HTML 以正式规范的形式涵盖了尽可能多的 BOM 特性。由于 HTML5 的出现，之前很多与 BOM 有关的问题都迎刃而解了。



总体来说，BOM 主要针对浏览器窗口和子窗口（frame），不过人们通常会把任何特定于浏览器的 扩展都归在 BOM 的范畴内。比如，下面就是这样一些扩展：

- 弹出新浏览器窗口的能力
- 移动、缩放和关闭浏览器窗口的能力
- navigator对象，提供关于浏览器的详尽信息
- location对象，提供浏览器加载页面的详尽信息
- screen对象，提供关于用户屏幕分辨率的详尽信息
- performance对象，提供浏览器内存占用，导航行为和时间统计的详尽信息
- 对cookie的支持
- 其他自定义对象，如`XMLHttpRequest` 和 IE 的 `ActiveXObject`。



因为在很长时间内都没有标准，所以每个浏览器实现的都是自己的 BOM。有一些所谓的事实标准， 比如对于 window 对象和 navigator 对象，每个浏览器都会给它们定义自己的属性和方法。现在有了 HTML5，BOM 的实现细节应该会日趋一致。



# HTML中的JavaScript



## \<script>元素

将 JavaScript 插入 HTML 的主要方法是使用\<script>元素。\<script>元素有下列 8 个属性。

1. async

   可选。表示应该立即开始下载脚本，但不能阻止其他页面动作，比如下载资源或等待其他脚本加载。只对外部脚本文件有效。

2. charset

   可选。使用 src 属性指定的代码字符集。这个属性很少使用，因为大多数浏览器不在乎它的值。

3. crossorigin

   可选。配置相关请求的CORS（跨源资源共享）设置。默认不使用CORS。crossorigin=  "anonymous"配置文件请求不必设置凭据标志。crossorigin="use-credentials"设置凭据 标志，意味着出站请求会包含凭据。

4. defer

   可选。表示脚本可以延迟到文档完全被解析和显示之后再执行。只对外部脚本文件有效。 在 IE7 及更早的版本中，对行内脚本也可以指定这个属性。

5. integrity

   可选。允许比对接收到的资源和指定的加密签名以验证子资源完整性（SRI， Subresource Integrity）。如果接收到的资源的签名与这个属性指定的签名不匹配，则页面会报错， 脚本不会执行。这个属性可以用于确保内容分发网络（CDN，Content Delivery Network）不会提供恶意内容。

6. language

   废弃。最初用于表示代码块中的脚本语言（如"JavaScript"、"JavaScript 1.2" 或"VBScript"）。大多数浏览器都会忽略这个属性，不应该再使用它。

7. src

   可选。表示包含要执行的代码的外部文件。

8. type

   可选。代替 language，表示代码块中脚本语言的内容类型（也称 MIME 类型）。按照惯例，这个值始终都是"text/javascript"，尽管"text/javascript"和"text/ecmascript" 都已经废弃了。JavaScript 文件的 MIME 类型通常是"application/x-javascript"，不过给 type 属性这个值有可能导致脚本被忽略。在非 IE 的浏览器中有效的其他值还有 "application/javascript"和"application/ecmascript"。`如果这个值是 module，则代 码会被当成 ES6 模块，而且只有这时候代码中才能出现 import 和 export 关键字。`



\<script>元素的一个最为强大、同时也备受争议的特性是，它可以包含来自外部域的 `JavaScript`文件。跟\<img>元素很像，\<script>元素的 `src` 属性可以是一个完整的 `URL`，而且这个 `URL` 指向的资源可以跟包含它的 `HTML` 页面不在同一个域中，比如这个例子：

```html
<script src="http://www.somewhere.com/afile.js"></script>
```

浏览器在解析这个资源时，会向 src 属性指定的路径发送一个 GET 请求，以取得相应资源，假定 是一个 `JavaScript` 文件。这个初始的请求不受浏览器同源策略限制，但返回并被执行的 `JavaScript` 则受限 制。当然，这个请求仍然受父页面 `HTTP/HTTPS` 协议的限制。

来自外部域的代码会被当成加载它的页面的一部分来加载和解释。这个能力可以让我们通过不同的 域分发 `JavaScript`。不过，引用了放在别人服务器上的 `JavaScript` 文件时要格外小心，因为恶意的程序员 随时可能替换这个文件。在包含外部域的 `JavaScript` 文件时，要确保该域是自己所有的，或者该域是一个可信的来源。\<script>标签的 integrity 属性是防范这种问题的一个武器，但这个属性也不是所有
浏览器都支持。

不管包含的是什么代码，浏览器都会按照\<script>在页面中出现的顺序依次解释它们，前提是它们没有使用 `defer` 和 `async` 属性。第二个\<script>元素的代码必须在第一个\<script>元素的代码解释完毕才能开始解释，第三个则必须等第二个解释完，以此类推。



### 标签位置

过去，所有\<script>元素都被放在页面的\<head>标签内，如下面的例子所示：

```html
<!DOCTYPE html> 
<html> 
 <head> 
 <title>Example HTML Page</title> 
 <script src="example1.js"></script> 
 <script src="example2.js"></script> 
 </head> 
 <body> 
 <!-- 这里是页面内容 --> 
 </body> 
</html> 
```

这种做法的主要目的是把外部的 `CSS` 和`JavaScript` 文件都集中放到一起。不过，把所有`JavaScript` 文件都放在里，也就意味着必须把所有 `JavaScript` 代码都下载、解析和解释完成后，才能开始渲染页面（页面在浏览器解析到的起始标签时开始渲染）。对于需要很多 `JavaScript` 的页面，这会 导致页面渲染的明显延迟，在此期间浏览器窗口完全空白。为解决这个问题，现代 Web 应用程序通常 将所有 `JavaScript` 引用放在元素中的页面内容后面，如下面的例子所示：

```html
<!DOCTYPE html> 
<html> 
 <head> 
 <title>Example HTML Page</title> 
 </head> 
 <body> 
 <!-- 这里是页面内容 --> 
 <script src="example1.js"></script> 
 <script src="example2.js"></script> 
 </body> 
</html> 
```

这样一来，页面会在处理 JavaScript 代码之前完全渲染页面。用户会感觉页面加载更快了，因为浏览器显示空白页面的时间短了。



### 推迟执行脚本

HTML 4.01 为\<script>元素定义了一个叫 `defer` 的属性。这个属性表示脚本在执行的时候不会改变页面的结构。也就是说，脚本会被延迟到整个页面都解析完毕后再运行。因此，在\<script>元素上设置 defer 属性，相当于告诉浏览器立即下载，但延迟执行。



```html
<!DOCTYPE html> 
<html> 
 <head> 
 <title>Example HTML Page</title> 
 <script defer src="example1.js"></script> 
 <script defer src="example2.js"></script> 
 </head> 
 <body> 
 <!-- 这里是页面内容 --> 
 </body> 
</html> 
```

虽然这个例子中的\<script>元素包含在页面的\<head>中，但它们会在浏览器解析到结束的\</html>标签后才会执行。HTML5 规范要求脚本应该按照它们出现的顺序执行，因此第一个推迟的脚本会在第二个推迟的脚本之前执行，而且两者都会在 `DOMContentLoaded` 事件之前执行。不过在实际当中，推迟执行的脚本不一定总会按顺序执行或者在 `DOMContentLoaded`事件之前执行，因此最好只包含一个这样的脚本。

如前所述，`defer` 属性只对外部脚本文件才有效。这是 `HTML5` 中明确规定的，因此支持 HTML5 的浏览器会忽略行内脚本的 defer 属性。IE4~7 展示出的都是旧的行为，IE8 及更高版本则支持 `HTML5` 定义的行为。且并不是所有版本的浏览器都支持这个属性，考虑到这一点，还是把要推迟执行的脚本放在页面底部比较好。



### 异步执行脚本

`HTML5` 为\<script>元素定义了 `async` 属性。从改变脚本处理方式上看，`async` 属性与 `defer` 类似。当然，它们两者也都只适用于外部脚本，都会告诉浏览器立即开始下载。不过，与 `defer` 不同的 是，标记为 `async` 的脚本并不保证能按照它们出现的次序执行，比如：

```html
<!DOCTYPE html> 
<html> 
 <head> 
 <title>Example HTML Page</title> 
 <script async src="example1.js"></script> 
 <script async src="example2.js"></script> 
 </head> 
 <body> 
 <!-- 这里是页面内容 --> 
 </body> 
</html> 
```

在这个例子中，第二个脚本可能先于第一个脚本执行。因此，重点在于它们之间没有依赖关系。给 脚本添加 async 属性的目的是告诉浏览器，不必等脚本下载和执行完后再加载页面，同样也不必等到 该异步脚本下载和执行后再加载其他脚本。正因为如此，异步脚本不应该在加载期间修改 `DOM`。



异步脚本保证会在页面的 `load` 事件前执行，但可能会在 `DOMContentLoaded`之前或之后。使用 async 也会告诉页面你不会使用 `document.write`，不过好的 Web 开发实践根本就不推荐使用这个方法。



### 动态加载脚本

除了\<script>标签，还有其他方式可以加载脚本。因为 `JavaScript` 可以使用 `DOM API`，所以通过向 `DOM` 中动态添加 `script` 元素同样可以加载指定的脚本。只要创建一个 `script` 元素并将其添加到`DOM` 即可。

```javascript
let script = document.createElement('script'); 
script.src = 'gibberish.js'; 
document.head.appendChild(script); 
```



当然，在把 HTMLElement 元素添加到 DOM 且执行到这段代码之前不会发送请求。默认情况下， 以这种方式创建的\<script>元素是以异步方式加载的，相当于添加了 `async` 属性。不过这样做可能会有问题，因为所有浏览器都支持 `createElement()`方法，但不是所有浏览器都支持 `async` 属性。因此，如果要统一动态脚本的加载行为，可以明确将其设置为同步加载：

```javascript
let script = document.createElement('script'); 
script.src = 'gibberish.js'; 
script.async = false; 
document.head.appendChild(script); 
```

以这种方式获取的资源对浏览器预加载器是不可见的。这会严重影响它们在资源获取队列中的优先 级。根据应用程序的工作方式以及怎么使用，这种方式可能会严重影响性能。要想让预加载器知道这些 动态请求文件的存在，可以在文档头部显式声明它们:

```html
<link rel="preload" href="gibberish.js">
```



## 行内代码与外部文件

虽然可以直接在 `HTML` 文件中嵌入 `JavaScript` 代码，但通常认为最佳实践是尽可能将 `JavaScript` 代 码放在外部文件中。不过这个最佳实践并不是明确的强制性规则。推荐使用外部文件的理由如下。

1. 可维护性

   JavaScript 代码如果分散到很多 `HTML` 页面，会导致维护困难。而用一个目录保存 所有 `JavaScript` 文件，则更容易维护，这样开发者就可以独立于使用它们的 HTML 页面来编辑代码。

2. 缓存

   浏览器会根据特定的设置缓存所有外部链接的 `JavaScript`文件，这意味着如果两个页面都 用到同一个文件，则该文件只需下载一次。这最终意味着页面加载更快。



## 文档模式

IE5.5 发明了文档模式的概念，即可以使用 doctype 切换文档模式。最初的文档模式有两种：混杂 模式（quirks mode）和标准模式（standards mode）。前者让 IE 像 IE5 一样（支持一些非标准的特性）， 后者让 IE 具有兼容标准的行为。虽然这两种模式的主要区别只体现在通过 CSS 渲染的内容方面，但对 JavaScript 也有一些关联影响，或称为副作用。



混杂模式在所有浏览器中都以省略文档开头的 doctype 声明作为开关。这种约定并不合理，因为 混杂模式在不同浏览器中的差异非常大，不使用黑科技基本上就没有浏览器一致性可言。

标准模式通过下列几种文档类型声明开启：

```html
<!-- HTML 4.01 Strict --> 
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" 
"http://www.w3.org/TR/html4/strict.dtd"> 
<!-- XHTML 1.0 Strict --> 
<!DOCTYPE html PUBLIC 
"-//W3C//DTD XHTML 1.0 Strict//EN" 
"http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd"> 
<!-- HTML5 --> 
<!DOCTYPE html>
```



## \<noscript>元素

针对早期浏览器不支持 JavaScript 的问题，需要一个页面优雅降级的处理方案。最终， 元素出现，被用于给不支持 JavaScript 的浏览器提供替代内容。虽然如今的浏览器已经 100%支持 JavaScript，但对于禁用 JavaScript 的浏览器来说，这个元素仍然有它的用处。

下面是一个例子：

```html
<body> 
 <noscript> 
 <p>This page requires a JavaScript-enabled browser.</p> 
 </noscript> 
</body> 
```



