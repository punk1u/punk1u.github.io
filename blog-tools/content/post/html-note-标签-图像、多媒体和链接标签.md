---
title: "html学习笔记(5)——图像、多媒体和链接标签" 
date: 2022-06-12T23:32:39+08:00
draft: false
tags: ["web","html"]
categories:
  - "web"
  - "html"
---



本文来自于对 [HTML教程](https://wangdoc.com/html/) 的整理。



# 1. 图像标签

图片是互联网的重要组成部分，让网页变得丰富多彩。本章介绍如何在网页插入图片。



## 1. \<img>

`<img>`标签用于插入图片。它是单独使用的，没有闭合标签。

```html
<img src="foo.jpg">
```



上面代码在网页插入一张图片`foo.jpg`。`src`属性指定图片的网址，上例是相对 `URL`，表示图片与网页在同一个目录。

`<img>`默认是一个行内元素，与前后的文字处在同一行。



```html
<p>Hello<img src="foo.jpg">World</p>
```



上面代码的渲染结果是，文字和图片在同一行显示。

图像默认以原始大小显示。如果图片很大，又与文字处在同一行，那么图片将把当前行的行高撑高，并且图片的底边与文字的底边在同一条水平线上。

`<img>`可以放在`<a>`标签内部，使得图片变成一个可以点击的链接。



```html
<a href="example.html">
  <img src="foo.jpg">
</a>
```



上面代码中，图片可以像链接那样点击，点击后会产生跳转。



1. **alt属性**

   `alt`属性用来设定图片的文字说明。图片不显示时（比如下载失败，或用户关闭图片加载），图片的位置上会显示该文本。

   ```html
   <img src="foo.jpg" alt="示例图片">
   ```

   上面代码中，`alt`是图片的说明。图片下载失败时，浏览器会在图片位置，显示文字“示例图片”。

2. **width属性,height属性**

   图片默认以原始大小插入网页，`width`属性和`height`属性可以指定图片显示时的宽度和高度，单位是像素或百分比。

   ```html
   <img src="foo.jpg" width="400" height="300">
   ```

   上面代码中，`width`属性指定图片显示的宽度为400像素，`height`属性指定显示高度为300像素。

   注意，一旦设置了这两个属性，浏览器会在网页中预先留出这个大小的空间，不管图片有没有加载成功。不过，由于图片的显示大小可以用 `CSS` 设置，所以不建议使用这两个属性。

   一种特殊情况是，`width`属性和`height`属性只设置了一个，另一个没有设置。这时，浏览器会根据图片的原始大小，自动设置对应比例的图片宽度或高度。举例来说，图片大小是 800像素 x 800像素，`width`属性设置成200，那么浏览器会自动将`height`设成200。

3. **srcset,sizes**

   详见下文的《响应式图像》部分。

4. **referrerpolicy**

   `<img>`导致的图片加载的 `HTTP` 请求，默认会带有`Referer`的头信息。`referrerpolicy`属性对这个行为进行设置。

5. **crossorigin**

   有时，图片和网页属于不同的网站，网页加载图片就会导致跨域请求，对方服务器可能要求跨域认证。`crossorigin`属性用来告诉浏览器，是否采用跨域的形式下载图片，默认是不采用。

   简单说，只要打开了这个属性，HTTP 请求的头信息里面，就会加入`origin`字段，给出请求发出的域名，不打开这个属性就不加。

   一旦打开该属性，它可以设为两个值。

   - `anonymous`：跨域请求不带有用户凭证（通常是 Cookie）。
   - `use-credentials`：跨域请求带有用户凭证。

   下面是一个例子。

   ```html
   <img src="foo.jpg" crossorigin="anonymous">
   ```

   `crossorigin`属性如果省略值的部分，则等同于`anonymous`。

   ```html
   <img src="foo.jpg" crossorigin>
   ```

6. **loading**

   浏览器的默认行为是，只要解析到`<img>`标签，就开始加载图片。对于很长的网页，这样做很浪费带宽，因为用户不一定会往下滚动，一直看到网页结束。用户很可能是点开网页，看了一会就关掉了，那些不在视口的图片加载的流量，就都浪费了。

   `loading`属性改变了这个行为，可以指定图片的懒加载，即图片默认不加载，只有即将滚动进入视口，变成用户可见时才会加载，这样就节省了带宽。

   `loading`属性可以取以下三个值。

   - `auto`：浏览器默认行为，等同于不使用`loading`属性。
   - `lazy`：启用懒加载。
   - `eager`：立即加载资源，无论它在页面上的哪个位置。

   ```html
   <img src="image.png" loading="lazy" alt="…" width="200" height="200">
   ```

   由于行内图片的懒加载，可能会导致页面布局重排，所以使用这个属性的时候，最好指定图片的高和宽。



## 2. \<figure>,\<figcaption>

`<figure>`标签可以理解为一个图像区块，将图像和相关信息封装在一起。`<figcaption>`是它的可选子元素，表示图像的文本描述，通常用于放置标题，可以出现多个。



```html
<figure>
  <img src="https://example.com/foo.jpg">
  <figcaption>示例图片</figcaption>
</figure>
```

除了图像，`<figure>`还可以封装引言、代码、诗歌等等。它等于是一个将主体内容与附加信息，封装在一起的语义容器。

```html
<figure>
  <figcaption>JavaScript 代码示例</figcaption>
  <p><code>const foo = 'hello';</code></p>
</figure>
```



## 3. 响应式图像

网页在不同尺寸的设备上，都能产生良好的显示效果，叫做响应式设计（responsive web design）。响应式设计的网页图像，就是“响应式图像”（responsive image）。

响应式图像的解决方案有很多，`JavaScript` 和 `CSS` 都可以实现。这里只介绍语义性最好的 HTML 方法，浏览器原生支持。



### 3.1 问题的由来

我们知道，`<img>`标签用于插入网页图像，所有情况默认插入的都是同一张图像。

```html
<img src="foo.jpg">
```

上面代码在桌面端和手机上，插入的都是图像文件`foo.jpg`。

这种处理方法固然简单，但是有三大弊端。

**（1）体积**

一般来说，桌面端显示的是大尺寸的图像，文件体积较大。手机的屏幕较小，只需要小尺寸的图像，可以节省带宽，加速网页渲染。

**（2）像素密度**

桌面显示器一般是单倍像素密度，而手机的显示屏往往是多倍像素密度，即显示时多个像素合成为一个像素，这种屏幕称为 Retina 屏幕。图像文件很可能在桌面端很清晰，放到手机上会有点模糊，因为图像没有那么高的像素密度，浏览器自动把图像的每个像素复制到周围像素，满足像素密度的要求，导致图像的锐利度有所下降。

**（3）视觉风格**

桌面显示器的面积较大，图像可以容纳更多细节。手机的屏幕较小，许多细节是看不清的，需要突出重点。



### 3.2 srcset属性

为了解决上面这些问题，`HTML` 语言提供了一套完整的解决方案。首先，`<img>`标签引入了`srcset`属性。

`srcset`属性用来指定多张图像，适应不同像素密度的屏幕。它的值是一个逗号分隔的字符串，每个部分都是一张图像的 URL，后面接一个空格，然后是像素密度的描述符。请看下面的例子。

```html
<img srcset="foo-320w.jpg,
             foo-480w.jpg 1.5x,
             foo-640w.jpg 2x"
     src="foo-640w.jpg">
```

上面代码中，`srcset`属性给出了三个图像 URL，适应三种不同的像素密度。

图像 URL 后面的像素密度描述符，格式是像素密度倍数 + 字母`x`。`1x`表示单倍像素密度，可以省略。浏览器根据当前设备的像素密度，选择需要加载的图像。

如果`srcset`属性都不满足条件，那么就加载`src`属性指定的默认图像。



### 3.3 sizes属性

像素密度的适配，只适合显示区域一样大小的图像。如果希望不同尺寸的屏幕，显示不同大小的图像，`srcset`属性就不够用了，必须搭配`sizes`属性。

第一步，`srcset`属性列出所有可用的图像。

```html
<img srcset="foo-160.jpg 160w,
             foo-320.jpg 320w,
             foo-640.jpg 640w,
             foo-1280.jpg 1280w"
     src="foo-1280.jpg">
```

上面代码中，`srcset`属性列出四张可用的图像，每张图像的 URL 后面是一个空格，再加上宽度描述符。

宽度描述符就是图像原始的宽度，加上字符`w`。上例的四种图片的原始宽度分别为160像素、320像素、640像素和1280像素。

第二步，`sizes`属性列出不同设备的图像显示宽度。

`sizes`属性的值是一个逗号分隔的字符串，除了最后一部分，前面每个部分都是一个放在括号里面的媒体查询表达式，后面是一个空格，再加上图像的显示宽度。

```html
<img srcset="foo-160.jpg 160w,
             foo-320.jpg 320w,
             foo-640.jpg 640w,
             foo-1280.jpg 1280w"
     sizes="(max-width: 440px) 100vw,
            (max-width: 900px) 33vw,
            254px"
     src="foo-1280.jpg">
```

上面代码中，`sizes`属性给出了三种屏幕条件，以及对应的图像显示宽度。宽度不超过440像素的设备，图像显示宽度为100%；宽度441像素到900像素的设备，图像显示宽度为33%；宽度900像素以上的设备，图像显示宽度为`254px`。

第三步，浏览器根据当前设备的宽度，从`sizes`属性获得图像的显示宽度，然后从`srcset`属性找出最接近该宽度的图像，进行加载。

假定当前设备的屏幕宽度是`480px`，浏览器从`sizes`属性查询得到，图片的显示宽度是`33vw`（即33%），等于`160px`。`srcset`属性里面，正好有宽度等于`160px`的图片，于是加载`foo-160.jpg`。

如果省略`sizes`属性，那么浏览器将根据实际的图像显示宽度，从`srcset`属性选择最接近的图片。一旦使用`sizes`属性，就必须与`srcset`属性搭配使用，单独使用`sizes`属性是无效的。



## 4. \<picture>

### 4.1 响应式用法

`<img>`标签的`srcset`属性和`sizes`属性分别解决了像素密度和屏幕大小的适配，但如果要同时适配不同像素密度、不同大小的屏幕，就要用到`<picture>`标签。

`<picture>`是一个容器标签，内部使用`<source>`和`<img>`，指定不同情况下加载的图像。

```html
<picture>
  <source media="(max-width: 500px)" srcset="cat-vertical.jpg">
  <source media="(min-width: 501px)" srcset="cat-horizontal.jpg">
  <img src="cat.jpg" alt="cat">
</picture>
```

上面代码中，`<picture>`标签内部有两个`<source>`标签和一个`<img>`标签。

`<picture>`内部的`<source>`标签，主要使用`media`属性和`srcset`属性。`media`属性给出媒体查询表达式，`srcset`属性就是`<img>`标签的`srcset`属性，给出加载的图像文件。`sizes`属性其实这里也可以用，但由于有了`media`属性，就没有必要了。

浏览器按照`<source>`标签出现的顺序，依次判断当前设备是否满足`media`属性的媒体查询表达式，如果满足就加载`srcset`属性指定的图片文件，并且不再执行后面的`<source>`标签和`<img>`标签。

`<img>`标签是默认情况下加载的图像，用来满足上面所有`<source>`都不匹配的情况，或者不支持`<picture>`的老式浏览器。

上面例子中，设备宽度如果不超过`500px`，就加载竖屏的图像，否则加载横屏的图像。

下面给出一个例子，同时考虑屏幕尺寸和像素密度的适配。



```html
<picture>
  <source srcset="homepage-person@desktop.png,
                  homepage-person@desktop-2x.png 2x"
          media="(min-width: 990px)">
  <source srcset="homepage-person@tablet.png,
                  homepage-person@tablet-2x.png 2x"
          media="(min-width: 750px)">
  <img srcset="homepage-person@mobile.png,
               homepage-person@mobile-2x.png 2x"
       alt="Shopify Merchant, Corrine Anestopoulos">
</picture>
```

上面代码中，`<source>`标签的`media`属性给出屏幕尺寸的适配条件，每个条件都用`srcset`属性，再给出两种像素密度的图像 URL。



### 4.2 图像格式的选择

除了响应式图像，`<picture>`标签还可以用来选择不同格式的图像。比如，如果当前浏览器支持 `Webp` 格式，就加载这种格式的图像，否则加载 `PNG` 图像。

```html
<picture>
  <source type="image/svg+xml" srcset="logo.xml">
  <source type="image/webp" srcset="logo.webp"> 
  <img src="logo.png" alt="ACME Corp">
</picture>
```

上面代码中，`<source>`标签的`type`属性给出图像的 MIME 类型，`srcset`是对应的图像 URL。

浏览器按照`<source>`标签出现的顺序，依次检查是否支持`type`属性指定的图像格式，如果支持就加载图像，并且不再检查后面的`<source>`标签了。上面例子中，图像加载优先顺序依次为 `svg` 格式、`webp` 格式和 `png` 格式。



# 2. 链接标签

链接（`hyperlink`）是互联网的核心。它允许用户在页面上，从一个网址跳转到另一个网址，从而把所有资源联系在一起。

`URL` 是链接指向的地址。链接不仅可以指向另一个网页，也可以指向文本、图像、文件等资源。可以这样说，所有互联网上的资源，都可以通过链接访问。



## 1. \<a>

链接通过`<a>`标签表示，用户点击后，浏览器会跳转到指定的网址。下面就是一个典型的链接。

```html
<a href="https://wikipedia.org/">维基百科</a>
```

上面代码就定义了一个超级链接。浏览器显示“维基百科”，文字下面默认会有下划线，表示这是一个链接。用户点击后，浏览器跳转到`href`属性指定的网址。

 `<a>`标签内部不仅可以放置文字，也可以放置其他元素，比如段落、图像、多媒体等等。

```html
<a href="https://www.example.com/">
  <img src="https://www.example.com/foo.jpg">
</a>
```

上面代码中，`<a>`标签内部就是一个图像。用户点击图像，就会跳转到指定网址。

`<a>`标签有如下属性。

1. **href**

   `href`属性给出链接指向的网址。它的值应该是一个 URL 或者锚点。

   上文已经给出了完整 URL 的例子，下面是锚点的例子。

   ```html
   <a href="#demo">示例</a>
   ```

   上面代码中，`href`属性的值是`#`加上锚点名称。点击后，浏览器会自动滚动，停在当前页面里面`demo`锚点所在的位置。

2. **hreflang**

   `hreflang`属性给出链接指向的网址所使用的语言，纯粹是提示性的，没有实际功能。

   ```html
   <a
     href="https://www.example.com"
     hreflang="en"
   >示例网址</a>
   ```

   上面代码表明，`href`属性指向的网址的语言是英语。

   该属性的值跟通用属性`lang`一样，语言代码可以参考《属性》一章的`lang`属性的介绍。

3. **title**

   `title`属性给出链接的说明信息。鼠标悬停在链接上方时，浏览器会将这个属性的值，以提示块的形式显示出来。

   ```html
   <a
     href="https://www.example.com/"
     title="hello"
   >示例</a>。
   ```

   上面代码中，用户鼠标停留在链接上面，会出现文字提示`hello`。

4. **target**

   `target`属性指定如何展示打开的链接。它可以是在指定的窗口打开，也可以在`<iframe>`里面打开。

   ```html
   <p><a href="http://foo.com" target="test">foo</a></p>
   <p><a href="http://bar.com" target="test">bar</a></p>
   ```

   上面代码中，两个链接都在名叫`test`的窗口打开。首先点击链接`foo`，浏览器发现没有叫做`test`的窗口，就新建一个窗口，起名为`test`，在该窗口打开`foo.com`。然后，用户又点击链接`bar`，由于已经存在`test`窗口，浏览器就在该窗口打开`bar.com`，取代里面已经打开的`foo.com`。

   `target`属性的值也可以是以下四个关键字之一。

   - `_self`：当前窗口打开，这是默认值。
   - `_blank`：新窗口打开。
   - `_parent`：上层窗口打开，这通常用于从父窗口打开的子窗口，或者`<iframe>`里面的链接。如果当前窗口没有上层窗口，这个值等同于`_self`。
   - `_top`：顶层窗口打开。如果当前窗口就是顶层窗口，这个值等同于`_self`。

   ```html
   <a
     href="https://www.example.com"
     target="_blank"
   >示例链接</a>
   ```

   上面代码点击后，浏览器会新建一个窗口，在该窗口打开链接，并且新窗口没有名字。

   注意，使用`target`属性的时候，最好跟`rel="noreferrer"`一起使用，这样可以避免安全风险。

5. **rel**

   `rel`属性说明链接与当前页面的关系。

   ```html
   <a href="help.html" rel="help">帮助</a>
   ```

   上面代码的`rel`属性，说明链接是当前页面的帮助文档。

   下面是一些常见的`rel`属性的值。

   - `alternate`：当前文档的另一种形式，比如翻译。
   - `author`：作者链接。
   - `bookmark`：用作书签的永久地址。
   - `external`：当前文档的外部参考文档。
   - `help`：帮助链接。
   - `license`：许可证链接。
   - `next`：系列文档的下一篇。
   - `nofollow`：告诉搜索引擎忽略该链接，主要用于用户提交的内容，防止有人企图通过添加链接，提高该链接的搜索排名。
   - `noreferrer`：告诉浏览器打开链接时，不要将当前网址作为 HTTP 头信息的`Referer`字段发送出去，这样可以隐藏点击的来源。
   - `noopener`：告诉浏览器打开链接时，不让链接窗口通过 JavaScript 的`window.opener`属性引用原始窗口，这样就提高了安全性。
   - `prev`：系列文档的上一篇。
   - `search`：文档的搜索链接。
   - `tag`：文档的标签链接。

6. **referrerpolicy**

   `referrerpolicy`属性用于精确设定点击链接时，浏览器发送 HTTP 头信息的`Referer`字段的行为。

   该属性可以取下面八个值：`no-referrer`、`no-referrer-when-downgrade`、`origin`、`origin-when-cross-origin`、`unsafe-url`、`same-origin`、`strict-origin`、`strict-origin-when-cross-origin`。

   其中，`no-referrer`表示不发送`Referer`字段，`same-origin`表示同源时才发送`Referer`字段，`origin`表示只发送源信息（协议+域名+端口）。其他几项的解释，请查阅 HTTP 文档。

7. **ping**

   `ping`属性指定一个网址，用户点击的时候，会向该网址发出一个 POST 请求，通常用于跟踪用户的行为。

   ```html
   <a href="http://localhost:3000/other" ping="http://localhost:3000/log">
     Go to Other Page
   </a>
   ```

   上面示例中，用户点击链接时，除了发生跳转，还会向`http://localhost:3000/log`发送一个 POST 请求。服务端收到这个请求以后，就会知道用户点击了这个链接。

   这个请求的 HTTP 标头，包含了`ping-from`属性（点击行为发生的页面）和`ping-to`属性（`href`属性所指向的页面）。

   ```html
   headers: {
     'ping-from': 'http://localhost:3000/',
     'ping-to': 'http://localhost:3000/other'
     'content-type': 'text/ping'
     // ...other headers
   },
   ```

   注意，`ping`属性只对链接有效，对其他的交互行为无效，比如按钮点击或表单提交。另外，`Firefox` 浏览器不支持该属性。并且，也无法让它发送任何的自定义数据。

8. **type**

   `type`属性给出链接 URL 的 MIME 类型，比如到底是网页，还是图像或文件。它也是纯粹提示性的属性，没有实际功能。

   ```html
   <a
     href="smile.jpg"
     type="image/jpeg"
   >示例图片</a>
   ```

   上面代码中，`type`属性提示这是一张图片。

9. **download**

   `download`属性表明当前链接用于下载，而不是跳转到另一个 URL。

   ```html
   <a href="demo.txt" download>下载</a>
   ```

   上面代码点击后，会出现下载对话框。

   注意，`download`属性只在链接与网址同源时，才会生效。也就是说，链接应该与网址属于同一个网站。

   如果`download`属性设置了值，那么这个值就是下载的文件名。

   ```html
   <a
     href="foo.exe"
     download="bar.exe"
   >点击下载</a>
   ```

   上面代码中，下载文件的原始文件名是`foo.exe`。点击后，下载对话框提示的文件名是`bar.exe`。

   注意，如果链接点击后，服务器的 HTTP 回应的头信息设置了`Content-Disposition`字段，并且该字段的值与`download`属性不一致，那么该字段优先，下载时将显示其设置的文件名。

   `download`属性还有一个用途，就是有些地址不是真实网址，而是数据网址，比如`data:`开头的网址。这时，`download`属性可以为虚拟网址指定下载的文件名。

   ```html
   <a href="data:,Hello%2C%20World!">点击</a>
   ```

   上面链接点击后，会打开一个虚拟网页，上面显示`Hello World!`。

   ```html
   <a
     href="data:,Hello%2C%20World!"
     download="hello.txt"
   >点击</a>
   ```

   上面链接点击后，下载的`hello.txt`文件内容就是“Hello, World!”。



## 2. 邮件链接

链接也可以指向一个邮件地址，使用`mailto`协议。用户点击后，浏览器会打开本机默认的邮件程序，让用户向指定的地址发送邮件。

```html
<a href="mailto:contact@example.com">联系我们</a>
```

上面代码中，链接就指向邮件地址。点击后，浏览器会打开一个邮件地址，让你可以向`contact@example.com`发送邮件。

除了邮箱，邮件协议还允许指定其他几个邮件要素。

- `subject`：主题
- `cc`：抄送
- `bcc`：密送
- `body`：邮件内容

使用方法是将这些邮件要素，以查询字符串的方式，附加在邮箱地址后面。

```html
<a
  href="mailto:foo@bar.com?cc=test@test.com&subject=The%20subject&body=The%20body"
>发送邮件</a>
```

上面代码中，邮件链接里面不仅包含了邮箱地址，还包含了`cc`、`subject`、`body`等邮件要素。这些要素的值需要经过 URL 转义，比如空格转成`%20`。

不指定邮箱也是允许的，就像下面这样。这时用户自己在邮件程序里面，填写想要发送的邮箱，通常用于邮件分享网页。

```html
<a href="mailto:">告诉朋友</a>
```



## 3. 电话协议

如果是手机浏览的页面，还可以使用`tel`协议，创建电话链接。用户点击该链接，会唤起电话，可以进行拨号。

```html
<a href="tel:13312345678">13312345678</a>
```

上面代码在手机中，点击链接会唤起拨号界面，可以直接拨打指定号码。



## 4. \<link>

### 4.1 基本用法

`<link>`标签主要用于将当前网页与相关的外部资源联系起来，通常放在`<head>`元素里面。最常见的用途就是加载 `CSS` 样式表。

```html
<link rel="stylesheet" type="text/css" href="theme.css">
```

上面代码为网页加载样式表`theme.css`。

除了默认样式表，网页还可以加载替代样式表，即默认不生效、需要用户手动切换的样式表。

```html
<link href="default.css" rel="stylesheet" title="Default Style">
<link href="fancy.css" rel="alternate stylesheet" title="Fancy">
<link href="basic.css" rel="alternate stylesheet" title="Basic">
```

上面代码中，`default.css`是默认样式表，默认就会生效。`fancy.css`和`basic.css`是替换样式表（`rel="alternate stylesheet"`），默认不生效。`title`属性在这里是必需的，用来在浏览器菜单里面列出这些样式表的名字，供用户选择，以替代默认样式表。

`<link>`还可以加载网站的 `favicon` 图标文件。

```html
<link rel="icon" href="/favicon.ico" type="image/x-icon">
```

手机访问时，网站通常需要提供不同分辨率的图标文件。



```html
<link rel="apple-touch-icon-precomposed" sizes="114x114" href="favicon114.png">
<link rel="apple-touch-icon-precomposed" sizes="72x72" href="favicon72.png">
```

上面代码指定 iPhone 设备需要的114像素和72像素的图标。

`<link>`也用于提供文档的相关链接，比如下面是给出文档的 `RSS Feed` 地址。



```html
<link rel="alternate" type="application/atom+xml" href="/blog/news/atom">
```



### 4.2 rel属性

`rel`属性表示外部资源与当前文档之间的关系，是`<link>`标签的必需属性。它可以但不限于取以下值。

- `alternate`：文档的另一种表现形式的链接，比如打印版。
- `author`：文档作者的链接。
- `dns-prefetch`：要求浏览器提前执行指定网址的 DNS 查询。
- `help`：帮助文档的链接。
- `icon`：加载文档的图标文件。
- `license`：许可证链接。
- `next`：系列文档下一篇的链接。
- `pingback`：接收当前文档 pingback 请求的网址。
- `preconnect`：要求浏览器提前与给定服务器，建立 HTTP 连接。
- `prefetch`：要求浏览器提前下载并缓存指定资源，供下一个页面使用。它的优先级较低，浏览器可以不下载。
- `preload`：要求浏览器提前下载并缓存指定资源，当前页面稍后就会用到。它的优先级较高，浏览器必须立即下载。
- `prerender`：要求浏览器提前渲染指定链接。这样的话，用户稍后打开该链接，就会立刻显示，感觉非常快。
- `prev`：表示当前文档是系列文档的一篇，这里给出上一篇文档的链接。
- `search`：提供当前网页的搜索链接。
- `stylesheet`：加载一张样式表。

下面是一些示例。

```html
<!-- 作者信息 -->
<link rel="author" href="humans.txt">

<!-- 版权信息 -->
<link rel="license" href="copyright.html">

<!-- 另一个语言的版本 -->
<link rel="alternate" href="https://es.example.com/" hreflang="es">

<!-- 联系方式 -->
<link rel="me" href="https://google.com/profiles/someone" type="text/html">
<link rel="me" href="mailto:name@example.com">
<link rel="me" href="sms:+15035550125">

<!-- 历史资料 -->
<link rel="archives" href="http://example.com/archives/">

<!-- 目录 -->
<link rel="index" href="http://example.com/article/">

<!-- 导航 -->
<link rel="first" href="http://example.com/article/">
<link rel="last" href="http://example.com/article/?page=42">
<link rel="prev" href="http://example.com/article/?page=1">
<link rel="next" href="http://example.com/article/?page=3">
```



### 4.3 资源的预加载

某些情况下，你需要浏览器预加载某些资源，也就是先把资源缓存下来，等到使用的时候，就不用再从网上下载了，立即就能使用。预处理指令可以做到这一点。

预加载主要有下面五种类型。

1. \<link rel="preload">

   `<link rel="preload">`告诉浏览器尽快下载并缓存资源（如脚本或样式表），该指令优先级较高，浏览器肯定会执行。当加载页面几秒钟后需要该资源时，它会很有用。下载后，浏览器不会对资源执行任何操作，脚本未执行，样式表未应用。它只是缓存，当其他东西需要它时，它立即可用。

   ```html
   <link rel="preload" href="image.png" as="image">
   ```

   `rel="preload"`除了优先级较高，还有两个优点：一是允许指定预加载资源的类型，二是允许`onload`事件的回调函数。下面是`rel="preload"`配合`as`属性，告诉浏览器预处理资源的类型，以便正确处理。

   ```html
   <link rel="preload" href="style.css" as="style">
   <link rel="preload" href="main.js" as="script">
   ```

   上面代码要求浏览器提前下载并缓存`style.css`和`main.js`。

   `as`属性指定加载资源的类型，它的值一般有下面几种。

   - "script"
   - "style"
   - "image"
   - "media"
   - "document"

   如果不指定`as`属性，或者它的值是浏览器不认识的，那么浏览器会以较低的优先级下载这个资源。

   有时还需要`type`属性，进一步明确 MIME 类型。

   ```html
   <link rel="preload" href="sintel-short.mp4" as="video" type="video/mp4">
   ```

   上面代码要求浏览器提前下载视频文件，并且说明这是 MP4 编码。

   下面是预下载字体文件的例子。

   ```html
   <link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin>
   ```

   注意，所有预下载的资源，只是下载到浏览器的缓存，并没有执行。如果希望资源预下载后立刻执行，可以参考下面的写法。

   ```html
   <link rel="preload" as="style" href="async_style.css" onload="this.rel='stylesheet'">
   ```

   上面代码中，`onload`指定的回调函数会在脚本下载完成后执行，立即插入页面。

2. \<link rel="prefetch">

   `<link rel="prefetch">`的使用场合是，如果后续的页面需要某个资源，并且希望预加载该资源，以便加速页面渲染。该指令不是强制性的，优先级较低，浏览器不一定会执行。这意味着，浏览器可以不下载该资源，比如连接速度很慢时。

   ```html
   <link rel="prefetch" href="https://www.example.com/">
   ```

3. \<link rel="preconnect">

   `<link rel="preconnect">`要求浏览器提前与某个域名建立 TCP 连接。当你知道，很快就会请求该域名时，这会很有帮助。

   ```html
   <link rel="preconnect" href="https://www.example.com/">
   ```

4. \<link rel="dns-prefetch">

   `<link rel="dns-prefetch">`要求浏览器提前执行某个域名的 DNS 解析。

   ```html
   <link rel="dns-prefetch" href="//example.com/">
   ```

5. \<link rel="prerender">

   `<link rel="prerender">`要求浏览器加载某个网页，并且提前渲染它。用户点击指向该网页的链接时，就会立即呈现该页面。如果确定用户下一步会访问该页面，这会很有帮助。

   ```html
   <link rel="prerender" href="http://example.com/">
   ```



### 4.4 media属性

`media`属性给出外部资源生效的媒介条件。

```html
<link href="print.css" rel="stylesheet" media="print">
<link href="mobile.css" rel="stylesheet" media="screen and (max-width: 600px)">
```

上面代码中，打印时加载`print.css`，移动设备访问时（设备宽度小于600像素）加载`mobile.css`。

下面是使用`media`属性实现条件加载的例子。

```html
<link rel="preload" as="image" href="map.png" media="(max-width: 600px)">
<link rel="preload" as="script" href="map.js" media="(min-width: 601px)">
```

上面代码中，如果屏幕宽度在600像素以下，则只加载第一个资源，否则就加载第二个资源。



### 4.5 其他属性

`<link>`标签的其他属性如下。

- `crossorigin`：加载外部资源的跨域设置。
- `href`：外部资源的网址。
- `referrerpolicy`：加载时`Referer`头信息字段的处理方法。
- `as`：`rel="preload"`或`rel="prefetch"`时，设置外部资源的类型。
- `type`：外部资源的 MIME 类型，目前仅用于`rel="preload"`或`rel="prefetch"`的情况。
- `title`：加载样式表时，用来标识样式表的名称。
- `sizes`：用来声明图标文件的尺寸，比如加载苹果手机的图标文件。



## 5. \<script>

`<script>`用于加载脚本代码，目前主要是加载 `JavaScript` 代码。

```html
<script>
console.log('hello world');
</script>
```

上面代码嵌入网页，会立即执行。

`<script>`也可以加载外部脚本，`src`属性给出外部脚本的地址。

```html
<script src="javascript.js"></script>
```

上面代码会加载`javascript.js`脚本文件，并执行。

`type`属性给出脚本的类型，默认是 `JavaScript` 代码，所以可省略。完整的写法其实是下面这样。

```html
<script type="text/javascript" src="javascript.js"></script>
```

`type`属性也可以设成`module`，表示这是一个 `ES6` 模块，不是传统脚本。

```html
<script type="module" src="main.js"></script>
```

对于那些不支持 `ES6` 模块的浏览器，可以设置`nomodule`属性。支持 `ES6` 模块的浏览器，会不加载指定的脚本。这个属性通常与`type="module"`配合使用，作为老式浏览器的回退方案。



```html
<script type="module" src="main.js"></script>
<script nomodule src="fallback.js"></script>
```

`<script>`还有下面一些其他属性，大部分跟 `JavaScript` 语言有关，可以参考相关的 `JavaScript` 教程。

- `async`：该属性指定 JavaScript 代码为异步执行，不是造成阻塞效果，JavaScript 代码默认是同步执行。
- `defer`：该属性指定 JavaScript 代码不是立即执行，而是页面解析完成后执行。
- `crossorigin`：如果采用这个属性，就会采用跨域的方式加载外部脚本，即 HTTP 请求的头信息会加上`origin`字段。
- `integrity`：给出外部脚本的哈希值，防止脚本被篡改。只有哈希值相符的外部脚本，才会执行。
- `nonce`：一个密码随机数，由服务器在 HTTP 头信息里面给出，每次加载脚本都不一样。它相当于给出了内嵌脚本的白名单，只有在白名单内的脚本才能执行。
- `referrerpolicy`：HTTP 请求的`Referer`字段的处理方法。



## 6. \<noscript>

`<noscript>`标签用于浏览器不支持或关闭 `JavaScript` 时，所要显示的内容。用户关闭 `JavaScript` 可能是为了节省带宽，以延长手机电池寿命，或者为了防止追踪，保护隐私。

```html
<noscript>
  您的浏览器不能执行 JavaScript 语言，页面无法正常显示。
</noscript>
```

上面这段代码，只有浏览器不能执行 `JavaScript` 代码时才会显示，否则就不会显示。



# 3. 多媒体标签

除了图像，网页还可以放置视频和音频。

## 1. \<video>

`<video>`标签是一个块级元素，用于放置视频。如果浏览器支持加载的视频格式，就会显示一个播放器，否则显示`<video>`内部的子元素。

```html
<video src="example.mp4" controls>
  <p>你的浏览器不支持 HTML5 视频，请下载<a href="example.mp4">视频文件</a>。</p>
</video>
```

上面代码中，如果浏览器不支持该种格式的视频，就会显示`<video>`内部的文字提示。

`<video>`有以下属性。

- `src`：视频文件的网址。
- `controls`：播放器是否显示控制栏。该属性是布尔属性，不用赋值，只要写上属性名，就表示打开。如果不想使用浏览器默认的播放器，而想使用自定义播放器，就不要使用该属性。
- `width`：视频播放器的宽度，单位像素。
- `height`：视频播放器的高度，单位像素。
- `autoplay`：视频是否自动播放，该属性为布尔属性。
- `loop`：视频是否循环播放，该属性为布尔属性。
- `muted`：是否默认静音，该属性为布尔属性。
- `poster`：视频播放器的封面图片的 URL。
- `preload`：视频播放之前，是否缓冲视频文件。这个属性仅适合没有设置`autoplay`的情况。它有三个值，分别是`none`（不缓冲）、`metadata`（仅仅缓冲视频文件的元数据）、`auto`（可以缓冲整个文件）。
- `playsinline`：iPhone 的 Safari 浏览器播放视频时，会自动全屏，该属性可以禁止这种行为。该属性为布尔属性。
- `crossorigin`：是否采用跨域的方式加载视频。它可以取两个值，分别是`anonymous`（跨域请求时，不发送用户凭证，主要是 Cookie），`use-credentials`（跨域时发送用户凭证）。
- `currentTime`：指定当前播放位置（双精度浮点数，单位为秒）。如果尚未开始播放，则会从这个属性指定的位置开始播放。
- `duration`：该属性只读，指示时间轴上的持续播放时间（总长度），值为双精度浮点数（单位为秒）。如果是流媒体，没有已知的结束时间，属性值为`+Infinity`。

下面是一个例子。

```html
<video width="400" height="400"
       autoplay loop muted
       poster="poster.png">
</video>
```

上面代码中，视频播放器的大小是 400 x 400，会自动播放和循环播放，并且静音，还带有封面图。这是网站首页背景视频的常见写法。

HTML 标准没有规定浏览器需要支持哪些视频格式，完全由浏览器厂商自己决定。为了避免浏览器不支持视频格式，可以使用`<source>`标签，放置同一个视频的多种格式。



```html
<video controls>
  <source src="example.mp4" type="video/mp4">
  <source src="example.webm" type="video/webm">
  <p>你的浏览器不支持 HTML5 视频，请下载<a href="example.mp4">视频文件</a>。</p>
</video>
```

上面代码中，`<source>`标签的`type`属性的值是视频文件的 MIME 类型，上例指定了两种格式的视频文件：`MP4` 和 `WebM`。如果浏览器支持 `MP4`，就加载 `MP4` 格式的视频，不再往下执行了。如果不支持 `MP4`，就检查是否支持 `WebM`，如果还是不支持，则显示提示。



## 2. \<audio>

`<audio>`标签是一个块级元素，用于放置音频，用法与`<video>`标签基本一致。

```html
<audio controls>
  <source src="foo.mp3" type="audio/mp3">
  <source src="foo.ogg" type="audio/ogg">
  <p>你的浏览器不支持 HTML5 音频，请直接下载<a href="foo.mp3">音频文件</a>。</p>
</audio>
```

上面代码中，`<audio>`标签内部使用`<source>`标签，指定了两种音频格式：优先使用 MP3 格式，如果浏览器不支持则使用 Ogg 格式。如果浏览器不能播放音频，则提供下载链接。

`<audio>`标签的属性与`<video>`标签类似，参见上一节。

- `autoplay`：是否自动播放，布尔属性。
- `controls`：是否显示播放工具栏，布尔属性。如果不设置，浏览器不显示播放界面，通常用于背景音乐。
- `crossorigin`：是否使用跨域方式请求。
- `loop`：是否循环播放，布尔属性。
- `muted`：是否静音，布尔属性。
- `preload`：音频文件的缓冲设置。
- `src`：音频文件网址。



## 3. \<track>

`<track>`标签用于指定视频的字幕，格式是 `WebVTT` （`.vtt`文件），放置在`<video>`标签内部。它是一个单独使用的标签，没有结束标签。

```html
<video controls src="sample.mp4">
   <track label="英文" kind="subtitles" src="subtitles_en.vtt" srclang="en">
   <track label="中文" kind="subtitles" src="subtitles_cn.vtt" srclang="cn" default>
</video>
```

上面代码指定视频文件的英文字幕和中文字幕。

`<track>`标签有以下属性。

- `label`：播放器显示的字幕名称，供用户选择。
- `kind`：字幕的类型，默认是`subtitles`，表示将原始声音成翻译外国文字，比如英文视频提供中文字幕。另一个常见的值是`captions`，表示原始声音的文字描述，通常是视频原始使用的语言，比如英文视频提供英文字幕。
- `src`：vtt 字幕文件的网址。
- `srclang`：字幕的语言，必须是有效的语言代码。
- `default`：是否默认打开，布尔属性。



## 4. \<source>

`<source>`标签用于`<picture>`、`<video>`、`<audio>`的内部，用于指定一项外部资源。单标签是单独使用的，没有结束标签。

它有如下属性，具体示例请参见相应的容器标签。

- `type`：指定外部资源的 MIME 类型。
- `src`：指定源文件，用于`<video>`和`<audio>`。
- `srcset`：指定不同条件下加载的图像文件，用于`<picture>`。
- `media`：指定媒体查询表达式，用于`<picture>`。
- `sizes`：指定不同设备的显示大小，用于`<picture>`，必须跟`srcset`搭配使用。



## 5. \<embed>

`<embed>`标签用于嵌入外部内容，这个外部内容通常由浏览器插件负责控制。由于浏览器的默认插件都不一致，很可能不是所有浏览器的用户都能访问这部分内容，建议谨慎使用。

下面是嵌入视频播放器的例子。

```html
<embed type="video/webm"
       src="/media/examples/flower.mp4"
       width="250"
       height="200">
```

上面代码嵌入的视频，将由浏览器插件负责控制。如果浏览器没有安装 MP4 插件，视频就无法播放。

`<embed>`标签具有如下的通用属性。

- `height`：显示高度，单位为像素，不允许百分比。
- `width`：显示宽度，单位为像素，不允许百分比。
- `src`：嵌入的资源的 URL。
- `type`：嵌入资源的 MIME 类型。

浏览器通过`type`属性得到嵌入资源的 MIME 类型，一旦该种类型已经被某个插件注册了，就会启动该插件，负责处理嵌入的资源。

下面是 `QuickTime` 插件播放 `MOV` 视频文件的例子。

```html
<embed type="video/quicktime" src="movie.mov" width="640" height="480">
```

下面是启动 Flash 插件的例子。

```html
<embed src="whoosh.swf" quality="medium"
       bgcolor="#ffffff" width="550" height="400"
       name="whoosh" align="middle" allowScriptAccess="sameDomain"
       allowFullScreen="false" type="application/x-shockwave-flash"
       pluginspage="http://www.macromedia.com/go/getflashplayer">
```

上面代码中，如果浏览器没有安装 Flash 插件，就会提示去`pluginspage`属性指定的网址下载。



## 6. \<object>,\<param>

`<object>`标签作用跟`<embed>`相似，也是插入外部资源，由浏览器插件处理。它可以视为`<embed>`的替代品，有标准化行为，只限于插入少数几种通用资源，没有历史遗留问题，因此更推荐使用。

下面是插入 `PDF` 文件的例子。

```html
<object type="application/pdf"
    data="/media/examples/In-CC0.pdf"
    width="250"
    height="200">
</object>
```

上面代码中，如果浏览器安装了 `PDF` 插件，就会在网页显示 `PDF` 浏览窗口。

`<object>`具有如下的通用属性。

- `data`：嵌入的资源的 URL。
- `form`：当前网页中相关联表单的`id`属性（如果有的话）。
- `height`：资源的显示高度，单位为像素，不能使用百分比。
- `width`：资源的显示宽度，单位为像素，不能使用百分比。
- `type`：资源的 MIME 类型。
- `typemustmatch`：布尔属性，表示`data`属性与`type`属性是否必须匹配。

下面是插入 Flash 影片的例子。

```html
<object data="movie.swf"
  type="application/x-shockwave-flash"></object>
```

`<object>`标签是一个容器元素，内部可以使用`<param>`标签，给出插件所需要的运行参数。

```html
<object data="movie.swf" type="application/x-shockwave-flash">
  <param name="foo" value="bar">
</object>
```





