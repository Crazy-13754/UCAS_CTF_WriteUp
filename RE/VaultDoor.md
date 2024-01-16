---
title: "VaultDoor"
author: Crazy_13754
date: March 22, 2005
output:
  word_document:
    highlight: "tango"
---
# hello world
```java
public class HelloWorld{ //本文件应当命名为 HelloWorld.java
    public static void main(String[] args) { //这是一个公共的静态的无返回值的主函数
        System.out.println("hello world");   //调用 System 中的方法。
    }
}
```
- 函数分为静态与动态。

1. 静态的属性是类共同拥有的，而动态的属性是类各对象独立拥有的。

2. 静态上内存空间上是固定的，动态中内存空间在各个附属类里面分配。

3. 分配顺序不同，先分配静态对象的空间，继而再对非静态对象。

- Java 静态对象到底有什么好处？

1. 静态对象的数据在全局是唯一的，一改都改。如果你想要处理的东西是整个程序中唯一的，弄成静态是个好方法。 非静态的东西你修改以后只是修改了他自己的数据，但是不会影响其他同类对象的数据。

2. 引用方便。直接用 类名.静态方法名 或者 类名.静态变量名就可引用并且直接可以修改其属性值，不用get和set方法。

3. 保持数据的唯一性。此数据全局都是唯一的，修改他的任何一处地方，在程序所有使用到的地方都将会体现到这些数据的修改。有效减少多余的浪费。

4. static final用来修饰成员变量和成员方法，可简单理解为“全局常量”。对于变量，表示一旦给值就不可修改；对于方法，表示不可覆盖。

原文链接：https://blog.csdn.net/weixin_39915210/article/details/114208140

静态和动态调用的时候有奇怪的问题，虽然这个看起来很明白，但我还是不太明白……

# 关于题目
这八个题高度相似，加密原理均为为字符替换或移位。通常做法为写一个解密函数或者直接用题目的函数得答案。手写也不是不行。

可拓展的知识大抵是恩尼格玛之类。丢几个有点意思的链接

有关古典密码安全性讨论 https://crypto.stackexchange.com/questions/13150/how-cryptographically-secure-was-the-original-ww2-enigma-machine-from-a-modern

以及对恩尼格的历史介绍 https://www.zhihu.com/question/28397034

第八题看起来格外让人容易懵。在这贴一下。

```java
public char switchBits(char c, int p1, int p2) {
    /*
    * Move the bit in position p1 to position p2, and move the bit
    * that was in position p2 to position p1. Precondition: p1 < p2
    */
    char mask1 = (char) (1 << p1); //"<<"是左移操作。这样生成了一个位置 p1 是 1，而其余部分是 0 的“掩码”。
    char mask2 = (char) (1 << p2); //同上
    /* char mask3 = (char)(1<<p1<<p2); mask1++; mask1--; */
    char bit1 = (char) (c & mask1); //将 p1 部分的 0 或 1 “提取出来”
    char bit2 = (char) (c & mask2); // 同上
    /*
    * System.out.println("bit1 " + Integer.toBinaryString(bit1));
    * System.out.println("bit2 " + Integer.toBinaryString(bit2));
    */
    //并没能看懂原题的这个释义在干什么
    char rest = (char) (c & ~(mask1 | mask2)); // 对两个掩码取并后再通过“~”取反，从而清除了 c 中的相应的位置 
    char shift = (char) (p2 - p1); //计算偏移
    char result = (char) ((bit1 << shift) | (bit2 >> shift) | rest); //换位置
    return result; //其实就是换位置的函数，不过写的很复杂。再通过前面函数调用的多次取换，还是古典密码的范畴。
}
```