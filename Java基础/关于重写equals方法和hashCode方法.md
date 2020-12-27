# 关于重写equals方法和hashCode方法

## 一.equals

equals方法和hashCode方法原本是Object中的方法，在Object类中equals方法实现为确定两个对象引用是否相等，通俗的来讲就是根据两个对象的内存地址来判断，下面用具体的代码说明：

```java
public class Testequals {
    public static void main(String[] args) {
        String a = "hello";
        String b = "hello";
        test t1 = new test(1);
        test t2 = new test(1);
        System.out.println(a.equals(b));//true
        System.out.println(t1.equals(t2));//false
    }
}
class test{
    private int id;

    public test(int id) {
        this.id = id;
    }
    
}
```

这里因为test并没有重写equals方法，所以调用的是Object中的equals，a和b同时指向了字符串"hello"对象，所以结果是true，而t1和t2指向的是不同的test对象，所以结果是false，当我们重写了equals方法后，便可以按照我们理解的那样去判断(id一样，对象也就一样).

```java
    @Override
    public boolean equals(Object otherObject) {
        if (this == otherObject) return true;
        if (otherObject == null || getClass() != otherObject.getClass()) return false;
        test other = (test) otherObject;
        return id == other.id;
    }
```

这里解释一下怎么重写equals方法:

1. 显示参数命名为otherObject，稍后需要将它转换为另一个名为other的变量

2. 检测this与otherObject是否相等:

	`if(this == otherObject) return true;`

	这条语句只是一个优化，实际上这是一个经常采用的形式，目的是为了减小开销

3. 检测otherObject是否为null，如果为null，返回false，注意这个检测是必须的

	`if (otherObject == null) return false;`

4. 比较this与otherObject的类，如果equals的语义可以在子类中改变就用getClass,如果所有的子类都有相同的相等性语义，可以使用instanceof:

	`if (getClass() != otherObject.getClass()) return false` and `if (!(otherObject instanceof ClassName)) return false`

5. 现在根据相等性概念的要求来比较字段.使用`==`来比较基本类型字段，使用`Objects.equals`比较对象字段

6. 如果在子类中重新定义equals方法，就要在其中包含一个`super.equals(other)`调用

提示：

1. 为什么不直接使用一般类的equals方法`a.equals(b)`？这是因为`Object.equals(a,b)`可以处理对象引用为null的情况,如果a和b两个都为null,那么该方法将返回`true`，如果其中一个参数为null则返回false,如果两个参数都不为null时，这个方法将会调用`a.equals(b)`
2. 再重写equals方法的时候，注意参数一定要声明为Object，否则覆写的将不会是Object中的equals方法.

扩展：

这里有关于`==`判断相等性的相关问题的说明:

```java
Integer x1 = 100;
Integer y1 = 100;
Integer x2 = 1000;
Integer y2 = 1000;
System.out.println(x1 == y1);//true
System.out.println(x2 == y2);//false
```

这个问题的关键在于在java.lang.Integer类下有一个私有类:`IntegerCache`,这个类会缓存-128~127之间的所有的整数对象。在`Integer x1 = 100`的时候，在内部实际发生的是：`Integer x1 = Integer.valueOf(100)`，现在去查看`valueOf()`方法的源码：

```java
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
```

如果值i在-128~127之间就会从高速缓存返回实例.所以x1和y1指向的是同一个对象。具体这样做的原因是因为在此范围内的“小”的整数使用率比大整数使用率高很多,为了减少潜在的内存占有，so……

## 二.hashCode

hashCode是通过具体的哈希算法得出的一个int值，一般来说不同的对象的hashCode一般不同，但难免也有哈希冲突的时候,在Object中定义的hashCode方法通过对象的内存地址得出。具体用到hashCode或者说是要重写hashCode方法的场景更多的是在使用散列表的时候.如果在重写了equals方法后，该类有可能会被用在散列表中就必须重写hashCode方法.

重写hashCode有多种方式，关键是要保证`x.equals(y)`返回true时，`x.hashCode()`与`y.hashCode()`必须返回相同的值。如果对象可能为null就使用`Object.hash(Object... objects)`方法，该方法可以返回提供的所有的对象的散列码的组合而成的散列码，简而言之就是可以多个对象一起.当然也可以使用单一的`a.hashCode()`方法。

## 三.重要性

给段代码吧，直接点：

```java
import java.util.HashMap;
import java.util.Objects;

public class WithoutHashCode {
    public static void main(String[] args) {
        Key k1 = new Key(1);
        Key k2 = new Key(1);
        HashMap<Key, String> hm = new HashMap<>();
        hm.put(k1,"Key with id is 1");
        System.out.println(hm.get(k2));
        System.out.println(k1.hashCode());//可见k1和k2的hashcode已经一样了，
        System.out.println(k2.hashCode());
    }

}
class Key{
    private Integer id;
    
    public Integer getId() {
        return id;
    }

    public Key(Integer id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Key key = (Key) o;
        return Objects.equals(id, key.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

当你没有重写hashCode方法和equals方法时，你虽然放的是k1，但想用明明值和k1相同的k2去取map里面的值时，出现的却是null,这显然是违法我们常识的,而如果你只重写了hashCode方法，虽然此时k1和k2在map上的同一位置，但是输出仍为null,这是因为hashMap是用链地址法来处理冲突，也就是说，在100号位置上，有可能存在着多个用链表形式存储的对象。它们通过hashCode方法返回的hash值都是100。当我们通过k2的hashCode到100号位置查找时，确实会得到k1。但k1有可能仅仅是和k2具有相同的hash值，但未必和k2相等（k1和k2两把钥匙未必能开同一扇门），这个时候，就需要调用Key对象的equals方法来判断两者是否相等了。

如果要在HashMap的“键”部分存放自定义的对象，一定要在这个对象里用自己的equals和hashCode方法来覆盖Object里的同名方法。

