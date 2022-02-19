---
layout: post
category: java
date: 2017-12-06 15:29:03 UTC
title: 【Java基础】Comparator vs Comparable
tags: [排序，比较函数，Comparator，Comparable]
permalink: /java/oop/comparator-comparable
key:
description: 本文主要剖析Comparator以及Comparable的相关区别
keywords: [排序，比较函数，Comparator，Comparable]
---

看到这两个接口,  有过开发经历的肯定都知道它们是用于定义排序逻辑的，每次IDE直接自动提示就完事了，久而久之一些细节的知识点要么就是忽略了，要不就是忘记了。

比如说它们之间有什么不同、Arrays.sort和Collections.sort使用时候，相应的对象是需要实现Comparator还是Comparable、如何进行多字段排序。

本文主要通过相关的梳理，来完善这块知识体系,  以达到碰到相关排序逻辑定义的时候能合理选用相关的接口以及处理各种复杂逻辑，同时也将本文归纳到**容易忽视的Java基础知识**系统。

# 1. Comparator

## 1.1 定义

> A comparison function, which imposes a **total ordering** on some collection of objects.  Comparators can be passed to a sort method (such as {@link Collections#sort(List,Comparator) Collections.sort} or {@link Arrays#sort(Object[],Comparator) Arrays.sort}) to allow precise control over the sort order

用于排序的函数，一般用于集合类对象的排序，常用于`Collections#sort` 和 `Arrays.sort`等。

```java
public interface Comparator<T> {

    int compare(T o1, T o2);
    
}
```

## 1.2 基本使用

需要排序的Java对象继承Comparator接口，然后实现compare逻辑，也就是谁大谁小如何判断。按照Java中的定义，两值比较

+ o1 > o2， 返回1
+ o1 = o2， 返回0
+ o1 < o2,  返回-1

定义Student类，有属性id和score,  针对一系列学生`List<Student>`按照score降序排，id降序排

```java
public class Student implements Comparator<Student> {

    public Integer id;

    private Double score;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public Double getScore() {
        return score;
    }

    public void setScore(Double score) {
        this.score = score;
    }
    
    @Override
    public int compare(Student o1, Student o2) {
        
    }

}
```

### 1.2.1 方法1: 依次进行属性比较

主要借助可比较对象的compareTo方法

```java
public int compare(Student o1, Student o2) {
    int firstFieldResult = o1.getScore().compareTo(o2.getScore());
    // 为0，说明第一个字段相同; 然后基于第二个字段比较
    if (firstFieldResult == 0) {
        return o1.getId().compareTo(o2.getId());
    } else {
        return firstFieldResult;
    }
}
```

注意由于是降序，正常情况是升序，所以需要调用reversed()进行调整。

```java
Student s1 = new Student(1, 120.15);
Student s2 = new Student(10, 130.40);
List<Student> students = Lists.newArrayList(s1, s2);

Collections.sort(students, s1.reversed());
```

从这个地方可以看出，这种实现方式的缺点就是**无法灵活的调整 升序还是降序**， 特别是一个字段升序，一个字段降序，因此我们可以采用下面的方法。

### 1.2.2 方法2: 借助Comparing中的内置方法

```java
public static <T, U extends Comparable<? super U>> Comparator<T> comparing(Function<? super T, ? extends U> keyExtractor) {   
    return (Comparator<T> & Serializable) （(c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2))）;
}
```

+ 注意抽取的比较key, 是需要实现Comparable的，本例中score是double，包装类Double肯定实现了Comparable

```java
Collections.sort(students,
    Comparator.comparing(Student::getScore).reversed().thenComparing(Student::getId)
);
```

这种方式借助lambda表达式，**可以灵活的指定排序顺序和方向**，底层也是基于compareTo实现的。注意sort接口的参数是Comparator。另外一点就是倒序的实现就是通过交换排序字段的顺序就可以了，Collections内部通过**ReverseComparator**去封装的。

```java
private static class ReverseComparator2<T> implements Comparator<T>, Serializable {
    
    final Comparator<T> cmp;

    // 正常的比较顺序是t1, t2
    public int compare(T t1, T t2) {
        return cmp.compare(t2, t1);
    }
    
}
```

# 2. Comparable

## 2.1 定义

> This interface imposes a total ordering on the objects of each class that implements it.  This ordering is referred to as **the class's <i>natural ordering</i>**, and the class's <tt>compareTo</tt> method is referred to as
> its natural comparison method
>
> Lists (and arrays) of objects that implement this interface can be sorted automatically by {@link Collections#sort(List) Collections.sort} (and {@link Arrays#sort(Object[]) Arrays.sort})

从源码层面的解释看，两者并没有什么不同，主要是接口定义的地方。它主要接收待比较的对象，而Comparator需要同时传入待比较的对象。判断大小的规则也是一行，根据结果值是正数、零还是负数。

````java
public interface Comparable<T> {
    
    // 待比较的对象
    int compareTo(T o);
    
}
````

## 2.2 基本使用

```java
public class Student implements Comparable<Student> {
    
    @Override
    public int compareTo(Student student) {
            
    }
    
}
```

# 3.  Arrays.sort vs Collections.sort

在Java中，当我们需要对于**集合对象**进行排序的时候，通常会用到上面两个帮助类中的sort。

+ Arrays.sort(T[], Comparator)

对于这个接口，如果显示指定Comparator了接口，就会利用该逻辑去具体排序。所以，如果对象实现了Comparator接口，并在`compare中定义了逻辑，那么会被覆盖`。

另外Comparator是可以传null, 这个时候**对象需要实现Comparable**，否则就会报`xx cannot be cast to java.lang.Comparable`。

这个从源码中的实现也可以看出，当Comparator为null, 排序的实现变成了CompableTimSort。

```java
public static <T> void sort(T[] a, Comparator<? super T> c) {
    
    if (c == null) {
        ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
    } else {
        TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
    
}
```

+ Collections.sort(List<T> list)

实际上，Collections.sort底层是基于Arrays.sort的实现，只是实现之前将集合转换成了**对象数组**， 排序完成之后对原集合按位置进行赋值。

```java
// import java.util.List
public interface List<E> extends Collection<E> {

    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        
        ListIterator<E> i = this.listIterator();
        
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
    
}
```

# 4. 总结

既然对象比较一般是当前对象和其它对象，所以建议直接 **对象实现Comparable** + **Collections.sort** + **Comparing中的内置方法**来实现对象集合中的排序。

# 5. 参考

> [SO Java中基于Collections帮助类进行多字段排序](https://stackoverflow.com/questions/4258700/collections-sort-with-multiple-fields)