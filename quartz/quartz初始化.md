研究源码，从简单使用开始，跑一遍demo后，再研究是如何初始化的，我们先研究以下的代码：

```java
public class QuartzSimpleDemo {

    public static void main(String[] args) throws SchedulerException {
        //1. 创建Scheduler
        SchedulerFactory sfact = new StdSchedulerFactory();
        Scheduler sched  = sfact.getScheduler();

        //2. 创建job信息
        JobDetail testTaskJob = JobBuilder.newJob(TestTaskJob.class).storeDurably()
                .withIdentity("TestTaskJob").build();
        //3. 创建触发器
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule("*/5 * * * * ?");
        CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                .forJob(testTaskJob)
                .withIdentity("testTaskJob")
                .withSchedule(cronScheduleBuilder)
                .build();
        //4. 启动
        sched.scheduleJob(testTaskJob, cronTrigger);
        sched.start();
    }
}
```

## quartz原生初始化

### ![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/quartz%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B.png)

### 1. 创建Scheduler

```java
SchedulerFactory sfact = new StdSchedulerFactory();
```

工厂模式创建一个Scheduler工厂，quartz中有两个实现类，DirectSchedulerFactory和StdSchedulerFactory，常用StdSchedulerFactory类。

```
Scheduler sched  = sfact.getScheduler();
```

从工厂中获取一个Scheduler对象，这个是我们研究的重点。我们看下getSchedulerd的逻辑

```java
public Scheduler getScheduler() throws SchedulerException {
        //初始化配置文件
  			if (cfg == null) {
            initialize();
        }

        SchedulerRepository schedRep = SchedulerRepository.getInstance();
  			//根据scheduler名字从缓存中获取Scheduler对象
        Scheduler sched = schedRep.lookup(getSchedulerName());

        if (sched != null) {
            if (sched.isShutdown()) {
                schedRep.remove(getSchedulerName());
            } else {
                return sched;
            }
        }
  			
  			//如果缓存中没有，则初始化
        sched = instantiate();

        return sched;
    }
```
#### 1.1 初始化配置
- 如果创建SchedulerFactory时，传了配置文件，则使用传入的

- 如果创建SchedulerFactory时没有传，则看系统参数中配置了org.quartz.properties，也就-Dorg.quartz.properties是否配置。

- 如果系统参数没有传，或者传了找不到，则在各种路径中查找quartz.properties文件。所以建议直接在配置文件中新增quartz.properties文件就可以。  
#### 1.2 缓存中加载Scheduler
  上面代码是从SchedulerRepository获取缓存的Scheduler，SchedulerRepository中有一个全局的HashMap，key是Scheduler名，值是Scheduler对象，开始时肯定是空，所以调用instantiate方法.

#### 1.3 初始化

如果缓存中没有，则创建 Scheduler对象，看下创建逻辑，创建逻辑主要是读取配置，根据配置创建对应的类.这个方法有点长，不过注释写的好，很容易看懂,我留下主要的代码

```java
private Scheduler instantiate() throws SchedulerException {
        // Get Scheduler Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // If Proxying to remote scheduler, short-circuit here...
        // ~~~~~~~~~~~~~~~~~


        // Create class load helper

        // If Proxying to remote JMX scheduler, short-circuit here...
        // ~~~~~~~~~~~~~~~~~~
        
        // Get ThreadPool Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // Get JobStore Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // Set up any DataSources
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // Set up any SchedulerPlugins
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // Set up any JobListeners
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        
        // Set up any TriggerListeners
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


        // Get ThreadExecutor Properties
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


        // Fire everything up
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
       
    		// Create correct run-shell factory...
          
       //create QuartzSchedulerResources
       // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
       QuartzSchedulerResources rsrcs = new QuartzSchedulerResources();
       //create QuartzScheduler
       // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    
        qs = new QuartzScheduler(rsrcs, idleWaitTime, dbFailureRetry);
        qsInited = true;

        // Create Scheduler ref...
        Scheduler scheduler = instantiate(rsrcs, qs);

        schedRep.bind(scheduler);
        return scheduler;
        }
    }

```

