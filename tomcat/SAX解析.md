# SAX解析

### 工作原理

SAX，它既是一个接口，也是一个软件包.但作为接口，SAX是[事件驱动](https://baike.baidu.com/item/事件驱动)型XML解析的一个标准接口不会改变 SAX的工作原理简单地说就是对文档进行顺序扫描，当扫描到文档（document）开始与结束、元素（element）开始与结束、文档（document）结束等地方时通知事件处理函数，由事件处理函数做相应动作，然后继续同样的扫描，直至文档结束。

大多数SAX都会产生以下类型的事件：

1.在文档的开始和结束时触发文档处理事件。

2.在文档内每一XML元素接受解析的前后触发元素事件。

3.任何元数据通常由单独的事件处理

4.在处理文档的DTD或Schema时产生DTD或Schema事件。

5.产生错误事件用来通知主机应用程序解析错误。

过程：

1.首先SAXParserFactory来创建一个SAXParserFactory实例

SAXParserFactory factory = SAXParserFactory.newInstance();

2.根据SAXParserFactory实例来创建SAXParser

3.SAXParser产生SAXReader

XMLReader reader = factory.newSAXParser().getXMLReader();

4.XMLReader 加载XML，然后解析XML，在解析的过程中触发相对于接口中的事件处理程序