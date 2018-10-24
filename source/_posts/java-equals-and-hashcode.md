---
title: java equals() and hashcode()
date: 2018-08-16 14:34:47
tags:
---
在java中我们经常会重写到的两个方法，一个是`equals()`,还有一个是`hashcode()`.

## 为什么我们需要重写这两个方法？
`equals()` 是用来比较两个对象是否相等，默认的`Object`的 `equals()`方法如下，是判断两个对象的地址是否相等，所以我们需要重写它来实现我们想要的两个对象是否相等。

<!--more-->

```java

public boolean equals(Object obj) {  
    return (this == obj);  
}
  
```
## 那为什么需要重写`hashcode()`呢?

`hashcode()`是用于散列数据的快速存取，如利用HashSet/HashMap/Hashtable类来存储数据时，都是根据存储对象的hashcode值来进行判断是否相同的.
java文档里`equals()`方法下也有写：

```
Note that it is generally necessary to override the hashCode method whenever this method is overridden, so as to maintain the general contract for the hashCode method, which states that equal objects must have equal hash codes.
```

## 如何重写这两个方法？
- equals()
```
- It is reflexive: for any non-null reference value x, x.equals(x) should return true.
- It is symmetric: for any non-null reference values x and y, x.equals(y) should return true if and only if y.equals(x) returns true.
- It is transitive: for any non-null reference values x, y, and z, if x.equals(y) returns true and y.equals(z) returns true, then x.equals(z) should return true.
- It is consistent: for any non-null reference values x and y, multiple invocations of x.equals(y) consistently return true or consistently return false, provided no information used in equals comparisons on the objects is modified.
- For any non-null reference value x, x.equals(null) should return false.
```
对于equals主要就是遵循上面几条原则。

看一下String对象里面的实现方式：

```
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String) anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                            return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }


```

如果对象相同直接返回相等，如果对象不同，按照自己想要的相等逻辑实现。



- hashcode()


```
- Whenever it is invoked on the same object more than once during an execution of a Java application, the hashCode method must consistently return the same integer, provided no information used in equals comparisons on the object is modified. This integer need not remain consistent from one execution of an application to another execution of the same application.
-  If two objects are equal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce the same integer result.
- It is not required that if two objects are unequal according to the equals(java.lang.Object) method, then calling the hashCode method on each of the two objects must produce distinct integer results. However, the programmer should be aware that producing distinct integer results for unequal objects may improve the performance of hash tables.



```

上面是java文档中给出的实现hashcode的规范，总结一下就是

1、如果两个对象equals，Java运行时环境会认为他们的hashcode一定相等。 
2、如果两个对象不equals，他们的hashcode有可能相等。 
3、如果两个对象hashcode相等，他们不一定equals。 
4、如果两个对象hashcode不相等，他们一定不equals。 


同样，我们看一看String里的code
```
// s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;


```

这里肯定有人会问为什么那个常数是31？在`Effective Java `这本书里给过答案：

```
The value 31 was chosen because it is an odd prime. If it were even and the multiplication overflowed, information would be lost, as multiplication by 2 is equivalent to shifting. The advantage of using a prime is less clear, but it is traditional. A nice property of 31 is that the multiplication can be replaced by a shift and a subtraction for better performance: 31 * i == (i << 5) - i. Modern VMs do this sort of optimization automatically.


```