上面的代码，我把代码都删了，留下注释和几行核心的代码，看起来更清晰。主要逻辑是根据配置创建不同的类。然后把所有的类都封装到QuartzSchedulerResources对象中。然后再创建QuartzScheduler对象。Scheduler大多数操作都是依赖QuartzScheduler完成的。最后创建Scheduler对象，并把Scheduler对象存到缓存中。

QuartzScheduler创建时启动了生产者线程，到研究quartz线程模型的时候会写到.

```java
public QuartzScheduler(QuartzSchedulerResources resources, long idleWaitTime, @Deprecated long dbRetryInterval)
        throws SchedulerException {
        this.resources = resources;
        if (resources.getJobStore() instanceof JobListener) {
            addInternalJobListener((JobListener)resources.getJobStore());
        }

        this.schedThread = new QuartzSchedulerThread(this, resources);
        ThreadExecutor schedThreadExecutor = resources.getThreadExecutor();
        schedThreadExecutor.execute(this.schedThread);
        if (idleWaitTime > 0) {
            this.schedThread.setIdleWaitTime(idleWaitTime);
        }
    }
```

### 2. 创建JobDetail

利用构造者模式创建JobDetail对象,比较简单

### 3. 创建Trigger

利用构造者模式创建Trigger对象,比较简单。

### 4. 调度Job

```java
public Date scheduleJob(JobDetail jobDetail,
            Trigger trigger) throws SchedulerException {
        validateState();

        if (jobDetail == null) {
            throw new SchedulerException("JobDetail cannot be null");
        }
        
        if (trigger == null) {
            throw new SchedulerException("Trigger cannot be null");
        }
        
        if (jobDetail.getKey() == null) {
            throw new SchedulerException("Job's key cannot be null");
        }

        if (jobDetail.getJobClass() == null) {
            throw new SchedulerException("Job's class cannot be null");
        }
        
        OperableTrigger trig = (OperableTrigger)trigger;

        if (trigger.getJobKey() == null) {
            trig.setJobKey(jobDetail.getKey());
        } else if (!trigger.getJobKey().equals(jobDetail.getKey())) {
            throw new SchedulerException(
                "Trigger does not reference given job!");
        }

        trig.validate();

        Calendar cal = null;
        if (trigger.getCalendarName() != null) {
            cal = resources.getJobStore().retrieveCalendar(trigger.getCalendarName());
        }
        Date ft = trig.computeFirstFireTime(cal);

        if (ft == null) {
            throw new SchedulerException(
                    "Based on configured schedule, the given trigger '" + trigger.getKey() + "' will never fire.");
        }

        resources.getJobStore().storeJobAndTrigger(jobDetail, trig);
        notifySchedulerListenersJobAdded(jobDetail);
        notifySchedulerThread(trigger.getNextFireTime().getTime());
        notifySchedulerListenersSchduled(trigger);

        return ft;
    }
```

job调度，开始参数校验，然后存储job和trigger，然后通知各种监听器。

### 5. 启动

启动主要也是通知监听事情，表示Scheduler启动起来了。

## spring boot quartz初始化
### 1. 查找配置类
spring boot quartz的使用我们并没有使用任何注解就能够使用，那它是如何初始化的呢？因为使用了spring boot，所以我猜测使用了spring boot的自动配置功能，所以我用IDEA搜索了QuartzAutoConfiguration，还真让我搜到了。如果不知道spring boot的加载过程，可以查看SchedulerFactory#getScheduler被哪里调用了，反推初始化过程。

```java
@Configuration
@ConditionalOnClass({ Scheduler.class, SchedulerFactoryBean.class, PlatformTransactionManager.class })
@EnableConfigurationProperties(QuartzProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class })
public class QuartzAutoConfiguration {}
```

QuartzAutoConfiguration使用了@Configuration注解，在spring boot启动的时候就会加载，当系统中有Scheduler相关的类时，就会创建对应的bean。
### 2. 构造函数

