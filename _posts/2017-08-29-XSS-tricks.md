---
layout: post
title: "XSS 技巧整理"
date: 2017-08-29
categories: Note
---

1.
```
  for(var i=0,tags=document.querySelectorAll('iframe[src],frame[src],script[src],link[rel=stylesheet],object[data],embed[src]'),tag;tag=tags[i];i++){
    var a = document.createElement('a');
    a.href = tag.src||tag.href||tag.data;
    if(a.hostname!=location.hostname){
      console.warn(location.hostname+' 发现第三方资源['+tag.localName+']:'+a.href);
    }
  }
  // 用于发现第三方资源调用

2. 引用编码解码 html实体编码，进制编码，十六进制，十进制。JS：unicode编码，十六进制，八进制，纯转义。CSS：八进制，十六进制。

3. SVG 是个好东西。
4. `–!>` 也能闭合注释
5. input 的type 属性，后面的不能覆盖前面的。
6. form 的 action 也能执行 JavaScript。
7. 由于xml编码特性。在SVG向量里面的script元素（或者其他CDATA元素 ），会先进行xml解析。因此&#x28（十六进制）或者&#40（十进制）或者&lpar；（html实体编码）会被还原成
8. 可利用 特殊编码例如： U+2028，是Unicode中的行分隔符，U+2029，是Unicode中的段落分隔符。
9. toUppercase支持unicode字符，例：字符ſ经过函数toUpperCase()处理后，会变成ASCII码字符”S”。
10. 在大部分操作符都被过滤的情况下，可尝试 `in` 操作符，例：`(prompt(1))in"."`
11. replace()这个函数，他还接受一些特殊的匹配模式。$` 替换查找的字符串，并且在头部加上比配位置前的字符串部分。
`'123456'.replace('34','$`xss')` 得到 `	'1212xss56'`
12. `"onclick=prompt(1) id="a";callback=a.click;`