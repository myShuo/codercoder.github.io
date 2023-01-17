---
id: 793
title: '踩坑之InputStream.read(byte[])方法'
date: '2019-10-13T17:16:17+08:00'
author: Smile
layout: post
guid: 'http://codercoder.cn/?p=793'
permalink: /index.php/2019/10/inputstream-readbyte/
views:
    - '26487'
    - '26487'
    - '26487'
    - '26487'
    - '26487'
bigfa_ding:
    - '1'
categories:
    - Tech
    - Fell-in-pit
tags:
    - Java
    - Tech
    - Fell-in-pit
---

 项目之前都是好好的，最近现场那边出现一个问题，报错不是合法的json字符串，这个json字符串是通过http请求访问获得的。  
 通过直接在浏览器上直接访问http这个请求，发现返回的json也是完全正确的。后来排查代码才发现了原来错误出在从字节流中读取数据这里：  
 看下之前出错代码：这个方法是处理InputStream，然后返回成一个字符串。

```java
public String process(InputStream in, String charset) {
     byte[] buf = new byte[1024];
     StringBuffer sb = new StringBuffer();
     try {
         while (in.read(buf) != -1) {
             sb.append(new String(buf, charset));
         }
    } catch (IOException e) {
         e.printStackTrace();
    }
     return sb.toString();
}

```

 有问题的代码在第6行，发现之前项目没出错的原因是InputStream里面的数据少，还不够1024个字节，while那里循环一次就好了，得到一个正确格式的json串；后面出错了是因为InputStream里面数据比较多，while需要2次了，第一次读取之后buf里面满了，第二次读取发现read(byte\[\])方法不会去清空缓冲区数组，第二次实际上read字节只有100个，buf里面只替换前100个字节内容，然后通过第6行代码append到字符串里了，就造成了最后获得的字符串不是json格式，造成之后json解析出错。

 知道问题后，将之前代码改为下，既然每次不会去清空缓冲区数组内容，那就通过读取长度来append字符串，问题解决：

```java
public String process(InputStream in, String charset) {
    byte[] buf = new byte[1024];
    StringBuffer sb = new StringBuffer();
    int len = 0;
    try {
        while ((len=in.read(buf)) != -1) {
            sb.append(new String(buf, 0, len, charset));
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return sb.toString();
}

```

 后来查了下JDK API，发现API上也说明过了，只是以前没注意，关键在于加黑字体，不影响b\[k\]到b\[b.length-1\]的元素：

 附上API：

 public int read(byte\[\] b) throws IOException   
 从输入流中读取一定数量的字节，并将其存储在缓冲区数组 b 中。以整数形式返回实际读取的字节数。  
 在输入数据可用、检测到文件末尾或者抛出异常前，此方法一直阻塞。  
 如果 b 的长度为 0，则不读取任何字节并返回 0；否则，尝试读取至少一个字节。如果因为流位于文件末尾而没有可用的字节，则返回值 -1；否则，至少读取一个字节并将其存储在 b 中。将读取的第一个字节存储在元素 <font color="#0099ff">b\[0\]</font> 中，下一个存储在 <font color="#0099ff">b\[1\]</font> 中，依次类推。读取的字节数最多等于 b 的长度。设 k 为实际读取的字节数；这些字节将存储在 <font color="#0099ff">b\[0\]</font> 到 <font color="#0099ff">b\[k-1\]</font> 的元素中，不影响 <font color="#0099ff">b\[k\]</font> 到 <font color="#0099ff">b\[b.length-1\]</font> 的元素。