```java
public QuartzAutoConfiguration(QuartzProperties properties,
			ObjectProvider<SchedulerFactoryBeanCustomizer> customizers, ObjectProvider<JobDetail[]> jobDetails,
			Map<String, Calendar> calendars, ObjectProvider<Trigger[]> triggers,
			ApplicationContext applicationContext) {
		this.properties = properties;
		this.customizers = customizers;
		this.jobDetails = jobDetails.getIfAvailable();
		this.calendars = calendars;
		this.triggers = triggers.getIfAvailable();
		this.applicationContext = applicationContext;
	}
```

构造函数中注入了jobDetail和trigger。

### 3. SchedulerFactoryBean

```java
@Bean
	@ConditionalOnMissingBean
	public SchedulerFactoryBean quartzScheduler() {
		SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
		SpringBeanJobFactory jobFactory = new SpringBeanJobFactory();
		jobFactory.setApplicationContext(this.applicationContext);
		schedulerFactoryBean.setJobFactory(jobFactory);
		if (this.properties.getSchedulerName() != null) {
			schedulerFactoryBean.setSchedulerName(this.properties.getSchedulerName());
		}
		schedulerFactoryBean.setAutoStartup(this.properties.isAutoStartup());
		schedulerFactoryBean.setStartupDelay((int) this.properties.getStartupDelay().getSeconds());
		schedulerFactoryBean.setWaitForJobsToCompleteOnShutdown(this.properties.isWaitForJobsToCompleteOnShutdown());
		schedulerFactoryBean.setOverwriteExistingJobs(this.properties.isOverwriteExistingJobs());
		if (!this.properties.getProperties().isEmpty()) {
			schedulerFactoryBean.setQuartzProperties(asProperties(this.properties.getProperties()));
		}
		if (this.jobDetails != null && this.jobDetails.length > 0) {
			schedulerFactoryBean.setJobDetails(this.jobDetails);
		}
		if (this.calendars != null && !this.calendars.isEmpty()) {
			schedulerFactoryBean.setCalendars(this.calendars);
		}
		if (this.triggers != null && this.triggers.length > 0) {
			schedulerFactoryBean.setTriggers(this.triggers);
		}
		customize(schedulerFactoryBean);
		return schedulerFactoryBean;
	}
```

声明了一个SchedulerFactoryBean bean，设置了配置、jobDetail和trigger属性值。接下来看下SchedulerFactoryBean的初始化过程。

```java
public class SchedulerFactoryBean extends SchedulerAccessor implements FactoryBean<Scheduler>,
		BeanNameAware, ApplicationContextAware, InitializingBean, DisposableBean, SmartLifecycle {
    ...
}
```

SchedulerFactoryBean实现了InitializingBean和FactoryBean，所以在初始化的时候会调用afterPropertiesSet方法，在得到bena的时候会调用getObject方法，先看afterPropertiesSet方法。

```java
@Override
	public void afterPropertiesSet() throws Exception {
		if (this.dataSource == null && this.nonTransactionalDataSource != null) {
			this.dataSource = this.nonTransactionalDataSource;
		}

		if (this.applicationContext != null && this.resourceLoader == null) {
			this.resourceLoader = this.applicationContext;
		}

		// Initialize the Scheduler instance...
		this.scheduler = prepareScheduler(prepareSchedulerFactory());
		try {
			registerListeners();
			registerJobsAndTriggers();
		}
		catch (Exception ex) {
			try {
				this.scheduler.shutdown(true);
			}
			catch (Exception ex2) {
				logger.debug("Scheduler shutdown exception after registration failure", ex2);
			}
			throw ex;
		}
	}
```

看到初始化scheduler的注释，所以应该就是在prepareScheduler中初始化scheduler对象的。在创建scheduler之前，先创建了SchedulerFactory，看下创建过程。

