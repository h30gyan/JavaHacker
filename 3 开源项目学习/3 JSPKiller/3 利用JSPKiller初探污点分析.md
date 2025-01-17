## 1.前言

这里用到的项目是 4ra1n师傅的JSPKiller 进行学习

遇到不认识的字节码指令查：https://blog.csdn.net/u010502101/article/details/120549638



AnalyzerAdapter介绍：https://lsieun.github.io/java-asm-01/analyzer-adapter-intro.html

---



## 污点分析定义：

​	污点分析可以抽象成一个**三元组`<sources,sinks,sanitizers>`**的形式，其中，`source` 即污点源，代表直接引入不受信任的数据或者机密数据到系统中；`sink` 即污点汇聚点，代表直接产生安全敏感操作(违反[数据完整性](https://so.csdn.net/so/search?q=数据完整性&spm=1001.2101.3001.7020))或者泄露隐私数据到外界(违反数据保密性)；`sanitizer` 即无害处理，代表通过[数据加密](https://so.csdn.net/so/search?q=数据加密&spm=1001.2101.3001.7020)或者移除危害操作等手段使数据传播不再对软件系统的信息安全产生危害。

​	污点分析就是分析程序中由污点源引入的数据是否能够不经无害处理，而直接传播到污点汇聚点。如果不能，说明系统是信息流安全的；否则，说明系统产生了隐私数据泄露或危险数据操作等安全问题。

​	在漏洞分析中，使用污点分析技术将所感兴趣的数据(通常来自程序的外部输入，**假定所有输入都是危险的**)标记为污点数据，然后通过跟踪和污点数据相关的信息的流向，可以知道它们是否会影响某些关键的程序操作，进而挖掘程序漏洞。即将程序是否存在某种漏洞的问题转化为污点信息是否会被 Sink 点上的操作所使用的问题。

​																													——[污点分析技术](https://blog.csdn.net/weixin_44442186/article/details/123263226)

---

Source : 

-  request.getParameter（对策：对Method污点分析,visitMethodInsn）
- Class.forName 的参数 （对策：对LDC污点分析,visitLdcInsn）
-  rt.getMethod("getRuntime")（对策：对Method污点分析,visitMethodInsn）



##  rt.getMethod("getRuntime") 为什么需要 `ANEWARRAY` 操作(?)

 rt.getMethod有多个参数,所以使用 `ANEWARRAY` 数组进行操作.

`ANEWARRAY`：创建引用类数组对象



`AASTORE`所需参数：

- arrayRef
- 数组下标
- 主体，比如 request param

**注意，执行完 AASTORE的变化：**

**"消耗一个数组做操作实际上另一个数组引用对象也改变了（因为两个数组引用对象引用的同一个对象，可以用C语言的指针去理解），换句话说加入了 cmd 参数"**

---

## 字节码方法调用

**一.方法使用单参数调用:**

如： `rt.getMethod("exec")`解析成字节码如下：

1.先载入class

2.载入方法所需的参数

3.创建数组：

ICONST_0 （设置数组长度为0）
ANEWARRAY java/lang/Class

（**执行完成后，"`ICONST_0` 和  `ANEWARRAY xxx` 会出操作数栈"，返回一个数组引用放在操作数栈顶** ）

4.invoke xx方法（调用方法）

**二.方法使用多参数调用:**

如： `rt.getMethod("exec", String.class)`解析成字节码如下：

1.先载入class

2.载入方法所需的参数 （LDC xxx）

3.创建数组（只不过多参数调用创建多个数组）：

ICONST_0
ANEWARRAY java/lang/Class（**并且"`ICONST_0` 和  `ANEWARRAY xxx` 会出操作数栈"** ）

4.invoke xx方法（调用方法）



## 为什么AASTORE之后要clear呢：

```java
public void visitInsn(int opcode) {
        if (opcode == Opcodes.AASTORE) {
            if (operandStack.get(0).contains("get-param")) {
                logger.info("store request param into array");
                super.visitInsn(opcode);
                // AASTORE 模拟操作之后栈顶是数组引用
                operandStack.get(0).clear();
                // 由于数组中包含了可控变量所以设置 flag
                operandStack.get(0).add("get-param");
                return;
            }
        }
        super.visitInsn(opcode);
    }
```

因为执行完AASTORE就收尾了，所以执行模拟操作栈的clear

---

## JSPKiller待解决的问题：

1.多方法情况下（使用jsppaser解决）

2.runtime利用链污点追踪

3.加解密，包含库和自定义方法

4.考虑多种外部输入
