---
title: 三、深入分析Java Web中的中文编码问题
date: 2017-08-07 22:56
updated: 2018-12-22 18:00
tags:
  - 编码
categories: 
  - 理论
  - 深入分析Java_Web技术
comments: true
permalink: deepknowjavaweb/2_java_io.html  
---

![][0]

<!-- more -->

# 1 几种常见的编码格式

## 1.1 为什么需要编码？

1. 计算机中存储信息的最小单位是1个字节，即8个bit，所以能表示的字符范围是2^8=256个
2. 人类符号过于复杂，至少一个几个字节才能满足人类的一个单位

## 1.2 常见编码

编码即就是人类的字符->机器的字符的过程。

### 1.2.1 ASCII码

总共有128个，用1个字节的低七位表示，0~31是控制字符，如换行、回车、删除，32~126是打印字符，可以通过键盘输入并且能够显示出来。

### 1.2.2 ISO-8859-1

128个字符显示是不够的，于是ISO组织在ASCII码基础上又制定了一系列标准来扩展ASCII编码，他们是ISO-8859-1至ISO-8859-15。ISO-8859-1仍然 是单字节编码，它总共能表示256个字符。

### 1.2.3 GB2312

GB2312全称是《信息技术·中文编码字符集》，总的编码范围是：A1~F7。它是双字节编码。包含了符号以及汉字。

### 1.2.4 GBK

GBK全称是《汉字内码扩展规范》，是国家技术监督局为Windows95所制定新的汉字内码规范，它的出现是为了扩展GB2312，并加入更多的汉字。 编码范围是8140~FEFE，总共23940，表示21003个汉字，编码是和GB2312兼容，也就是GB2312编码的汉字可以用GBK解码，不会乱码。

### 1.2.5 GB18030

应用不广泛，与GB2312兼容

### 1.2.6 UTF-16

1. Unicode（Universal Code统一码），ISO试图创建一个全新的超语言字典，世界上所有的语言都可以通过这个字典来相互翻译。可想而知这个字典是多么 复杂。Unicode是Java和XML的基础
2. UTF-16具体定义了Unicode字符在计算机中的存取方法，UTF-16用两个字节来表示Unicode的转化格式，它采用定长的表示方法，即不论什么字符都可以用 两个字节表示
3. 两个字节是16个bit，所以叫UTF-16。UTF-16表示字符非常方便，每两个字节表示一个字符，简化了字符串操作，这也是Java以UTF-16作为内存的字符存储格式的一个重要的原因

### 1.2.7 UTF-8

UTF-16统一采用两个字节表示一个字符，虽然表示上简单方便，但是也有其缺点，很大一部分字符用一个字节就可以表示的现在要用两个字节表示，存储空间放大了一倍。而UTF-8采用了一种变长技术，每个编码区域有不同的字码长度。不同类型的字符可以由1~6个字节组成。  
1. 如果是一个字节。最高为为0，则表示这是一个ASCII字符，可见，所有ASCII编码已经是UTF-8了。
2. 如果是一个字节，以11开头，则连续的1的个数暗示这个字符的字节数。例如：110xxxxx代表它是双字节UTF-8字符的首字节。
3. 如果是一个字节，以10开始，表示它不是首字节，需要向前查找才能得到当前字符的首字节。

## 1.2 编码的场景

### 1.2.1 I/O操作

Reader类和InputStream之间的InputStreamReader，通过StreamDecoder以及StreamEncoder进行字符和字节的转换，在解码过程必须指定编码格式， 否则按系统编码。  
```java
String file = "D:\\source\\eclipse\\liwen\\src\\main\\java\\liwen\\com\\io\\data.txt";
String charset = "UTF-8";
FileOutputStream fileOutputStream = new FileOutputStream(file);
OutputStreamWriter writer = new OutputStreamWriter(fileOutputStream);
writer.write("这是要保存的中文字符");
writer.close();
FileInputStream fileInputStream = new FileInputStream(file);
InputStreamReader reader = new InputStreamReader(fileInputStream, charset);
char[] buf = new char[1024];
int count = 0;
StringBuffer buffer = new StringBuffer();
while((count = reader.read(buf)) != -1) {
    buffer.append(buf, 0, count);
}
System.out.print(buffer.toString());
reader.close();
```

