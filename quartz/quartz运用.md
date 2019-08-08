学会了如果使用quartz，也研究quartz的底层源码，写一个需求能够对quartz更融会贯通。我在用spring boot quartz的时候，发现所有的jobDetail和trigger的配置都是一样的，除了定时时间不一样之外，其他的都是一样的，为了减少bena的配置，能不能在写job类的时候，增加一个注解，然后能够自动注入到spring bean中。还有quartz没有记录运行日志，不知道到底运行了没有，运行结果如何，如果增加一张日志表，记录quartz的运行日志。

## 1. 注解注册job

### 1.1 思路

要能够自动注入bean，spring boot是如何做的？我想起了spring boot mybatis的做法，定义一个MapperScan注解，MapperScan注解中有一个@Import(MapperScannerRegistrar.class)， 在Spring boot启动的时候会加载MapperScannerRegistrar注册bean。我们是不是也可以按照这个思路。

1. 定义一个注解EnableQuartzScheduled，也写一个Registrar可以让Spring扫描所有的job，job上定义一个注解，设置trigger的时间。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(QuartzScheduledRegistrar.class)
public @interface EnableQuartzScheduled {

    /**
     * 扫描的基础包
     * @return
     */
    String basePackage() default "";
}
```

2. job上的注解QuartzScheduled

```java
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface QuartzScheduled {
    String cron();
}
```

这里有个问题就是spring如何去扫描包，并加载类的，这块我研究了spring mybatis的源码，是利用ClassPathScanningCandidateComponentProvider类，设置扫描包名，过滤条件(因为不是所有的类都需要加载成bean的)

```java
public class QuartzScheduledRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware, BeanFactoryAware {

    protected final Log logger = LogFactory.getLog(getClass());

    private Environment environment;

    private BeanFactory beanFactory;

    private ClassLoader classLoader;

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {

    }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AnnotationAttributes annoAttrs =  AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableQuartzScheduled.class.getName()));
        for (String pkg : annoAttrs.getStringArray("basePackage")) {
            if (StringUtils.hasText(pkg)) {
                registerQuartzScheduled(pkg);
            }
        }
    }

    private void registerQuartzScheduled(String basePackage) {
        //扫描所有的类
        ClassPathScanningCandidateComponentProvider classScanner = new ClassPathScanningCandidateComponentProvider(false, this.environment);
        AnnotationTypeFilter  annotationTypeFilter = new AnnotationTypeFilter(QuartzScheduled.class);
        classScanner.addIncludeFilter(annotationTypeFilter);
        Set<BeanDefinition> beanDefinitionSet = classScanner.findCandidateComponents(basePackage);
        if (beanDefinitionSet == null || beanDefinitionSet.isEmpty()) {
            return;
        }
        for (BeanDefinition beanDefinition : beanDefinitionSet) {
            if (beanDefinition instanceof AnnotatedBeanDefinition) {
                registerBeans(((AnnotatedBeanDefinition) beanDefinition));
            }
        }
     }

    private void registerBeans(AnnotatedBeanDefinition annotatedBeanDefinition) {
        String className = annotatedBeanDefinition.getBeanClassName();
        //得到job上的注解
        AnnotationMetadata metadata = annotatedBeanDefinition.getMetadata();
        AnnotationAttributes annoAttrs =  AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(QuartzScheduled.class.getName()));
        //获取注解上cron的值
        String cron = annoAttrs.getString("cron");
        if (StringUtils.isEmpty(cron) || !CronExpression.isValidExpression(cron)) {
            logger.error(className + " @QuartzScheduled cron error");
            return;
        }
        try {
            Class jobClass = ClassUtils.forName(annotatedBeanDefinition.getMetadata().getClassName(), classLoader);
            //生成jobDetail
            JobDetail jobDetail = JobBuilder.newJob(jobClass).storeDurably().withIdentity(jobClass.getSimpleName() + "_jobDetail").build();
            //生成trigger
            CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule(cron).withMisfireHandlingInstructionIgnoreMisfires();
            CronTrigger cronTrigger = TriggerBuilder.newTrigger().forJob(jobDetail).withIdentity(jobClass.getSimpleName() + "_trigger").withSchedule(cronScheduleBuilder).build();
            //注册bean
            ((DefaultListableBeanFactory) this.beanFactory).registerSingleton(jobClass.getSimpleName() + "_jobDetail", jobDetail);
            ((DefaultListableBeanFactory) this.beanFactory).registerSingleton(jobClass.getSimpleName() + "_trigger", cronTrigger);
        } catch (Exception e) {
            logger.error("register" + className + " error", e);
        }

    }
}
```

## 2. 记录日志

当初想的就是能够把quartz运行的日志记录到数据库，能够在界面上展示出来，这里代码没写全，只是写了个思路。在阅读源码后，发现quartz在运行job前后都会通知监听器，如果我们加一个job的监听器，是不是就可以实现日志记录了。在这遇到两个问题。

1.  quartz如何写监听器
2. spring boot quartz是如何增加监听器的

### 2.1 quartz编写job监听器

开源框架，为了扩展性，基本都会定义一个接口，直接继承接口就能够使用了,quartz也不例外，我们实现JobListener就可以了。记录日志就要操作数据库。数据源使用和quartz使用同一个，@QuartzDataSource标注quartz的数据源。

表结构：

```sql

```

```java
@Component
public class QuartzJobLogListener implements JobListener {
    protected final Log logger = LogFactory.getLog(getClass());

    @Autowired
    @QuartzDataSource
    private DataSource dataSource;
    private static final String INSERT_SQL = "insert into QRTZ_LOG(JOB_NAME, status, errMsg, createdate, modifytime) " +
            "values(?, ?, ?, now() ,now())";

    @Override
    public String getName() {
        return "QuartzJobLogListener";
    }

    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        //logger.info(context.getJobDetail().getKey() + "jobToBeExecuted...");
    }

    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {

    }

    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
        logger.info(context.getJobDetail().getKey() + "jobWasExecuted..." + context.getJobRunTime());
        int status = 1;
        String errMsg = null;
        if (jobException != null) {
            status = 0;
            errMsg = jobException.getMessage();
        }
        saveLog(context.getJobDetail().getKey().getName(), status, errMsg);
    }

    private void saveLog(String jobName, int status, String errorMsg) {
        try (Connection connection = dataSource.getConnection()){
            PreparedStatement ps = connection.prepareStatement(INSERT_SQL);

            ps.setString(1, jobName);
            ps.setInt(2, status);
            ps.setString(3, errorMsg);
            ps.execute();

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

### 2.2 监听器如何与spring boot结合

spring boot quartz在加载的时候，会去加载所有的SchedulerFactoryBeanCustomizer，并把值注入到SchedulerFactoryBean中，代码逻辑在QuartzAutoConfiguration#customize中。所以只需下面这么做就好了

```java
@Configuration
public class QuartzListenerConfiguration {

    @Bean
    public SchedulerFactoryBeanCustomizer listenerCustomizer() {
        return schedulerFactoryBean -> schedulerFactoryBean.setGlobalJobListeners(new QuartzJobLogListener());
    }
}
```

