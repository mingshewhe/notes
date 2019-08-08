## 基础概念

- Scheduler - 与调度器交互的主要API。
- Job - 需要被调度器调度的任务必须实现的接口。
- JobDetail - 用于定义任务的实例。
- Trigger - 用于定义调度器何时调度任务执行的组件。
- JobBuilder - 用于定义或创建JobDetail的实例 。
- TriggerBuilder - 用于定义或创建触发器实例。

## 原生使用

### 1. 新建job

job需要实现Job接口，重写execute方法

```java
public class TestTaskJob implements Job {
    private static final Logger logger = LoggerFactory.getLogger(TestTaskJob.class);
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        String localDateTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        logger.info("TestTaskJob-->" + localDateTime);
    }
}
```

### 2. 配置文件

如果不配置配置文件，则使用默认配置文件 quartz.properties,位于/org/quartz-scheduler/quartz/2.3.1/quartz-2.3.1.jar!/org/quartz/quartz.properties

### 3. 启动quartz

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

## spring boot quartz使用

### 1. pom.xml配置

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

### 2. 创建job

job需要实现QuartzJobBean接口，并实现executeInternal方法

```java
public class TestTaskJob1 extends QuartzJobBean {
    private static final Logger logger = LoggerFactory.getLogger(TestTaskJob1.class);

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String localDateTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        logger.info("TestTaskJob1-->" + localDateTime);
    }
}
```

### 3. 声明bean配置

声明jobDetail和trigger bean

```java
@Configuration
public class QuartzConfig {
  @Bean
    public JobDetail testQuartz1() {
        return JobBuilder.newJob(TestTaskJob1.class).storeDurably().withIdentity("TestTaskJob1").build();
    }

    @Bean
    public Trigger testQuartzTrigger2() {
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule("*/5 * * * * ?");
        return TriggerBuilder.newTrigger()
                .forJob(testQuartz1())
                .withIdentity("testTaskJob1")
                .withSchedule(cronScheduleBuilder)
                .build();
    }
}
```

### 4. 启动quart

```java
@SpringBootApplication
public class SpringBootQuartz {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootQuartz.class, args);
    }
}
```

## 总结

对比原生的quart和Spring boot quartz，spring boot quart简化了很多，不需要我们声明Scheduler，我们只要关注job和trigger的创建。