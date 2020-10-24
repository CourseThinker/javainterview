这个问题相信每个学习java的同学都不陌生，作为一个经典的面试题，到现在工作这么多年了我真是认为挺操蛋的一个问题，在网上到现在你仍然可以看见很多讨论这个问题的人，其中不乏工作很多年的人都有争论，我认为还是有必要来说一说这个问题的。



### 从方法区说起

常量池存在于方法区，而方法区在jdk1.7版本前后改变比较大，所以还是先来说说方法区的演变。

在jdk1.7版本之前，常量池存在于方法区，方法区是堆的一个逻辑部分，他有一个名字叫做*非堆*。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixcs99pryj30yg0a03zi.jpg)

1.7版本把字符串常量池放到了堆中。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixcu7plr9j30yo09utag.jpg)

而在1.8以后，则是移除了永久代，方法区概念保留，方法区的实现改为了元空间，常量池还是在堆中。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixcyixgupj30ym09k40w.jpg)

为什么要说方法区的改变，只是为了文章接下来的内容不会由于JDK的版本而产生分歧，接下来内容都会以jdk1.8版本作为基础来讨论。



### String s = new String("xyz");

先来一段代码

``` java
public class Test {
    public static void main(String[] args) {
        String s = "xyz";
    }
}
```

接着我们javac编译代码，然后用javap来反编译，执行javap -c Test

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixdfijdmzj311k0h2gp6.jpg)

从结果来看，ldc命令**在常量池中创建了一个"xyz"的对象，然后把他推至操作数栈顶**，然后astore保存到局部变量，return返回。



接着看第二段面试题中的代码

``` java
public class Test {
    public static void main(String[] args) {
        String s = new String("xyz");
    }
}
```

同样反编译分析

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixdfvnukuj319c0kogrf.jpg)

很明显，我们看到new 创建了一个String对象，同时ldc在常量池中创建了"xyz"字符串对象，之后invokespecial执行构造函数，astore_1赋值，return返回。

通过以上两个例子，可以知道String s = new String("xyz"); 创建了2个对象，而有些答案说的3个对象，则是把引用s也算作一个对象。

还有答案说xyz存在就创建了2个，不存在就创建了3个（包含引用s），再来测试一下。

``` java
public class Test {
    public static void main(String[] args) {
        String s = "xyz";
        String s2 = new String("xyz");
    }
}
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixdl4c8ynj319o0kijx1.jpg)

从这里，很明显的发现这就是我们例子1和2的一个结合，但是注意两次ldc后面的#2，#号代表着索引，说明第二次new String("xyz")的时候并没有重新创建xyz对象。

一些常见的指令助记符含义：

1. nop， 什么都不做。
2. aconst_null，将 null 推送至栈顶。
3. iconst_i(变量数字)，将 int 型 i 推送至栈顶。同理有lconst_0，fconst_0这种你应该知道什么意思了
4. ldc，将 int，float 或 String 型常量值从常量池中推送至栈顶。
5. iload，将指定的 int 型局部变量推送至栈顶。
6. istore，将栈顶 int 型数值存入指定局部变量。同理astore_i代表将栈顶引用型数值存入第i个局部变量。
7. dup，复制栈顶数值并将复制值压入栈顶。
8. invokevirtual，调用实例方法。
9. invokespecial，调用超类构造方法，实例初始化方法，私有方法。
10. invokestatic，调用静态方法。
11. invokeinterface，调用接口方法。
12. invokedynamic，调用动态链接方法。
13. new，创建一个对象，并将其引用值压入栈顶。



### 总结

到底创建了几个对象呢？

1. 如果xyz不存在，引用算对象的话，那就是3个

2. 如果xyz不存在，引用不算对象的话，那就是2个
3. 如果xyz存在，引用算对象的话，那就是2个
4. 如果xyz存在，引用不算对象的话，那就是1个

当然，我认为引用肯定是不算对象的，最终答案应该是1或者2个，这个面试题说实话不应该出现在初级面试题里。

另外，如果你不看不懂反编译后的字节码指令，关注公众号回复关键字111可获取《Java虚拟机规范》电子书。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gixlp4z6d2j31bi0hc0vj.jpg)

