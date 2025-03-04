---
title: 【Java】Java泛型与通配符详解
date: 2025-03-04 12:00:00
tags:
  - Java
  - 泛型
  - 通配符
categories: 
  - Java
  - 技术笔记
  - Java基础
abbrlink: 9f8e706f
---

# Java泛型与通配符详解

在Java编程中，泛型是一个强大的特性，它提供了编译时类型安全检查机制，允许程序员定义类型安全的数据结构。本文将深入探讨Java泛型的概念、用法以及通配符的使用方式。

## 泛型基础

### 什么是泛型？

泛型是Java 5引入的一个重要特性，它允许在定义类、接口和方法时使用类型参数。通过泛型，我们可以：

1. 实现类型安全的集合
2. 消除类型转换
3. 实现通用算法

### 泛型类的定义

```java
public class Box<T> {
    private T content;
    
    public void set(T content) {
        this.content = content;
    }
    
    public T get() {
        return content;
    }
}
```

使用示例：

```java
Box<String> stringBox = new Box<>();
stringBox.set("Hello Generics");
String content = stringBox.get(); // 无需类型转换
```

## 泛型方法

### 定义泛型方法

```java
public class Util {
    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
    
    public static <T extends Comparable<T>> T findMax(T[] array) {
        if (array == null || array.length == 0) return null;
        
        T max = array[0];
        for (int i = 1; i < array.length; i++) {
            if (array[i].compareTo(max) > 0) {
                max = array[i];
            }
        }
        return max;
    }
}
```

## 泛型接口

```java
public interface Generator<T> {
    T next();
}

// 实现泛型接口
public class NumberGenerator implements Generator<Integer> {
    private int current = 0;
    
    @Override
    public Integer next() {
        return current++;
    }
}
```

## 通配符

### 上界通配符（extends）

```java
public void processNumbers(List<? extends Number> numbers) {
    for (Number number : numbers) {
        System.out.println(number.doubleValue());
    }
}
```

### 下界通配符（super）

```java
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    // list.add("3"); // 编译错误
}
```

### 无界通配符

```java
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}
```

## 类型擦除

泛型信息在编译后会被擦除，这是Java泛型的一个重要特性：

```java
Box<String> stringBox = new Box<>();
Box<Integer> intBox = new Box<>();

// 在运行时，两者的类型是相同的
System.out.println(stringBox.getClass() == intBox.getClass()); // true
```

### 类型擦除的影响

1. 不能创建泛型数组
2. 不能用基本类型实例化泛型类
3. 不能捕获泛型类型的异常

## 最佳实践

### 1. 优先使用泛型集合

```java
// 推荐
List<String> list = new ArrayList<>();

// 不推荐
List list = new ArrayList(); // 原始类型
```

### 2. 合理使用通配符

```java
// 如果需要从集合中读取，使用 extends
public void readOnly(List<? extends Number> numbers) {
    Number first = numbers.get(0);
}

// 如果需要向集合中写入，使用 super
public void writeOnly(List<? super Integer> numbers) {
    numbers.add(42);
}
```

### 3. 泛型方法的类型推断

```java
public class Pair<K, V> {
    public static <K, V> Pair<K, V> of(K key, V value) {
        return new Pair<>(key, value);
    }
}

// 使用类型推断
Pair<String, Integer> pair = Pair.of("key", 1); // 无需显式指定类型
```

## 常见问题与解决方案

### 1. 泛型数组问题

```java
// 不能直接创建泛型数组
// T[] array = new T[10]; // 编译错误

// 解决方案
public class GenericArray<T> {
    private Object[] array;
    
    @SuppressWarnings("unchecked")
    public T get(int index) {
        return (T) array[index];
    }
    
    public void set(int index, T item) {
        array[index] = item;
    }
}
```

### 2. 类型安全的异构容器

```java
public class TypeSafeMap {
    private Map<Class<?>, Object> map = new HashMap<>();
    
    public <T> void put(Class<T> type, T instance) {
        map.put(type, instance);
    }
    
    @SuppressWarnings("unchecked")
    public <T> T get(Class<T> type) {
        return type.cast(map.get(type));
    }
}
```

## 总结

Java泛型是一个强大的特性，它提供了编译时类型安全性，使代码更加健壮和可维护。通过合理使用泛型和通配符，我们可以编写更加灵活和类型安全的代码。虽然有一些限制（如类型擦除），但通过遵循最佳实践和正确的使用模式，我们可以充分发挥泛型的优势。

---

希望这篇文章能帮助您更好地理解和使用Java泛型与通配符。如果您有任何问题，欢迎在评论区讨论！