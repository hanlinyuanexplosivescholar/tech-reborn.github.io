1. 业务方法内部调用
这是最常见也最容易被忽视的场景。假设你的服务中有一个方法 methodA 被 @Transactional 注解，methodA 内部又调用了 methodB（同样被 @Transactional 注解）。

```
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional // 事务会生效
    public void methodA() {
        // ... some operations ...
        methodB(); // 内部调用
    }

    @Transactional // 这里的事务不会生效
    public void methodB() {
        // ... some operations ...
    }
}
```


失效原因： Spring 事务是基于 AOP（面向切面编程）实现的。当你调用 methodA 时，Spring 会为它创建一个代理对象。事务的开启、提交、回滚等逻辑都是通过这个代理对象来执行的。然而，当 methodA 在其内部直接调用 methodB 时，methodB 的调用是通过 this 关键字进行的，而不是通过 Spring 的代理对象。这意味着 methodB 的事务注解不会被 Spring AOP 拦截到，因此其事务性不会生效。




2. 事务方法的访问修饰符不是 public

失效原因： Spring AOP 默认使用的是 JDK 动态代理或 CGLIB 代理。

JDK 动态代理： 只能代理接口方法，如果你的类没有实现接口，就无法使用 JDK 动态代理。

CGLIB 代理： 可以代理类，但它通过继承目标类并重写其方法来实现代理。非 public 方法（private, protected, 默认/包私有）在子类中无法被直接重写（或在代理中无法被拦截），因此 @Transactional 注解对它们不起作用。

解决方案： 确保所有需要事务管理的方法都是 public 方法。


3. 异常被捕获但未抛出

```
@Service
public class UserService {

    @Transactional
    public void methodA() {
        try {
            // ... database operation that might throw an exception ...
            throw new RuntimeException("Something went wrong!");
        } catch (RuntimeException e) {
            // 异常被捕获但未再次抛出
            System.out.println("Exception caught: " + e.getMessage());
            // 事务不会回滚
        }
    }
}

```


失效原因： Spring 事务管理器默认只对**未捕获的运行时异常（RuntimeException 或其子类）**或 Error 进行回滚。如果你在事务方法内部捕获了异常，但没有再次将其抛出，Spring 事务管理器就不会感知到异常的发生，从而不会触发事务回滚，导致事务提交。

4. 事务传播行为设置不当
@Transactional 注解的 propagation 属性定义了事务的传播行为。如果设置不当，可能会导致事务没有按预期生效。
```
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional // 外部方法有事务
    public void outerMethod() {
        // ...
        innerMethod();
    }

    @Transactional(propagation = Propagation.NOT_SUPPORTED) // 这里的事务不会生效
    public void innerMethod() {
        // ...
    }
}
```


失效原因： Propagation.NOT_SUPPORTED 意味着当前方法不应该在事务中运行。如果当前存在事务（outerMethod 的事务），它会被挂起。因此，innerMethod 会在无事务的上下文中执行。类似地，Propagation.NEVER 也会导致事务失效甚至抛异常。

解决方案： 根据业务需求选择合适的传播行为：

REQUIRED (默认)： 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是最常用的。

REQUIRES_NEW： 总是创建一个新事务，如果当前存在事务，则挂起当前事务。

NESTED： 如果当前存在事务，则在嵌套事务中执行；如果当前没有事务，则创建一个新事务。嵌套事务与外部事务有逻辑上的父子关系，可以独立回滚。


5. 数据库不支持事务或未开启事务
失效原因： 某些数据库存储引擎（如 MySQL 的 MyISAM）不支持事务。或者，即使数据库支持事务，但其配置可能未开启事务功能。


6. @Transactional 注解所在的类没有被 Spring 管理
```
public class MyService { // 没有 @Service 或 @Component 注解

    @Transactional
    public void doSomething() {
        // ...
    }
}

```

失效原因： 只有被 Spring IoC 容器管理的 Bean，Spring 才能对其进行 AOP 增强（包括事务代理）。如果你的类没有被 @Service、@Component、@Repository 等注解标记，或者没有在 XML 配置中声明为 Bean，Spring 就不会识别并处理它上面的 @Transactional 注解。

7. 多线程环境下事务失效
```
@Service
public class UserService {

    @Transactional
    public void methodA() {
        new Thread(() -> {
            // 在新线程中执行数据库操作
            // 这里的事务很可能失效，或者无法回滚外部事务
            userRepository.save(new User("Test User"));
        }).start();
    }
}

```


失效原因： Spring 的事务管理器是基于 线程局部变量（ThreadLocal） 来管理事务上下文的。当你在一个 @Transactional 方法内部启动一个新的线程时，新线程拥有自己的线程上下文，它无法访问到父线程的事务上下文。因此，在新线程中执行的数据库操作将不在父线程的事务范围内，其事务控制会失效。