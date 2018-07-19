这节主要记录我在使用Spring事务过程中遇到过的坑。
# 没有抛出异常
```java
@Transactional
public void addUser(User user) {
    try {
        userMapper.addUser(user);
        int i = 1/0;
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
上面代码user是能够添加成功的，因为不抛出异常，spring是没有办法捕获的，所以就不能执行回滚。
# this调用声明REQUIRES_NEW的事务，是不会重新创建事务的
```java
@Transactional
public void addUser(User user) {
    this.update();
}

@Transactional(propagation = REQUIRES_NEW)
public void update() {

}
```
update()方法不会重新创建一个事务的，因为this指向的是目标方法，而要使用事务必须使用代理对象才能生效。为了解决这个问题，可以通过下面方式：
```java
@EnableAspectJAutoProxy(exposeProxy = true)
```
然后将this.update()修改为：
```java
((UserService)AopContext.currentProxy()).update();
```
AopContext.currentProxy()是在创建代理的时候，通过下面代码赋值的
```java
if (this.advised.exposeProxy) {
    // Make invocation available if necessary.
    oldProxy = AopContext.setCurrentProxy(proxy);
    setProxyContext = true;
}
```
# Transaction rolled back because it has been marked as rollback-only异常
这个是一个同事问我，他想做的是在a方法中调用b方法，但是b失败，不影响a的事务，代码如下:
```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;
    @Autowired
    private EmployeeService employeeService;

    @Transactional
    public void addUser(User user) {
        userMapper.addUser(user);
        try {
            employeeService.addEmployee();
        } catch (Exception e) {
            //e.printStackTrace();
        }
    }
}
```
```java
@Service
public class EmployeeService {

    @Transactional
    public void addEmployee() {
        int i = 1/0;
    }
}
```
这个是因为在addEmployee中抛出异常时，在回滚时如果不是一个新的事务就会把事务状态设置成readOnly，但是在addUser中又没有把异常抛出，所以会提交事务，在提交的时候就会校验事务是否设置成readOnly，就抛出UnexpectedRollbackException异常。解决办法就是让addEmployee新启一个事务@Transactional(propagation = Propagation.REQUIRES_NEW)