### 1.2.2 在内存操作中的编码

```java
// 第一种，通过字符串操作
String s = "中文";
byte[] b = s.getBytes("UTF-8");
String n = new String(b, "UTF-8");
System.out.print(n);
// 第二种，通过nio中的Charset与Buffer实现编码解码。
Charset charset = Charset.forName("UTF-8");
ByteBuffer buffer = charset.encode(s);       //字符转字节
CharBuffer buffer1 = charset.decode(buffer); //字节转字符
char[] a = buffer1.array();
System.out.print(a);
// 第三种，通过将16bit的char拆分为2个8bit的byte，没有编码解码，只是软转化
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
ByteBuffer byteBuffer1 = byteBuffer.putChar('a');
```

# 3 在Java中如何编解码

1. UTF_32，GBK等编码都是继承自Charset（查看GB18030类的源码，会让你大吃一惊）。
2. Java内存编码采用的UTF-16编码，编码效率高，虽然用双字节存储，但是不适合网络之间传输，因为网络传输容易损坏字节流，当一个字节损坏，就两个字节没用了，UTF-8更适合网络传输。
3. UTF-8对ASCII字符采用单字节存储，另外单个字符损坏也不会影响后面的其他字符，编码效率上介于GBK和 UTF-16之间，所以UTF-8在编码效率上和编码安全性上做了平衡，是理想的中文编码方式。

# 4 在Java Web中设计的编解码

1. 有I/O的地方就会涉及编码。网络传输都是以字节为单位的，所以所有的数据必须能够被序列化，即继承Serializable
2. 一个文本的实际大小应该怎么计算。例如：把整型数字1234567当做字符哎存储，则采用UTF-8编码会占用7个字节，采用UTF-16编码
会占用14个字节，但是当把它当成int类型的数字来存储则只需要4个字节  
![][1]

## 4.1 URL的编码

![][2]  
1. 其中浏览器对PathInfo和QueryString是编码不同的，因此服务器分别在不同的地方对其进行解码
2. 例如Tomcat先判断URIEncoding是否有定义，如果没有则默认使用ISO-8859-1解析
3. 而QueryString，无论POST请求还是GET请求，对它们的解码都是在request.getParameters()方法中，当然内部对POST和GET解码是不同的
4. 其中GET请求，是通过HTTP的Header传到服务端的，是通过useBodyEncodingForURL设置。因此在服务器最好设置URIEncoding和useBodyEncoding两个参数

## 4.2 HTTP Header的编解码

如Cookie等，一些头信息，Tomcat对Header解码是在调用request.getHeader()方法时进行的。如果有非ASCII字符，使用URLEncoder进行编码，网络传输。

## 4.3 POST表单的编解码

提交时，浏览器先根据ContentType的Charset编码进行参数编码，然后再提交到服务端，服务端同样也用ContentType中的字符集进行解码。服务端可以通过 request.setCharacterEncoding(charset)来设置
>要在第一次调用request.getParameter()方法之前就设置request.setCharacterEncoding(charset)。
如果服务端没有设置request.setCharacterEncoding(charset)，那么表单提交的数据将会按照系统的默认编码方式解析。  
另外，针对multipart/form-data类型的参数，即上传文件，也是通过ContentType定义的字符编码。上传文件是用字节流的方式传输到服务器的本地 临时目录，这个过程并没有涉及字符编码，而真正编码是在讲文件内容添加到parameters，如果用这个不能编码，则会使用默认的ISO-8859-1编码。

## 4.4 HTTP BODY的编解码

通过Response返回给客户端浏览器。这个过程要经过编码，即response.setCharcterEncoding()来设置，它将会覆盖request.getCharacterEncoding() 的值，并通过Header的Content-Type返回客户端，浏览器接收到返回的Socket流时将通过Content-Type的charset来解码。如果返回的HTTP Header中Content-Type 没有设置charset，那么浏览器将根据浏览器的中的charset来解码， 如果浏览器中没有定义，则使用默认的编码。  
连接JDBC也是指定一致的编码：`jdbc:mysql://localhost:3306?DB?useUnicode=true&characterEncoding=GBK`

# 5 在JS的编码

## 5.1 外部引入JS文件

