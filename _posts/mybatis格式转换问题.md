# MyBatis类型问题

今天同事用SpringCloud的时候发现一个很奇怪的问题，首先问题描述如下：
微服务A和微服务B之间有个方法，A调用B返回的类型为MyObjectA，简化如下：
```java
class MyObjectA{
	private String Title;
	private String Other;
	private List<MyObjectB> list;
	// 省略get set方法
}

class MyObjectB{
	private String Name;
	private String ID;
	private String BOther;
	// 省略get set方法
}

// C在A中没有用到
class MyObjectC{
	private String Name;
	private Strign ID;
	private String COther;;
}
```

其中A,B,C对应的都有数据库表。
然后调用的时候发现，在服务A中拿到的MyObjectA的实例中，list中的MyObjectB的变量全为null，而MyObjectA的其他变量比如Title，Other这些不为空。但是在服务B中打断点看到的返回的MyObjectA中的list的MyObjectB的变量是有值的，这就很奇怪了。

看到问题，我首先想到的是MyObjectA,MyObjectB是不是没有写get set方法导致序列化出问题了，然后检查了一下，MyObjectA,MyObjectB都有写，然后在A的熔断器里面打了日志，发现也没有进入熔断器。代码里面没有加ControllerAdvice，如果进入熔断器了，应该返回的是null，而不是部分数据为null。这种问题很奇怪，引起了我的兴趣。

接下来排查问题，直接curl调用B的服务，发现能够正确的返回json数据，数据也是完全的，这就说明A服务HTTP调用B的时候，拿到返回的数据解析成对象出了问题。说明MyObjectA对象的JSON序列化的一部分除了问题。

于是在A服务中加入代码测试序列化：
```java
MyObjectA a;
// ...
// 中间是从数据库查出来的a
// ...
// JSONUtils是com.alibaba.druid.support.json的，随便找了一个
String string = JSONUtils.toString(a); // 此行报错 
MyObjectA parseA = (MyObjectA) JSONUtils.parse(string);
```

执行的时候发现报错，然后报错的原因是
```java
not support type : *.*.*MyObjectC
```

然后检查MyObjectA里面都没有MyObjectC的东西啊
接着检查MyObjectA的来源，MyObjectA从MyBatis里面查出来的，然后查看XML。
```xml
 <resultMap id="baseResult" type="*.*.*MyObjectA">
        <id column="TITLE" jdbcType="VARCHAR" property="Title"/>
        <result column="OTHER" jdbcType="VARCHAR" property="Other"/>
        <collection property="list" column="TITLE" select="*.*.*.MyObjectCMapper.select"/>
    </resultMap>
```

注意到 collection返回的MyObjectCMapper的select方法，返回的是List<MyObjectC>而不是List<MyObjectC>
之后我试了一下，确实能够set上，如果MyObjectC和MyObjectB之间有相同的字段的时候。

这样可以解释成MyBatis返回的MyObjectA中的list其实是List<MyObjectC>而不是List<MyObjectB>，所以在序列化的时候报错了。
但是为什么直接调用B的时候，同样的序列化就没报错呢。

TODO：
1. 深入MyBatis转换对象时候的映射策略
2. 查看SpringBoot为什么直接返回Json序列化没报错
3. 不同的Json包对上述情况进行测试

写得有点乱，orz，后期整理。