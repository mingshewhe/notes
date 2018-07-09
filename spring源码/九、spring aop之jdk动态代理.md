# 使用
动态代理有两个对象，目标对象和代理对象。使用JDK动态代理，目标对象必须实现一个接口。
```java
public class JdkDynamicProxyTest {

    interface IPerson {
        void say() ;
    }

    static class Person implements IPerson {
        @Override
        public void say() {
            System.out.println("hello world");
        }
    }

    static class JdkProxy implements InvocationHandler {

        //目标对象
        private Object target;

        //得到代理对象
        public Object getProxy(Object target) {
            this.target = target;
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                    target.getClass().getInterfaces(), this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("say before");
            method.invoke(target, args);
            System.out.println("say after");
            return null;
        }
    }

    public static void main(String[] args) {
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        JdkProxy jdkProxy = new JdkProxy();
        IPerson person = (IPerson) jdkProxy.getProxy(new Person());
        person.say();
    }
}
```
# 原理
要查看JDK动态代理原理，只需要在运行的时候，增加下面一句话：
```java
 System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```
会在项目路径下生成一个$Proxy0.class文件，反编译这个文件，就能够明白JDK动态代理原理。
```java
final class $Proxy0 extends Proxy implements IPerson {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void say() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.ming.aop.IPerson").getMethod("say");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
代理的大概结构包括4部分：
静态字段：被代理的接口所有方法都有一个对应的静态方法变量；
静态块：主要是通过反射初始化静态方法变量；
具体每个代理方法：逻辑都差不多就是 h.invoke，h是Proxy的属性。主要是调用我们定义好的invocatinoHandler逻辑,触发目标对象target上对应的方法;
```java
public class Proxy implements java.io.Serializable {
    protected InvocationHandler h;
}
```
构造函数：从这里传入我们InvocationHandler逻辑；构造函数在newProxyInstance方法中通过反射调用。
```java
public $Proxy0(InvocationHandler var1) throws  {
    super(var1);
}

protected Proxy(InvocationHandler h) {
    Objects.requireNonNull(h);
    this.h = h;
}
```
