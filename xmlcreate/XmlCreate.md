## 1. XmlNode

[TOC]

![image-20200507102054137](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200507102054137.png)

##### 1.1 XmlNode为普通单条对象

##### 1.2  通过构造函数进行传值

```java
//key为element name value 为TextValue next 为文件树的下一个节点
XmlNode(K key, V value, XmlNode<K,V> next){
    	this.key = key;
        this.value = value;
        this.next = next;
}
```

```java
//前三个参数和上面一致 增加了 atrr的属性名 和属性值
XmlNode(K key, V value, XmlNode<K,V> next,String attrKey,String attrValue) {
    this.key = key;
    this.value = value;
    this.next = next;
    this.attrKey = attrKey;
    this.attrValue = attrValue;
}
```

------

## 2. CollectXmlNode



![image-20200507101614138](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200507101614138.png)



##### 2.1 collectXmlNode为集合node

##### 2.2 构造函数参数

```java
//next 为下一个node
CollectXmlNode(XmlNode<K, V> next) {
        this(null,null,next);
}
```

##### 2.3 方法使用

```java
//获取集合node
public List<XmlNode<K, V>> getNodes() {
    return collectNodes;
}
```

```java
//添加node到集合 单笔
public void addNode(XmlNode<K, V> xmlNode) {
    collectNodes.add(xmlNode);
}
```

```java
//添加到集合 多笔
public void addNodes(List<XmlNode<K, V>> xmlNode) {
    try {
        lock.lock();
        collectNodes.addAll(xmlNode);
    }finally {
        lock.unlock();
    }
}
```

------

## 3. XmlCreateBuild

![image-20200507105652376](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200507105652376.png)

##### 3.1 xmlEncoding-头编码 xmlStandalone  encoding-生成xml字符串编码  head-node头

##### 3.2 方法

```java
//获取xml字符串 通过head
public String getXmlString() throws Exception {
    if(head == null){
        throw new NullPointerException("XmlCreateBuild head is null");
    }
    DocumentImpl document = getDocument();
    parseNode(document,head);
    String xmlStr = getString(document);
    return xmlStr;
}
```

```java
//获取文件 用过head 默认地址
public File getXmlFile() {
    ......
    return file;
}
```

```java
//获取文件 自定义地址
public File getXmlFile(String fileSource) {
    ......
    return file;
}
```

```java
//通过传入的 xml字符串 和实体的class 最终返回对应的实体对象
public <F> F parseToEntity(String xml, Class<F> entity) {
    .......
    return F;
}
```