```java
private SchedulerFactory prepareSchedulerFactory() throws SchedulerException, IOException {
		SchedulerFactory schedulerFactory = this.schedulerFactory;
		if (schedulerFactory == null) {
			// Create local SchedulerFactory instance (typically a StdSchedulerFactory)
			schedulerFactory = BeanUtils.instantiateClass(this.schedulerFactoryClass);
			if (schedulerFactory instanceof StdSchedulerFactory) {
				initSchedulerFactory((StdSchedulerFactory) schedulerFactory);
			}
			else if (this.configLocation != null || this.quartzProperties != null ||
					this.taskExecutor != null || this.dataSource != null) {
				throw new IllegalArgumentException(
						"StdSchedulerFactory required for applying Quartz properties: " + schedulerFactory);
			}
			// Otherwise, no local settings to be applied via StdSchedulerFactory.initialize(Properties)
		}
		// Otherwise, assume that externally provided factory has been initialized with appropriate settings
		return schedulerFactory;
	}
```

在SchedulerFactoryBean创建的时候并没有给schedulerFactory属性赋值，所以schedulerFactory是空的，如果是空的则加载schedulerFactoryClass类，schedulerFactoryClass的值是StdSchedulerFactory。这和我们直接使用quartz是一样的。接下来就是创建Scheduler对象。

```java
private Scheduler prepareScheduler(SchedulerFactory schedulerFactory) throws SchedulerException {
		...
		// Get Scheduler instance from SchedulerFactory.
		try {
			Scheduler scheduler = createScheduler(schedulerFactory, this.schedulerName);
			...
			return scheduler;
		}
```

prepareScheduler方法又调用了createScheduler，继续跟踪

```java
protected Scheduler createScheduler(SchedulerFactory schedulerFactory, @Nullable String schedulerName)
			throws SchedulerException {

		// Override thread context ClassLoader to work around naive Quartz ClassLoadHelper loading.
		Thread currentThread = Thread.currentThread();
		ClassLoader threadContextClassLoader = currentThread.getContextClassLoader();
		boolean overrideClassLoader = (this.resourceLoader != null &&
				this.resourceLoader.getClassLoader() != threadContextClassLoader);
		if (overrideClassLoader) {
			currentThread.setContextClassLoader(this.resourceLoader.getClassLoader());
		}
		try {
      //从缓存中获取Scheduler
			SchedulerRepository repository = SchedulerRepository.getInstance();
			synchronized (repository) {
				Scheduler existingScheduler = (schedulerName != null ? repository.lookup(schedulerName) : null);
				Scheduler newScheduler = schedulerFactory.getScheduler();
				if (newScheduler == existingScheduler) {
					throw new IllegalStateException("Active Scheduler of name '" + schedulerName + "' already registered " +
							"in Quartz SchedulerRepository. Cannot create a new Spring-managed Scheduler of the same name!");
				}
				if (!this.exposeSchedulerInRepository) {
					// Need to remove it in this case, since Quartz shares the Scheduler instance by default!
					SchedulerRepository.getInstance().remove(newScheduler.getSchedulerName());
				}
				return newScheduler;
			}
		}
		finally {
			if (overrideClassLoader) {
				// Reset original thread context ClassLoader.
				currentThread.setContextClassLoader(threadContextClassLoader);
			}
		}
	}

```

创建的过程和原生的quartz的逻辑是一样的了，都是先从缓存中获取，如果缓存中没有，再调用getScheduler方法获取。

所以Scheduler的创建spring boot只是包装了一层壳，最后的逻辑和quartz原生是一样的。那么job和trigger是什么时候注入进去的呢？afterPropertiesSet方法还只分析了一个方法，还有两个方法

```java
registerListeners();
registerJobsAndTriggers();
```

见名知意，这两个方法就是注册监听器、job和detail的。注入的逻辑和原生没什么区别，这里就不再阐述了。

## 总结

这篇笔记分析了quartz原生的初始化过程和spring boot quartz初始化过程。spring boot初始化bean基本都是包装了一层AutoConfiguration需要的bean，底层实现还是和原生的一样。还介绍了阅读源码的技术，去除干扰，带着目标去找代码的实现。



