---
title: "html学习笔记(7)——其他标签" 
date: 2022-06-15T23:32:39+08:00
draft: false
tags: ["web","html"]
categories:
  - "web"
  - "html"
---



本文来自于对 [HTML教程](https://wangdoc.com/html/) 的整理。



# 1. iframe

`<iframe>`标签用于在网页里面嵌入其他网页。

## 1. 基本用法

`<iframe>`标签生成一个指定区域，在该区域中嵌入其他网页。它是一个容器元素，如果浏览器不支持`<iframe>`，就会显示内部的子元素。

```html
<iframe src="https://www.example.com"
        width="100%" height="500" frameborder="0"
        allowfullscreen sandbox>
  <p><a href="https://www.example.com">点击打开嵌入页面</a></p>
</iframe>
```

上面的代码在当前网页嵌入`https://www.example.com`，显示区域的宽度是`100%`，高度是`500`像素。如果当前浏览器不支持`<iframe>`，则会显示一个链接，让用户点击。

浏览器普遍支持`<iframe>`，所以内部的子元素可以不写。

`<iframe>`的属性如下。

- `allowfullscreen`：允许嵌入的网页全屏显示，需要全屏 API 的支持，请参考相关的 JavaScript 教程。
- `frameborder`：是否绘制边框，`0`为不绘制，`1`为绘制（默认值）。建议尽量少用这个属性，而是在 CSS 里面设置样式。
- `src`：嵌入的网页的 URL。
- `width`：显示区域的宽度。
- `height`：显示区域的高度。
- `sandbox`：设置嵌入的网页的权限，详见下文。
- `importance`：浏览器下载嵌入的网页的优先级，可以设置三个值。`high`表示高优先级，`low`表示低优先级，`auto`表示由浏览器自行决定。
- `name`：内嵌窗口的名称，可以用于`<a>`、`<form>`、`<base>`的`target`属性。
- `referrerpolicy`：请求嵌入网页时，HTTP 请求的`Referer`字段的设置。参见`<a>`标签的介绍。



## 2. sandbox 属性

嵌入的网页默认具有正常权限，比如执行脚本、提交表单、弹出窗口等。如果嵌入的网页是其他网站的页面，你不了解对方会执行什么操作，因此就存在安全风险。为了限制`<iframe>`的风险，HTML 提供了`sandbox`属性，允许设置嵌入的网页的权限，等同于提供了一个隔离层，即“沙箱”。

`sandbox`可以当作布尔属性使用，表示打开所有限制。

```html
<iframe src="https://www.example.com" sandbox>
</iframe>
```

`sandbox`属性可以设置具体的值，表示逐项打开限制。未设置某一项，就表示不具有该权限。

- `allow-forms`：允许提交表单。
- `allow-modals`：允许提示框，即允许执行`window.alert()`等会产生弹出提示框的 JavaScript 方法。
- `allow-popups`：允许嵌入的网页使用`window.open()`方法弹出窗口。
- `allow-popups-to-escape-sandbox`：允许弹出窗口不受沙箱的限制。
- `allow-orientation-lock`：允许嵌入的网页用脚本锁定屏幕的方向，即横屏或竖屏。
- `allow-pointer-lock`：允许嵌入的网页使用 Pointer Lock API，锁定鼠标的移动。
- `allow-presentation`：允许嵌入的网页使用 Presentation API。
- `allow-same-origin`：不打开该项限制，将使得所有加载的网页都视为跨域。
- `allow-scripts`：允许嵌入的网页运行脚本（但不创建弹出窗口）。
- `allow-storage-access-by-user-activation`：`sandbox`属性同时设置了这个值和`allow-same-origin`的情况下，允许`<iframe>`嵌入的第三方网页通过用户发起`document.requestStorageAccess()`请求，经由 Storage Access API 访问父窗口的 Cookie。
- `allow-top-navigation`：允许嵌入的网页对顶级窗口进行导航。
- `allow-top-navigation-by-user-activation`：允许嵌入的网页对顶级窗口进行导航，但必须由用户激活。
- `allow-downloads-without-user-activation`：允许在没有用户激活的情况下，嵌入的网页启动下载。

注意，不要同时设置`allow-scripts`和`allow-same-origin`属性，这将使得嵌入的网页可以改变或删除`sandbox`属性。



## 3. loading属性

`<iframe>`指定的网页会立即加载，有时这不是希望的行为。`<iframe>`滚动进入视口以后再加载，这样会比较节省带宽。

`loading`属性可以触发`<iframe>`网页的懒加载。该属性可以取以下三个值。

- `auto`：浏览器的默认行为，与不使用`loading`属性效果相同。
- `lazy`：`<iframe>`的懒加载，即将滚动进入视口时开始加载。
- `eager`：立即加载资源，无论在页面上的位置如何。



```html
<iframe src="https://example.com" loading="lazy"></iframe>
```

上面代码会启用`<iframe>`的懒加载。

有一点需要注意，如果`<iframe>`是隐藏的，则`loading`属性无效，将会立即加载。只要满足以下任一个条件，Chrome 浏览器就会认为`<iframe>`是隐藏的。

> - `<iframe>`的宽度和高度为4像素或更小。
> - 样式设为`display: none`或`visibility: hidden`。
> - 使用定位坐标为负`X`或负`Y`，将`<iframe`>放置在屏幕外。



# 2. \<dialog>,\<details>

本章介绍一些最新引入标准的标签。

## 1. \<dialog>

### 1.1 基本用法

`<dialog>`标签表示一个可以关闭的对话框。

```html
<dialog>
  Hello world
</dialog>
```

上面就是一个最简单的对话框。

默认情况下，对话框是隐藏的，不会在网页上显示。如果要让对话框显示，必须加上`open`属性。

```html
<dialog open>
  Hello world
</dialog>
```

上面代码会在网页显示一个方框，内容是`Hello world`。

`<dialog>`元素里面，可以放入其他 HTML 元素。

```html
<dialog open>
  <form method="dialog">
    <input type="text">
    <button type="submit" value="foo">提交</button>
  </form>
</dialog>
```

上面的对话框里面，有一个输入框和提交按钮。

注意，上例中`<form>`的`method`属性设为`dialog`，这时点击提交按钮，对话框就会消失。但是，表单不会提交到服务器，浏览器会将表单元素的`returnValue`属性设为 Submit 按钮的`value`属性（上例是`foo`）。



### 1.2 JavaScript API

`<dialog>`元素的 JavaScript API 提供`Dialog.showModal()`和`Dialog.close()`两个方法，用于打开/关闭对话框。

开发者可以提供关闭按钮，让其调用`Dialog.close()`方法，关闭对话框。

`Dialog.close()`方法可以接受一个字符串作为参数，用于传递信息。`<dialog>`接口的`returnValue`属性可以读取这个字符串，否则`returnValue`属性等于提交按钮的`value`属性。

```javascript
modal.close('Accepted');
modal.returnValue // "Accepted"
```

`Dialog.showModal()`方法唤起对话框时，会有一个透明层，阻止用户与对话框外部的内容互动。CSS 提供了一个 Dialog 元素的`::backdrop`伪类，用于选中这个透明层，因此可以编写样式让透明层变得可见。

```css
dialog {
  padding: 0;
  border: 0;
  border-radius: 0.6rem;
  box-shadow: 0 0 1em black;
}

dialog::backdrop {
  /* make the backdrop a semi-transparent black */
  background-color: rgba(0, 0, 0, 0.4);
}
```

上面代码不仅为`<dialog>`指定了样式，还将对话框的透明层变成了灰色透明。

`<dialog>`元素还有一个`Dialog.show()`方法，也能唤起对话框，但是没有透明层，用户可以与对话框外部的内容互动。



### 1.3 事件

`<dialog>`元素有两个事件，可以监听。

- `close`：对话框关闭时触发
- `cancel`：用户按下`esc`键关闭对话框时触发

如果希望用户点击透明层，就关闭对话框，可以用下面的代码。

```javascript
modal.addEventListener('click', (event) => {
  if (event.target === modal) {
    modal.close('cancelled');
  }
});
```



## 2. \<details>,\<summary>

### 2.1 基本用法

`<details>`标签用来折叠内容，浏览器会折叠显示该标签的内容。

```html
<details>
这是一段解释文本。
</details>
```

上面的代码在浏览器里面，会折叠起来，显示`Details`，前面有一个三角形，就像下面这样。

```
▶ Details
```

用户点击这段文本，折叠的文本就会展开，显示详细内容。

```
▼ Details
这是一段解释文本。
```

再点击一下，展开的文本又会重新折叠起来。

`<details>`标签的open属性，用于默认打开折叠。

```html
<details open>
这是一段解释文本。
</details>
```

上面代码默认打开折叠。

`<summary>`标签用来定制折叠内容的标题。

```html
<details>
  <summary>这是标题</summary>
  这是一段解释文本。
</details>
```

上面的代码显示结果如下。

```
▶ 这是标题
```

点击后，展示的效果如下。

```
▼ 这是标题
这是一段解释文本。
```

通过 CSS 设置`summary::-webkit-details-marker`，可以改变标题前面的三角箭头。

```css
summary::-webkit-details-marker {
  background: url(https://example.com/foo.svg);
  color: transparent;
}
```

下面的样式是另一种替换箭头的方法。

```css
summary::-webkit-details-marker {
  display: none;
}
summary:before {
  content: "\2714";
  color: #696f7c;
  margin-right: 5px;
}
```



### 2.2 JavaScript API

`Details`元素的`open`属性返回`<details>`当前是打开还是关闭。

```javascript
const details = document.querySelector('details');

if (detail.open === true) {
  // 展开状态
} else {
  // 折叠状态
}
```

`Details`元素有一个`toggle`事件，打开或关闭折叠时，都会触发这个事件。

```javascript
details.addEventListener('toggle', event => {
  if (details.open) {
    /* 展开状况 */
  } else {
    /* 折叠状态 */
  }
});
```



