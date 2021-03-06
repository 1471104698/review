# Spring 事务管理器



## 1、注解开启 Spring 事务

SpringBoot 开启事务，添加 @EnableTransactionManagement：

```java
@SpringBootApplication
@EnableTransactionManagement(proxyTargetClass=true)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Spring 事务是借助 AOP 来完成的， @EnableTransactionManagement 注解如下

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
    //是否使用 CGLIB 代理
    boolean proxyTargetClass() default false;
    //事务通知模式，默认 proxy 代理模式（正是因为代理模式，所以自调用时事务传播机制不起作用）
    AdviceMode mode() default AdviceMode.PROXY;
}
```



它 跟 SpringBoot 的自动装配一样，Import 了 一个 TransactionManagementConfigurationSelector，它是一个 ImportSelector 类，在 selectImports() 中会返回一个 用于事务管理的 类，在扫描时注入到 IOC 中

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
                //默认是代理模式，所以进入该方法体
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}

	private String determineTransactionAspectClass() {
		return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
				TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
				TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
	}

```



## 2、PlatformTransactionManager（事务管理器）

在 事务管理 中，有一个很重要的接口 PlatformTransactionManager，它是一个 事务管理器 TransactionManager

有三个主要的方法：

- getTransaction()
- commit()
- rollback()

PlatformTransactionManager 接口的继承关系图如下：它实际上有多个实现类，不过主要的实现类是 DataSourceTransactionManager

*![image.png](https://pic.leetcode-cn.com/1604820784-BpmpdX-image.png)*





[事务部分实现原理](https://blog.csdn.net/weixin_44366439/article/details/89030080 )

## 3、事务 执行流程

存在以下 事务方法：

```java
/*
A
*/
@Service
public class AService {
    @Resource
    UserMapper userMapper;
    @Autowired
    BService bService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void A() {
        Te te = new Te(6, "1", "1");
        userMapper.insertTe(te);
        
        bService.B();
    }
}

/*
B
*/
@Service
class BService {
    @Resource
    UserMapper userMapper;

    @Transactional(propagation = Propagation.NESTED)
    public void B() {
        Te te = new Te(7, "2", "2");
        userMapper.insertTe(te);

        short s = 0;
        s = (short) (s + 1);
    }
}
```



存在以下 Controller 调用：

```java
@Controller
@RequestMapping("/user")
public class UserController {

    @Autowired
    AService aService;

    @GetMapping(value = "/te")
    public String te(){
        aService.A();
        return "不得了";
    }
}
```



我们需要知道，Spring 事务是基于 AOP 的，而 AOP 是基于方法拦截器的，调用方法实际上是调用的 代理对象的方法，然后在代理对象的方法中调用了方法拦截器的拦截方法，事务管理的 AOP 就是在 方法拦截器中执行 事务逻辑

所以 Controller 中注入的 AService 实际上是一个代理对象，调用的 A() 实际上是调用的 代理对象的 A()，在代理对象的 A() 中会调用方法拦截器的 intercept()，在 intercept() 中对方法多层调用，完成 事务管理

*![image.png](https://pic.leetcode-cn.com/1604822796-NAujfh-image.png)*



A() 方法调用时的事务执行流程：

- 调用 AService 代理对象的 A()
- 进入到 CglibAopProxy 对象中定义的一个方法拦截器 DynamicAdvisedInterceptor，调用它的 intercept()，在该方法中会调用 早早就存储进去的 TransactionInterceptor 的 invoke()
- TransactionInterceptor 的 invoke() 中会调用 invokeWithinTransaction()，开始进行事务管理

- invokeWithinTransaction() 会先获取 事务管理器，默认注入的事务管理器为 DataSourceTransactionManager
- 调用 事务管理器的 getTransaction() ，处理 事务传播机制，在特定 事务传播类型下会调用 doBegin()
- doBegin() 判断当前线程是否已经绑定了 Connection（使用 ThreadLocal），如果已经绑定，那么获取，如果没有绑定，那么获取一个新的连接，然后进行绑定

```java
TransactionInterceptor.invoke->
TransactionAspectSupport.invokeWithinTransaction->
TransactionAspectSupport.createTransactionIfNecessary->
AbstractPlatformTransactionManager.getTransaction->
AbstractPlatformTransactionManager.doGetTransaction->
```



事务拦截器的拦截方法，此处是事务的入口，内部深入就是事务的创建：

```java
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
                                         final InvocationCallback invocation) throws Throwable {


    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            //执行事务
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            /*
                发生异常，比如在 方法存在 int i = 1 / 0，
                那么这里捕获异常并且调用 completeTransactionAfterThrowing() 回滚事务，
                并且为了让上层知道发生了什么异常，还需要再将异常抛出 throw ex;
                */
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            cleanupTransactionInfo(txInfo);
        }

        if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            // Set rollback-only in case of Vavr failure matching our rollback rules...
            TransactionStatus status = txInfo.getTransactionStatus();
            if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
            }
        }

        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
}

protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    
    if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
        try {
            //调用 rollback() 回滚事务
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        }
        catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
        }
        catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
        }
    }
}
```





getTransaction() 处理事务传播行为，逻辑如下：

```java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
    throws TransactionException {

    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());
	
    //获取当前线程的事务
    Object transaction = doGetTransaction();
    boolean debugEnabled = logger.isDebugEnabled();
	
    /*
    如果当前线程存在事务，那么进入到 handleExistingTransaction()，该方法是处理已经存在事务的状态
    比如此时处理的是 B()，A() 是存在事务的，那么在这里会进入到该方法中
    */
    if (isExistingTransaction(transaction)) {
        return handleExistingTransaction(def, transaction, debugEnabled);
    }
	/*
		到这里表示当前线程不存在事务，而如果当前方法是 MANDATORY，表示强制要求当前线程存在事务，但是没有，所以抛出异常
	*/
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    /*
    当前方法的事务传播行为是 REQUIRED、REQUIRES_NEW、NESTED 其中一个，那么进入该方法体
    内部调用 doBegin() 获取一个新的 Connection，然后跟当前线程绑定在一起
    */
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
             def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
             def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
       
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                def, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            //调用 doBegin() 获取新连接
            doBegin(transaction, def);
            prepareSynchronization(status, def);
            return status;
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}
```



doGegin() 处理 con 连接的创建 和 绑定，逻辑如下：

```java
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        /*
        		
            	判断当前线程是否存在事务，如果不存在那么 获取一个新的连接 Connection newCon
            	将新的 con 连接设置到 txObject 中，封装为 ConnectionHolder
            */
        if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            //获取一个新的 con 连接
            Connection newCon = obtainDataSource().getConnection();
            //将 con 封装进行 txObject 
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        //获取 txObject 中的 con 连接
        con = txObject.getConnectionHolder().getConnection();

        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        //如果 con 是自动提交的，那么设置为手动提交，方便控制回滚
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            //设置手动提交
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        //setTransactionActive（true）设置该 con 已经存在事务
        txObject.getConnectionHolder().setTransactionActive(true);
		
        /*
       
        如果 txObject 持有的是一个新的 con，那么调用 bindResource() 将 con 和 当前线程绑定在一起，
        方便和其他方法共用一个连接，即意味着共用一个事务
        */
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
```



bindResource() 利用 ThreadLocal 将 con 和 线程 进行绑定，这里传入两个参数：数据库连接池 和 con，这是为了避免一个线程处理多个数据库连接池的情况

方法逻辑如下：

```java
public abstract class TransactionSynchronizationManager {
    //维护了一个 ThreadLocal 变量 resources， 内部元素是 Map
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");
    /*
    	key：con 所在的数据库连接池
    	value: 封装了 con 连接的对象
    */
    public static void bindResource(Object key, ConnectionHolder value) throws IllegalStateException {
        //从 key 中获取数据库连接池
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        /*
        这里的 resources 是一个 ThreadLocal
        private static final ThreadLocal<Map<Object, Object>> resources = new ThreadLocal<>();
        
        resources 中存储的是 Map，即一个线程对应一个 Map，Map 以 数据库连接池为 key，以 con 为 value
        先通过 ThreadLocal 获取线程对应的 map，然后再将 数据库连接池 和 con 进行映射
        这样后续通过 ThreadLocal 获取线程的 map，再通过 数据库连接池 获取唯一对应的 con
        目的是为了 一个线程 处理 多个数据库连接池 产生的多个 con
        */
        Map<Object, Object> map = resources.get();
        if (map == null) {
            map = new HashMap<>();
            resources.set(map);
        }
        //将 数据库连接池 和 con 进行映射
        Object oldValue = map.put(actualKey, value);
        if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
            oldValue = null;
        }
        if (oldValue != null) {
            throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
                                            actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
        }
    }
}
```



handleExistingTransaction() 方法逻辑如下：

```java
/*
进入到方法表示当前线程是存在事务的，比如 A() 调用了 B()，而 A() 是有事务的
*/
private TransactionStatus handleExistingTransaction(
    TransactionDefinition definition, Object transaction, boolean debugEnabled)
    throws TransactionException {

    /*
    	B() 事务传播行为为 NEVER，表示 A 和 B 都不能存在事务，而 A 存在事务，所以抛出异常
    */
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
    }
    /*
		B() 事务传播行为为 NOT_SUPPORTED，表示 B 不能存在事务，而 A 存在事务，所以将 A 的事务挂起，B 以非事务方式执行
	*/
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
    }
    /*
		B() 事务传播行为为 REQUIRES_NEW，那么 B 需要一个新的连接，单独开一个新的事物，所以调用 doBegin() 创建连接
		同时也使用 ThreadLocal 绑定到线程上
	*/
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        //初始化 transaction，使得 ConnectionHondler = null，然后在下面调用 deBegin() 创建一个新的 con
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }
    /*
		B() 事务传播行为为 NESTED，那么 B 开启一个嵌套事务，它默认不会创建一个新的连接，而是和 A 共用一个连接（事务）
		不过使用 Savepoint 机制
	*/
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        //不允许嵌套事务，抛出异常
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                "Transaction manager does not allow nested transactions by default - " +
                "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        //默认使用嵌套事务的方式，那么不会创建一个新的连接
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status =
                prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            //创建 Savepoint
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            //如果不使用嵌套事务的方式，那么创建一个新的 con
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                definition, transaction, true, newSynchronization, debugEnabled, null);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```





## 4、总结（可以看看）

**关于事务 和 con：**

一个 Connction 是一个数据库连接，同时也是 事务的基本单位，不同的连接代表不同的事务

**根据 事务传播行为的不同 来决定 A 和 B 是否是共用一个连接，即是否共用一个事务**

比如 A() 调用 B()，而 B 的事务传播行为 REQUIRES_NEW，那么会单独为 B 创建一个 con 连接然后进行绑定

- 如果 A 和 B 共用一个 con（事务），那么 B 发生异常，那么意味着 这个事务发生异常，那么 A 和 B 都会回滚

- 如果 A 和 B 不是共用一个 con（事务），那么 B 发生异常，如果 A 捕捉了异常，那么 A 不会回滚，如果 A 没有捕捉异常，那么这个异常也会抛出 A，导致 A 事务也发生异常，那么 A 也会回滚



**关于 con（事务）和 线程绑定：**

在 doBegin() 中每次创建一个新的 con 都会调用 bindResource() 将 线程和 con 进行绑定

而 bindResource() 传入的参数有两个：数据源（数据库连接池） 和  con，这是因为一个线程可能同时处理多个数据源的 con，因此需要通过 “数据源 + 线程” 来唯一确定 con

而内部是使用一个 ThreadLocal 来完成绑定的，

```java
private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");
```

ThreadLocal 的元素是一个 Map，key 为数据源，value 为 con

这样我们通过 ThreadLocal + 线程 获取线程对应的 map，然后通过 “数据源” 获取唯一的 con，这样就解决了多数据源的冲突问题



**关于异常回滚**：

由于使用的是 AOP ，Service 是 CGLIB 代码对象，内部使用的拦截器是 事务拦截器，调用 A() 和 B() 时会进入事务拦截器的代理方法，它在内部调用事务管理器进行事务处理，当 A() 调用 B()，而 B() 抛出异常，在不是自调用的情况下，异常的抛出过程是 B() -> 代理方法 -> A()，因此，当在 B() 中没有捕捉异常，而抛出到代理方法处被捕捉到了，那么它在 catch 中会进行事务的回滚，同时为了让 A() 感知异常，所以还会再将异常抛出

```java
/*
	事务拦截器
*/
public class TransactionInterceptor{
    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        //调用下面的方法事务处理
        return invokeWithinTransaction(invocation.getMethod());
    }

    protected Object invokeWithinTransaction(Method method) throws Throwable {
        try {
            //事务处理
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            //回滚事务
            completeTransactionAfterThrowing(txInfo, ex);
            //抛出异常
            throw ex;
        }
    }
}

