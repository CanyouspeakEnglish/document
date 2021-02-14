## JVM

##### 1 类装载机制

https://docs.oracle.com/javase/8/

开发者文档 https://docs.oracle.com/javase/8/docs/index.html

java 虚拟机规范 https://docs.oracle.com/javase/specs/jvms/se8/html/index.html

类文件格式 https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html

![image-20201230122258867](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20201230122258867.png)

```txt
ClassFile {
    u4             magic; //表示 cafe babe
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
每两位代表一个u1
```

1. Loading

   > 类加载器
   >
   > BootstrapClassLoader
   >
   > ExtensionClassLoader
   >
   > AppClassLoader

   使用Bootstrap类加载器加载

   使用用户自定义的类加载器加载 --双亲委派

   创建数组类 --jvm创建 使用特定的类加载器

   将类的描述信息加载到***方法区***

2. Linking

   进行类验证--验证确保类和接口二进制结构正确

   准备 --初始化静态属性赋值默认值

   解析

   - 类和接口解析--类和接口权限解析验证，数组验证
   - 属性解析
   - 方法解析
   - 接口方法解析
   - 方法类型和方法实现解析
   - 正常的放回值和异常的返回值

   权限--权限验证

   重新--子类重写父类方法

3. Initialization--初始化类或者接口初始化方法

4. 绑定本地方法

5. 退出虚拟机

   ![image-20200825152913358](D:\typora\image-20200825152913358.png)

![image-20200825174733296](D:\typora\image-20200825174733296.png)

##### 2 垃圾回收器

####     引用/可达性算法

1. 串行收集器

   - Serial--新生代收集器--复制算法
   - SerialOld--老年代收集器--单线程标记清理

2. 并行收集器

   - Parallel Scavenge --新生代收集器--多线程复制算法(吞吐量大)吞吐量 = 程序执行时间 / 程序执行时间+停顿时间
   - ParNew --新生代收集器--多线程复制算法
   - Parallel Old--老年代收集器--多线程标记清理算法(只对整个堆进行压缩)

3. 并发收集器

   - CMS --老年代收集器--标记-并发标记-再次标记--并行清理 (停顿时间短) 

   - G1--新生代和老年代垃圾收集器--标记--并发标记--再次标记--选择清理(尽可能减少停顿时间 标记压缩算法) set pause tartgets (更改了内存结构 regions)-XX:+UseG1GC -XX:MaxGCPauseMillis=200

     ![image-20200825174113478](C:\Users\lzx\AppData\Roaming\Typora\typora-user-images\image-20200825174113478.png)

   

##### iso网络7层模型

![image-20200829112827203](D:\typora\image-20200829112827203.png)

jvm 图

![jvm](C:\Users\Administrator\Desktop\document\jvm.png)