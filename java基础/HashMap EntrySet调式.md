昨天有朋友问我，IDEA调式HashMap，在调式下面代码的时候，entrySet一开始就有值了，但是没有找到给entrySet赋值的地方。
```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```
我写了段代码验证，发现确实如此，开始我以为是jdk1.8的原因，相同代码放到1.6后还是如此。
```java
public class HashMapEntrySetTest {
    public static void main(String[] args) {
        Map<String, String> map = new HashMap<>();
        map.put("aaa","aaaa");
        map.entrySet();
    }
}
```
调式结果:
![IDEA调式](https://upload-images.jianshu.io/upload_images/10236819-ac844e8d4e15118a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我在[https://blog.csdn.net/lwj_zeal/article/details/72899934](https://blog.csdn.net/lwj_zeal/article/details/72899934)找到了答案，IDEA在调式的时候会调用对象的toString方法。验证代码如下:
```java
public class DebugTest {
    @Override
    public String toString() {
        System.out.println("======debug======");
        return super.toString();
    }

    public static void main(String[] args) {
        System.out.println("======start======");
        DebugTest debugTest = new DebugTest();
    }
}
```
不加debug输出结果是：======start======
在System.out.println("======start======");行加断点，单步执行时会多输出======debug======。也就是说IDEA在单步执行时会调用对象的toString方法。
现在再来看HashMap的toString方法，HashMap没有重写toString，而是在它的父类AbstractMap重写了。
```java
public String toString() {
    Iterator<Entry<K,V>> i = entrySet().iterator();
    if (! i.hasNext())
        return "{}";

    StringBuilder sb = new StringBuilder();
    sb.append('{');
    for (;;) {
        Entry<K,V> e = i.next();
        K key = e.getKey();
        V value = e.getValue();
        sb.append(key   == this ? "(this Map)" : key);
        sb.append('=');
        sb.append(value == this ? "(this Map)" : value);
        if (! i.hasNext())
            return sb.append('}').toString();
        sb.append(',').append(' ');
    }
}
```
从上面代码可以看到，在toString中有调用entrySet()方法，entrySet第一次赋值是在这里。
为了再次确认是IDEA的问题，我又在eclipse中调式entrySet方法，发现第一次进去的时候是为空。
![eclipse调式](https://upload-images.jianshu.io/upload_images/10236819-6fdb31c2d0f0e16d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