```



**关于事务传播行为：**

在 getTransaction() 中会调用 doGetTransaction()，内部会获取当前使用的 数据源，然后通过数据源 从 ThreadLocal 中获取 con

- 如果 con 为空，那么表示当前不存在事务，那么继续判断事务传播行为，如果遇到异常的事务传播行为（比如 MANDATORY 这种的），那么就需要抛出异常
- 如果 con 不为空，表示当前存在事务（比如 A() 调用 B()，A 存在事务），那么调用 handleExistingTransaction() 来处理有事务的清空，根据 B 的事务传播行为进行处理，如果 B 的事务传播行为是 NEVER，那么抛出异常，如果 B 的事务传播行为是 REQUIRES_NEW 之类的，那么调用 deBegin() 为 B 创建一个新的 con，然后进行绑定





**关于事务自调用**：

它实际上也是 AOP 自调用，，如果是在事务方法 A 中直接调用内部的另一个事务方法 B，那么 B 的事务传播行为无效，因为不会被方法拦截器拦截，所以 A 和 B 必定是 A 和 B 共用一个事务了，无关 B 的事务传播行为

AOP 自调用举例：

```java
@Service
public class AService {
    @Resource
    UserMapper userMapper;

    @Transactional(propagation = Propagation.REQUIRED)
    public void A() {
        Te te = new Te(6, "1", "1");
        userMapper.insertTe(te);
        //AOP 自调用，所以 B() 的事务传播行为无效
        B();
    }
    @Transactional(propagation = Propagation.NESTED)
    public void B() {
        Te te = new Te(7, "2", "2");
        userMapper.insertTe(te);
    }
}
```

解决方法：

- 注入 ApplicationContext，然后调用 getBean() 获取代理对象，再通过代理对象调用
- 将 B() 划分到另外一个类中，然后在 A 中注入，这样获取到的就是代理对象了

```java
/*
以下是两种方法的集合
*/
@Service
public class AService {
    @Resource
    UserMapper userMapper;
    @Autowired
    BService bService;

    @Autowired
    ApplicationContext applicationContext;

    @Transactional(propagation = Propagation.REQUIRED)
    public void A() {
        Te te = new Te(6, "1", "1");
        userMapper.insertTe(te);
        /*
        	方法一：通过注入 ApplicationContext 获取代理对象
        */
        applicationContext.getBean(AService.class).B();
        /*
        	方法二：通过注入代理对象
        */
        bService.B();
    }
    @Transactional(propagation = Propagation.NESTED)
    public void B() {
        Te te = new Te(7, "2", "2");
        userMapper.insertTe(te);
    }
}

@Service
class BService {
    @Resource
    UserMapper userMapper;

    @Transactional(propagation = Propagation.NESTED)
    public void B() {
        Te te = new Te(7, "2", "2");
        userMapper.insertTe(te);
    }
}
```

