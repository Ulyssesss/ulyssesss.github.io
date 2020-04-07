---
title: Java 内部类
tags:
  - Java
categories:
  - 技术
date: 2017-06-08 11:26:41
---

内部类，即定义在另一个类中的类。



<!-- more -->



### 使用内部类的原因



#### 1. 访问外部类作用域的数据

内部类的对象包含一个隐式引用，指向创建它的外部类对象，所以可以直接访问外部类作用域中的数据，包括私有数据。

当需要访问其他类的数据域时，使用内部类可以免去对所需数据域的显式引用。



#### 2. 对其他类隐藏

内部类的声明是私有的，只有所在外部类的方法才能构造内部类对象。



#### 3. 定义回调函数

匿名内部类在定义回调函数时很便捷，不过 Java8 之后更好的方式是使用 Lambda 表达式。





### 局部内部类

如果内部类的名字仅在某一方法中创建内部类对象时使用了一次，则可以将内部类的定义挪到对应的方法中。

局部内部类不使用 public 或 private 等访问修饰符，作用域被限定在声明该类的代码块中，即使外部类的其他代码也不能访问，对外部代码完全隐藏。

局部内部类还可以访问局部变量，通过此特性可以减少需要显式编写的实例域。不过要访问的局部变量必须为事实上 final 的。

局部内部类示例如下：

```java
public class Outer {

    private int outerNumber;
  
    public Outer(int outerNumber) {
        this.outerNumber = outerNumber;
    }

    public void localInnerAction() {

        int methodNumber = outerNumber * 2;

        class LocalInner {
            private void action() {
                //methodNumber++; 必须为事实上final
                System.out.println("outerNumber: " + outerNumber);
                System.out.println("methodNumber: " + methodNumber);
            }
        }

        LocalInner localInner = new LocalInner();
        localInner.action();
    }

    public static void main(String[] args) {
        Outer outer = new Outer(100);
        outer.localInnerAction();
    }
}
```





### 匿名内部类

如果对某个类仅要创建一个对象，就不必给类命名，这种类称为匿名内部类。

匿名内部类常用于实现某个接口，也可以扩展一个已有的类。语法上只需在构造方法后增加一对大括号，示例如下：

```java
public void anonymousInnerClass() {
    new Runnable() {
        @Override
        public void run() {
            System.out.println("running");
        }
    }.run();
}
```





### 静态内部类

有时使用内部类只是为了把类隐藏在另一个类的内部，并不需要引用外部类对象，这种情况可以将内部类声明为 static ，以便取消产生的外部类隐式引用。

静态内部类的对象，除没有外部类对象的引用外，跟其他内部类没有区别。



