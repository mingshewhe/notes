# 使用
cglib使用需要实现MethodInterceptor接口，与JDK动态代理不同的是，cglib不需要目标类实现接口。
```java
public class CglibDynamicProxyTest {

    static class Person {
        public void say() {
            System.out.println("hello world");
        }
    }

    static class CglibProxy implements MethodInterceptor {

        Object target;

        public Object getProxy(Object target) {
            this.target = target;
            final Enhancer enhancer = new Enhancer();
            enhancer.setUseFactory(false);
            enhancer.setSuperclass(target.getClass());
            enhancer.setCallback(this);
            return enhancer.create();
        }

        @Override
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            System.out.println("say before");
            methodProxy.invokeSuper(o, objects);
            System.out.println("say after");
            return null;
        }
    }

    public static void main(String[] args) {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "G:\\gitRepository\\demo");
        CglibProxy cglibProxy = new CglibProxy();
        Person person = (Person) cglibProxy.getProxy(new Person());
        person.say();
    }
}
```
# 原理
查看cglib的原理，需在运行时添加
```java
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY,"G:\\gitRepository\\demo");
```
会在配置的路径下会生成三个字节码文件，其中有两个继承FastClass类的是索引文件，还有一个继承person对象，是我们要研究的。
```java
public class CglibDynamicProxyTest$Person$$EnhancerByCGLIB$$681783b8 extends Person {
    private boolean CGLIB$BOUND;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static final Method CGLIB$say$0$Method;
    private static final MethodProxy CGLIB$say$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$finalize$1$Method;
    private static final MethodProxy CGLIB$finalize$1$Proxy;
    private static final Method CGLIB$equals$2$Method;
    private static final MethodProxy CGLIB$equals$2$Proxy;
    private static final Method CGLIB$toString$3$Method;
    private static final MethodProxy CGLIB$toString$3$Proxy;
    private static final Method CGLIB$hashCode$4$Method;
    private static final MethodProxy CGLIB$hashCode$4$Proxy;
    private static final Method CGLIB$clone$5$Method;
    private static final MethodProxy CGLIB$clone$5$Proxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("com.ming.test.CglibDynamicProxyTest$Person$$EnhancerByCGLIB$$681783b8");
        Class var1;
        CGLIB$say$0$Method = ReflectUtils.findMethods(new String[]{"say", "()V"}, (var1 = Class.forName("com.ming.test.CglibDynamicProxyTest$Person")).getDeclaredMethods())[0];
        CGLIB$say$0$Proxy = MethodProxy.create(var1, var0, "()V", "say", "CGLIB$say$0");
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"finalize", "()V", "equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$finalize$1$Method = var10000[0];
        CGLIB$finalize$1$Proxy = MethodProxy.create(var1, var0, "()V", "finalize", "CGLIB$finalize$1");
        CGLIB$equals$2$Method = var10000[1];
        CGLIB$equals$2$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$2");
        CGLIB$toString$3$Method = var10000[2];
        CGLIB$toString$3$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$3");
        CGLIB$hashCode$4$Method = var10000[3];
        CGLIB$hashCode$4$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$4");
        CGLIB$clone$5$Method = var10000[4];
        CGLIB$clone$5$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$5");
    }

    final void CGLIB$say$0() {
        super.say();
    }

    public final void say() {
        //1. 得到MethodInterceptor，我们实现的接口，通过enhancer.setCallback(this)设置的。
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }
        //2. 调用代理的逻辑
        if (var10000 != null) {
            var10000.intercept(this, CGLIB$say$0$Method, CGLIB$emptyArgs, CGLIB$say$0$Proxy);
        } else {
            super.say();
        }
    }
    //..省略一些方法
    private static final void CGLIB$BIND_CALLBACKS(Object var0) {
        CglibDynamicProxyTest$Person$$EnhancerByCGLIB$$681783b8 var1 = (CglibDynamicProxyTest$Person$$EnhancerByCGLIB$$681783b8)var0;
        if (!var1.CGLIB$BOUND) {
            var1.CGLIB$BOUND = true;
            Object var10000 = CGLIB$THREAD_CALLBACKS.get();
            if (var10000 == null) {
                var10000 = CGLIB$STATIC_CALLBACKS;
                if (CGLIB$STATIC_CALLBACKS == null) {
                    return;
                }
            }

            var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
        }

    }

    static {
        CGLIB$STATICHOOK1();
    }
}
```
代理类继承了目标类，并重写目标类方法。代理的大概结构包括三大块：
1. 静态字段:目标类中所有方法(包括父类中的方法，Object除外)以及Object中finalize、equals、toString、hashCode、clone方法,都对应一个Method和MethodProxy静态变量。
2. 静态代码块:静态代码块中调用CGLIB$STATICHOOK1()静态方法，这个方法主要是通过反射初始化Method静态变量，并初始化MethodProxy静态变量。Method表示目标类的方法,MethodProxy表示代理类中方法。
3. 目标类中的方法：代理类中有两个方法，一个是重写目标类方法，重写的逻辑也比较简单，得到MethodInterceptor对象，然后调用intercept方法，intercept方法是我们实现的方法。还有一个CGLIB$say$0方法，这个是给MethodProxy调用，实现逻辑直接调用目标类方法，在调用methodProxy.invokeSuper(o, objects)时会调用这个方法。

# 遇到的坑
我在编码的时候，写过如下的代码，结果发生了死循环。
```java
@Override
public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
    System.out.println("say before");
    method.invoke(o, objects);
    System.out.println("say after");
    return null;
}
```
我们从反编译的代码中var10000.intercept(this, CGLIB\$say\$0\$Method, CGLIB\$emptyArgs, CGLIB\$say\$0\$Proxy)中可以看出，o表示的是代理类，method表示的是目标类中方法，但是因为代理类重写了目标类中方法，所以还是调用代理类中的方法。代理的方法又会调用intercept，就形成了死循环。

# jdk动态代理类与cglib代理区别
1. jdk动态代理目标类必须实现一个接口，而cglib不需要。是因为jdk代理会继承Proxy类，而java又是单继承。
2. jdk代理类与目标类实现相同的接口，cglib代理直接继承目标类。
3. jdk和cglib都是把实现逻辑代理给接口。jdk代理给InvocationHandler接口，cglib代理给MethodInterceptor接口。
4. cglib一个目标类方法会生成两个代理方法，一个重写目标方法，并实现代理逻辑，还有一个直接调用目标类方法。
