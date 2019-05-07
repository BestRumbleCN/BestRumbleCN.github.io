---
title: Sofa源码学习（1）- 深入学习Hessian序列化
date: 2019-05-07 19:14:03
tags: sofa hessian java
---
#前言
最近在和其他服务间联调的时候，神奇地发现请求参数中的一个Map字段的类型居然是ConcurrentHashMap,仔细研究了下这个情形，得出一个结论一个猜想：
 - 请求调用方问题很大！
    无论何种场景，请求参数里都不应该使用ConcurrentHashMap！
 - hessian序列化Map,List等集合时难道会带上具体的集合类型？
    如果这个猜想正确，那么我们在创建和使用服务间的调用参数时要小心了!就拿上面的例子来说，客户端传入的Map类型是ConcurrentHashMap,服务端如果只当普通的map操作，很可能报错（ConcurrentHashMap的key,value都不能为空）！这样的场景其实是很多的，譬如Collections.emptyMap()、Collections.unmodifiableMap()都是不可变的map,但是hessian在反序列化这两个map时却都变成了HashMap。现象很奇怪，决定带着问题好好地研究下hession源码。
<!-- more -->
## 1. 待解决的问题

 1. hessian序列化的结构是怎样的？
 2. 集合类是如何具体序列化的?为什么ConcurrentHashMap和Collections.emptyMap()结果不一样？
 3. 相比json序列化有什么优缺点？

## 2. hessian语法
### 2.1 hessian序列化逻辑结构
hessian序列化的核心是一张**二进制码映射表**（Bytecode map）,通过两个byte(或四个byte)指定接下来数据的类型、长度等数据信息。详细可参考：[英文文档](http://hessian.caucho.com/doc/hessian-serialization.html)，[中文文档](https://www.jianshu.com/p/e800d8af4e22)。
下面，以一个简单的对象序列化为例。
类定义如下：
``` java
class Car {
  String color;
  String model;
}
```
序列化：
``` java
out.writeObject(new Car("red", "corvette"));
out.writeObject(new Car("green", "civic"));
```
结果：
```bash
  x43                   # 类定义,x43代表类定义的二级制码。
  x0b example.Car       # 类名 xob指定了类名长度是11位
  x92                   # 该类有两个属性（x90代表0，x92代表2）
  x05 color             # 属性名是coler x05代表属性名长度
  x05 model             # 第二个属性名是model

  x4f                   # 对象定义(long form)
  x90                   # 代表当前对象是第0个类的实现
  x03 red               # color field value
  x08 corvette          # model field value

  x60                   # 代表当前对象是第0个类的实现(short form)
  x05 green             # color field value
  x05 civic             # model field value
```
分析:
    hessian在序列化第一个Car对象时，会先判断之前是否有序列化过Car类型的对象，没有的话，则先生成Car类的定义，然后再序列化对象。**类和字段只会序列化一次，对象只会序列化值**。

### 2.2 hessian Map的序列化逻辑结构
``` java
map = new HashMap();
map.put(new Integer(1), "fee");
map = Collections.unmodifiableMap(map);
out.writeObject(map);

---

  x4d                                       # map定义
  x91 java.util.Collections.UnmodifiableMap # map类型

  xa0                                       # 16
  x03 fie                                   # "fie"

  x5a                                       # map结束
```
从上面的例子中，我们可以看到在序列化Map时，hessian也会记录具体的map类型。那么为什么会有的map可以反序列化出来，有的却变成HashMap呢？我们接着往下分析

### 2.3 hessian Map的序列化源码分析
我们是基于sofa分析hessian源码的，以客户端请求服务端为例，其序列化核心类为：SofaRequestHessianSerializer。
#### 2.3.1 序列化
    略
#### 2.3.2 反序列化
    我们着重分析Map类型的反序列化。
1.0 SofaRequestHessianSerializer
    decodeObjectByTemplate方法是反序列化的入口，内部调用了Hessian2Input的readObject()方法。
``` java
 public void decodeObjectByTemplate(AbstractByteBuf data, Map<String, String> context, SofaRequest template) throws SofaRpcException {
        try {
            UnsafeByteArrayInputStream inputStream = new UnsafeByteArrayInputStream(data.array());
            Hessian2Input input = new Hessian2Input(inputStream);
            input.setSerializerFactory(this.serializerFactory);
            Object object = input.readObject();//反序列化入口
            SofaRequest tmp = (SofaRequest)object;
            String targetServiceName = tmp.getTargetServiceUniqueName();
                //.....省略代码。。。。
                String[] sig = template.getMethodArgSigs();
                Class<?>[] classSig = ClassTypeUtils.getClasses(sig);
                Object[] args = new Object[sig.length];

                for(int i = 0; i < template.getMethodArgSigs().length; ++i) {
                    args[i] = input.readObject(classSig[i]);//反序列化入口
                }
                template.setMethodArgs(args);
                input.close();
        } catch (IOException var14) {
            throw this.buildDeserializeError(var14.getMessage(), var14);
        }
    }
```
1.1 Hessian2Input
通过一个Deserializer工厂类获取当前class的Deserializer，最终通过Deserialize反序列化map，Deserializer是一个接口，不同的class有不同的实现，map类型的class对应的是MapDeserializer。
``` java
public Object readObject(Class cl) throws IOException {
        if (cl != null && cl != Object.class) {
            int tag = this._offset < this._length ? this._buffer[this._offset++] & 255 : this.read();
            int ref;
            String type;
            String type;
            int size;
            switch(tag) {
            case 77:
                type = this.readType();
                Deserializer reader = this.findSerializerFactory().getObjectDeserializer(type, cl);
                return reader.readMap(this);// 核心方法
            }
            //。。。省略。。。。
        }
```
1.2 MapDeserializer
MapDeserializer通过反射获取Class的公共无参构造方法，如果不存在这样的构造方法，则使用HashMap的构造方法！而Collections.emptyMap()、Collections.unmodifiableMap()，都是内部私有类，没有公共的无参构造方法！真相大白！
``` java
public class MapDeserializer extends Deserializer {
    private Class _type;
    private Constructor _ctor;
    public MapDeserializer(Class type) {
        this._type = type;
        Constructor[] ctors = type.getConstructors();
        for(int i = 0; i < ctors.length; ++i) {
        //获取公共的无参构造方法
            if (ctors[i].getParameterTypes().length == 0) {
                this._ctor = ctors[i];
            }
        }
        //若没有则使用HashMap
        if (this._ctor == null) {
            this._ctor = HashMap.class.getConstructor();
        }
    }
    // 反序列化方法
    public Object readMap(AbstractHessianInput in) throws IOException {
        //反射生成Map对象
        Object map = (Map)this._ctor.newInstance();
        in.addRef(map);
        while(!in.isEnd()) {
            ((Map)map).put(in.readObject(), in.readObject());
        }
        in.readEnd();
        return map;
    }
}
```
## 2.4 总结
1.针对Map，Hessian会使用该Map类的公共无参构造方法生成实例，如果不存在这样的构造方法，则使用HashMap替换。
2.Hessian针对序列化做了很多优化，例如类字段只序列化一次，同一对象多次出现也只序列化一次等。
  [1]: http://hessian.caucho.com/doc/hessian-serialization.html
  [2]: https://www.jianshu.com/p/e800d8af4e22