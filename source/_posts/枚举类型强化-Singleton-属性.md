---
title: 枚举类型强化 Singleton 属性
date: 2017-05-03 08:28:45
tags:
- Java
categories:
- 技术
---
**Singleton** 指仅仅被 **实例化一次** 的类，通常被用来代表本质上唯一的系统组件，如窗口管理器或文件系统。

使类称为 Singleton 会使它的客户端测试变得困难，因为无法给 Singleton 替换模拟实现。除非它实现一个充当其类型的接口。


<!--more-->

## Java 1.5 之前

在 Java 1.5 版本发行前，实现 Singleton 有两种方法，将构造器保持私有，并导出公有的静态成员。

### 1. 公有静态成员是个 final 域

```Java
public class TestSingleton {
  public static final TestSingleton INSTANCE = new TestSingleton();
  private TestSingleton() { ... }

  //public method
}
```

私有构造器仅被调用一次，用于实例化公有的静态 final 域实例。

### 2. 公有静态成员是个静态工厂方法

```Java
public class TestSingleton {
  private static final TestSingleton INSTANCE = new TestSingleton();
  private TestSingleton() { ... }
  public static TestSingleton getInstance() { return INSTANCE; }

  //public method
}
```

对于静态方法的所有调用，返回同一个对象引用。

**公有域** 的好处在于，类的成员声明清楚的表明这个类是一个 Singleton。

**工厂方法** 的优势在于提供了 **灵活性**。可以在不改变其 API 的前提下改变类是否应该为 Singleton。第二个优势与泛型有关。

***需要提醒一点：享有特权的客户端可以借助 AccesssibleObject.setAccessible 方法，通过反射机制调用私有构造器。要抵御这种攻击，可以修改私有构造器，让它被要求创建第二个实例时抛出异常。***

以上方法实现的 Singleton 如要变成 **可序列化**，仅在声明中添加 `implement Serializable` 是不够的，必须声明所有实例域都 **是瞬时（transient）** 的，并提供readResolve 方法。否则反序列化时会创建新的实例。

```Java
private Object readResolve() {
  return Instance;
}
```

## 实现 Singleton 的最佳方法

从 Java 1.5 发行版起，实现 Singleton 只需编写一个包含单个元素的枚举类型：

```Java
public enum TestSingleton {
  INSTANCE;
  
  //public method
}
```

### 优势

1. 简洁
2. 无偿提供序列化机制
3. 绝对防止多次实例化，即便面对反射攻击
