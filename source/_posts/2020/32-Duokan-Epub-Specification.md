---
title: 多看电子书规范扩展开放计划
date: 2020-08-01 20:26:44
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/reading_book.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 多看阅读
    link: http://www.duokan.com/

# post subtitle in your index page
subtitle: 本文《多看电子书规范扩展开放计划》原文内容转载。

categories: 
  - 读书笔记

tags: 
  - 读书笔记
---

## 背景介绍

博客有一个月没有更新了，今天更新一篇，不过也是与技术无关的内容，转载一份《多看电子书规范扩展开放计划》，多看是我非常喜欢的一款电子书阅读 `APP`，主要是电子书制作比较精美，而且还有一套开放的电子书制作标准，供广大书友参照，很多电子书的爱好者也参考这个标准制作了很多不错的电子书，最近在[多看阅读](http://www.duokan.com/)的官网论坛上已经搜索不到这份开放的标准了，我也是在网上的二手渠道才找到的，所以记录到自己的博客上，方便自己以后查看。

## 多看电子书规范扩展开放计划（20140819第四弹）

### 一、全屏插图页

全屏插图页扩展在 `ePub` 文件的 `opf` 文件中的 `spine` 节点下，`spine` 节点定义了 `ePub` 文件中文章出现的顺序，每一个 `itemref` 项为一章，我们扩展一个 `properties` 属性值 `duokan-page-fullscreen`，示例如下：

```html
<spine>
    <itemref idref="chapter100" properties="duokan-page-fullscreen">
    ……
</spine>
```

这样 `id` 为 `chapter100` 的章就会按全屏插图页逻辑处理，而相应的 `html` 内容应如下所示：

```html
<?xml version="1.0" encoding="utf-8"standalone="no"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN"
    "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title></title>
    <link href="../Styles/stylesheet.css"rel="stylesheet" type="text/css" />
  </head>

  <body>
    <p><img src="../Images/sanguoyanyi.png"/></p>
  </body>
</html>
```

注意，`html` 中应只含有一个 `img`，不需要设置任何样式，程序会自动将图片撑满展示。

### 二、富文本脚注

用户可以通过单击文内脚注的图标，弹出显示脚注内容的窗口。文内注可以支持复杂的内容描述，比如多段落，带有样式的文本等等，具体描述如下：

在需要插入注的位置插入如下代码：

```html
<a class="duokan-footnote"href="#df-1"><img src=" ../Images/note.png"/></a>
```

在文章的末尾插入如下代码：

```html
<ol class="duokan-footnote-content">
    <li class="duokan-footnote-item"id="df-1"><p>这是一个注释文本。</p></li>
</ol>
```

注和内容之间使用 `id` 链接，通过这样的扩展方式，可以将整个章节的所有文内注内容集中在一个有序列表中，这部分内容不会直接在页面上渲染出来，而是通过应用层的交互来呈现。

### 三、交互图

对于交互图，应用层会响应点击放大操作，提供额外的交互体验，具体扩展如下所示：

```html
<div class="duokan-image-single">
  <img src="../Images/tree.png"alt="" />
  <p class="duokan-image-maintitle">主标题：大自然</p>
  <p class="duokan-image-subtitle">副标题：森林中的树</p>
</div>
```

为了保证点击放大之后的图像呈现效果，采用交互模式的图像数据应该保证足够的分辨率。

注意：
1. 主标题和副标题可以不出现；
2. 主标题和副标题可以在 `img` 之前出现；
3. 交互图不可以嵌套出现。

//========================朴素的分割线========================
// 20130930 更新第二弹

### 四、多看字体使用

多看官方客户端可用字体列表：

**CSS写法：**

```
效果
font-family: "DK-SONGTI";
使用多看系统自带宋体。
font-family: "DK-FANGSONG";
使用多看系统自带仿宋。
font-family: "DK-KAITI";
使用多看系统自带楷体。
font-family: "DK-HEITI";
使用多看系统自带黑体。
font-family: "DK-XIAOBIAOSONG";
使用多看系统自带小标宋。
font-family: "DK-XIHEITI";
使用多看系统自带细黑体。
font-family: "DK-SERIF";
使用多看系统自带衬线西文字体。
font-family: "DK-CODE";
使用多看系统自带等宽西文字体。
font-family: "DK-SYMBOL";
使用多看系统自带符号字体（不常见符号，如音标等）。
```

**示例：**

首先在CSS文件中增加下面的样式

```html
p.usekaiti {
  font-family: "DK-KAITI";
}
```
然后在xhtml文件中使用

```html
<p class="usekaiti">这段文字使用楷体显示</p>
```

这样就 ok 了。

// ========================朴素的分割线========================
// 20140307 更新第三弹

### 五、多媒体（音频、视频）
         
由于移动设备的一些特性，`html` 中标准的音频、视频的排版特点并不完全满足我们的需求，因此，需要进行一些“小小”的扩展，才能得到比较完美的展示效果。

**音频**

在 `HTML 5` 规范中，音频采用标准的 `audio` 标签，但需要扩展说明其占位图像，示例如下：

```html
<audio class="duokan-audio content-speaker" placeholder="speaker.jpg" activestate="active-speaker.jpg" title="军港之夜">
    <source src="song.mp3" type="audio/mpeg" />
</audio>
```

`duokan-audio` 为扩展标签，表明了该 `audio` 标签为多看扩展类型，`placeholder` 用于表示占位图，`activestate` 表示活动状态下的占位图，`title` 表示标题名。

一般情况下可以指定 `audio` 采用的 `CSS` 样式，在绘制占位图时需要遵循该样式，示例代码如下：

```html
audio.content-speaker {
    font-size: 16px;
    width: 0.8em;
}
```

`audio` 的 `controls` 属性出现时，表明应用层需要显示控制栏，如果不出现，则无需显示控制栏。

**视频**

在 `HTML 5` 规范中，视频采用标准的 `video` 标签，示例如下：

```html
<video class="duokan-video content-matrix" poster="matrix.jpg">
    <source src="matrix.mp4" type="video/mp4" />
</video>
```

`duokan-video` 为扩展标签，表明了该 `video` 标签为多看扩展类型。

一般情况下可以指定 `video` 采用的 `CSS` 样式，在绘制 `poster` 时需要遵循该样式，示例代码如下：

```html
video.content-matrix {
    width: 320px;
    height: 240px;
}
```
`video` 的 `controls` 属性禁止出现。

//========================朴素的分割线========================
// 20140819 更新第四弹

### 六、画廊模式

画廊模式可以支持在同一个位置显示多张大小一致的图像，用户可以通过滑动等手势切换不同的图像内容。
如下，即为一个拥有三帧画面的画廊（每一个 `duokan-image-gallery-cell` 声明一个分帧）：

```html
<div class="duokan-image-gallery">
    <div class="duokan-image-gallery-cell">
        <img src="images/tree1.png"alt="" />
        <p class="duokan-image-maintitle">主标题：大自然</p>
        <p class="duokan-image-subtitle">副标题：柏树</p>
    </div>
    <div class="duokan-image-gallery-cell">
        <imgsrc="images/tree2.png" alt="" />
        <p class="duokan-image-maintitle">主标题：大自然</p>
        <p class="duokan-image-subtitle">副标题：柳树</p>
    </div>
    <div class="duokan-image-gallery-cell">
        <imgsrc="images/tree3.png" alt="" />
        <p class="duokan-image-maintitle">主标题：大自然</p>
        <p class="duokan-image-subtitle">副标题：杨树</p>
    </div>
</div>
```

注意：
1. 各分帧图片最好保持同样大小；
2. 最好各分帧都存在主标题和副标题；
3. 画廊整体样式应该放在 `duokan-image-gallery` 所在的 `div` 上。

//========================朴素的分割线========================

### 注意事项

<font color="red">首次制作多看版图书时，不注意就会发生的悲剧：</font>

1. 图像路径大小写不匹配，导致图像各种不出现，请注意 `ePub` 规范大小写敏感；
2. 富文本注内容使用 `ul` 标签，导致注点击无反应，更甚可能死机哦，请注意注内容应该放置在 `ol` 标签内；
3. 应该使用半角符号；
4. <font color="red">一定要注意红色部分属性和内容。</font>