`<script scr="script.js" charset="gbk"></script>`  
而script.js脚本中，有如下代码：  
`document.write("中国");`  
如果引入的时候没有设置charset，浏览器就会以当前页面的默认字符集解析这个JS文件。如果一致那就没问题，但是如果页面和js字符编码不一致，就会变成乱码。

## 5.2 JS的URL编码

### 5.2.1 escape()

![][3]  
这组函数已经从ECMAScript v3标准删除了，URL的编码可以用encodeURI和encodeURIComponent来代替。

### 5.2.2 encodeURI()

![][4]  
对某些特殊的字符不进行编码如 `!`、`a-z`、`A-Z`、`0-9`、`=`、`@`、`?`、`;`、`:`、`-`、`+`、`(`、`)`、`&`、`#`、`.`、`~`、`*`。

### 5.2.3 encodeURIComponent()

![][5]  
编码更加彻底，用于整个URL编码，因为它将&也编码了。除了`!`、`a-z`、`A-Z`、`0-9`、`-`、`.`、`~`、`*`

### 5.2.4 Java与JS编解码问题

1. Java端处理URL编解码有两个类，分别是URLEncoder和URLDecoder。这两个类可以将所有“%”加UTF-8码值用UTF-8解码，从而得到原始的值
2. 对应的前端JS是encodeURIComponent和decodeURLComponent。注意，前端用encodeURIComponent，服务端用URLDecoder解码可能会乱码， 可能是两个字符编码类型不一致，JS编码默认是UTF-8编码，而服务端中文解码一般都是GBK或者GB2312，所以encodeURIComponent编码后是 UTF-8，而Java用GBK去解码显然不对
3. 解决方式是encodeURIComponent两次编码，服务端使用request.getParameter()用GBK解码后，再用UTF-8解码

# 6 常见编码问题

## 6.1 中文变成看不懂的字符

```java
String a = "淘！我喜欢！";
byte[] b = a.getBytes("GBK");  //可以表示中文，占两个字节
String c = new String(b, "ISO-8859-1");  //将两个字节分别作为一个单独的字符显示
System.out.println(c); // output: ÌÔ£¡ÎÒÏ²»¶£¡
```
双字节变成单字节  
![][6]

## 6.2 中文变成一个问号

```java
String a = "淘！我喜欢！";
byte[] b = a.getBytes("ISO-8859-1");    //找不到对应的字符
String c = new String(b, "ISO-8859-1");
System.out.println(c); // ??????
```
中文变成问号  
![][7]

## 6.3 中文变成两个问号

经过了多次的编码解码  
![][8]

## 6.4 一种不正常的正确编码

直接调用 String value = request.getParameter(name);会出现乱码。  
但是 String value = new String(request.getParameter(name).getBytes("ISO-8859-1"), "GBK")会正常，为什么呢？  
![][9]  
网络通过GBK编码之后的字节数组进行传输，Tomcat没有配置useBodyEncodingForURI，造成第一次解析通过ISO-8859-1解析， 这时候我们手动通过ISO-8859-1编码，再通过GBK解码就可以获得正确的值，但是额外增加了一次编解码过程。

# 7. 总结

总结了几种常见编码格式的区别：  
1. ISO-8859-1：单字节编码，最多能表示256个字符。
2. GBK、GB2312：双字节编码，前者兼容后者。
3. UTF-16：双字节编码，Java内部内存额字符存储格式，操作方便，全部都是两个字节，但是浪费空间。
4. UTF-8：动态字节编码。
5. 以及IO的编码实现类：StreamEncoder/StreamDecoder，对char和byte的编解码。

HTTP过程的编码，包括：  
1. URL、URI的编码。
2. Header的编解码。
3. POST表单的编解码。Java使用request.getParameter()获取之前，先设置request.setCharacterEncoding(charset)。
4. BODY的编解码，即Response的编解码。
5. JS的编解码。
6. Tomcat编解码源码。以及常见乱码问题的原因。  

注意一定要手动设置编码的格式，实现真正的跨平台。

[0]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/background/2018-10-03%E5%B8%B8%E5%B7%9E%E5%A1%94.jpg
[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-1.png
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-2.png
[3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-3.png
[4]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-4.png
[5]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-5.png
[6]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-6.png
[7]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-7.png
[8]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-8.png
[9]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/deeplearnjavawebtech/deeplearnjavawebtech-3-1-9.png