---
title: "SpringBoot系列 - 批处理"
date: 2017-08-01 19:10:22 +0800
comments: true
toc: true
categories: spring
tags: [springboot]
---

Spring Batch是一个轻量级的框架,完全面向Spring的批处理框架,用于企业级大量的数据读写处理系统。以POJO和Spring 框架为基础，
包括日志记录/跟踪，事务管理、 作业处理统计工作重新启动、跳过、资源管理等功能。

Spring Batch官网是这样介绍的自己：一款轻量的、全面的批处理框架，用于开发强大的日常运营的企业级批处理应用程序。

框架主要有以下功能：

* Transaction management（事务管理）
* Chunk based processing（基于块的处理）
* Declarative I/O（声明式的输入输出）
* Start/Stop/Restart（启动/停止/再启动）
* Retry/Skip（重试/跳过）

如果你的批处理程序需要使用上面的功能，那就大胆地使用它吧。<!--more-->

## 框架介绍

先用一个图让你有一个大概印象，这个东西是什么：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-batch01.png)

框架一共有5个主要角色：

1. JobRepository是用户注册Job的容器，就是存储数据的地方，可以看做是一个数据库的接口，在任务执行的时候需要通过它来记录任务状态等等信息。
1. JobLauncher是任务启动器，通过它来启动任务，可以看做是程序的入口。
1. Job代表着一个具体的任务。
1. Step代表着一个具体的步骤，一个Job可以包含多个Step（想象把大象放进冰箱这个任务需要多少个步骤你就明白了）。
1. Item就是输出->处理->输出，一个完整Step流程。

下面简要的介绍一下这5个角色

### JobRepository

JobRepository用于存储任务执行的状态信息，比如什么时间点执行了什么任务、任务执行结果如何等等。
框架提供了2种实现，一种是通过Map形式保存在内存中，当Java程序重启后任务信息也就丢失了，
并且在分布式下无法获取其他节点的任务执行情况；另一种是保存在数据库中，并且将数据保存在下面6张表里：

* BATCH_JOB_INSTANCE
* BATCH_JOB_EXECUTION_PARAMS
* BATCH_JOB_EXECUTION
* BATCH_STEP_EXECUTION
* BATCH_JOB_EXECUTION_CONTEXT
* BATCH_STEP_EXECUTION_CONTEXT

Spring Batch框架的JobRepository支持主流的数据库：DB2、Derby、H2、HSQLDB、MySQL、Oracle、PostgreSQL、SQLServer、Sybase。

### JobLauncher

JobLauncher是任务启动器，该接口只有一个run方法：

``` java
public interface JobLauncher {
    JobExecution run(Job job, JobParameters jobParameters);
}
```

除了传入Job对象之外，还需要传入JobParameters对象，后续讲到Job再解释为什么要多传一个JobParameters。
通过JobLauncher可以在Java程序中调用批处理任务，也可以通过命令行或者其他框架
（如定时调度框架Quartz、Web后台框架Spring MVC）中调用批处理任务。
Spring Batch框架提供了一个JobLauncher的实现类SimpleJobLauncher。

### Job

Job代表着一个任务，一个Job与一个或者多个JobInstance相关联，而一个JobInstance又与一个或者多个JobExecution相关联：

![](https://xnstatic-1253397658.file.myqcloud.com/sb-batch02.png)

考虑到任务可能不是只执行一次就再也不执行了，更多的情况可能是定时任务，如每天执行一次，每个星期执行一次等等，
那么为了区分每次执行的任务，框架使用了JobInstance。如上图所示，Job是一个EndOfDay（每天最后时刻执行的任务），
那么其中一个JobInstance就代表着2007年5月5日那天执行的任务实例。
框架通过在执行JobLauncher.run(Job, JobParameters)方法时传入的JobParameters来区分是哪一天的任务。

由于2007年5月5日那天执行的任务可能不会一次就执行完成，比如中途被停止，或者出现异常导致中断，
需要多执行几次才能完成，所以框架使用了JobExecution来表示每次执行的任务。

### Step

一个Job任务可以分为几个Step步骤，与JobExection相同，每次执行Step的时候使用StepExecution来表示执行的步骤。
每一个Step还包含着一个ItemReader、ItemProcessor、ItemWriter，下面分别介绍这三者。

**ItemReader**

ItemReader代表着读操作，其接口如下：

``` java
public interface ItemReader<T> {
    T read();
}
```

框架已经提供了多种ItemReader接口的实现类，包括对文本文件、CSV文件、XML文件、数据库、JMS消息等读的处理，当然我们也可以自己实现该接口。

**ItemProcessor**

ItemReader代表着处理操作，其接口如下：

``` java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}
```

process方法的形参传入I类型的对象，通过处理后返回O型的对象。开发者可以实现自己的业务代码来对数据进行处理。

**ItemWriter**

ItemReader代表着写操作，其接口如下：

``` java
public interface ItemWriter<T> {
    void write(List<? extends T> items) throws Exception;
}
```

框架已经提供了多种ItemWriter接口的实现类，包括对文本文件、CSV文件、XML文件、数据库、JMS消息等写的处理，当然我们也可以自己实现该接口。

### Job监听器

还可以自定义任务监听器，在任务启动和完成之后进行相应的通知和响应。

``` java
/**
 * 监听器实现JobExecutionListener接口，并重写其beforeJob，afterJob方法即可
 */
public class MyJobListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution jobExecution) {
        
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        
    }
}
```

### 数据校验

我们可以JSR-303(主要实现由hibernate-validator)的注解，来校验ItemReader读取到的数据是否满足要求。

首先让我们的ItemProcessor实现ValidatingItemProcessor接口：

``` java
public class MyItemProcessor extends ValidatingItemProcessor<User> {
    @Override
    public User process(User item) throws ValidationException {
        super.process(item);
        return item;
    }
}
```

然后定义自己的校验器，实现的Validator接口来自于Spring，我们将使用JSR-303的Validator来校验：

``` java
public class MyBeanValidator<T> implements Validator<T>,InitializingBean {
    private Validator validator;

    @Override
    public void afterPropertiesSet() throws Exception {
    }
    
    @Override
    public void validate(T value)throws ValidationException{
    }
}
```

在定义我们的MyItemProcessor时必须将MyBeanValidator设置进去，代码如下：

``` java
@Bean
public ItemProcessor<User,User> processor(){
    //新建ItemProcessor接口的实现类返回
    MyItemProcessor processor = new MyItemProcessor();
    processor.setValidator(myBeanValidator());
    return processor;
}

@Bean
public Validator<User> myBeanValidator(){
    return new MyBeanValidator<User>();
}
```

## SpringBoot集成实例

Spring Boot对Spring Batch支持的源码位于`org.springframework.boot.autoconfigure.batch`下。

Spring Boot为我们自动初始化了Spring Batch存储批处理记录的数据库，且当我们程序启动时，
会自动执行我们定义的Job的Bean，不过我们可以通过配置定时器或手动触发方式启动。

Spring Boot提供如下属性来定制Spring Batch：

```
#启动时要执行的job，默认执行全部job
spring.batch.job.name=job1,job2
#是否自动执行定义的job，默认是
spring.batch.job.enabled=true
#是否初始化Spring Batch的数据库，默认为是
spring.batch.initializer.enabled=true
spring.batch.schema=
#设置Spring Batch的数据库表的前缀
spring.batch.table-prefix=
```

下面通过一个真实的例子，需要将客户导过来的csv文件导入到我们的业务Oracle数据库中，来说明怎样在SpringBoot中使用批处理框架。

### 添加maven依赖

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.hsqldb</groupId>
            <artifactId>hsqldb</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!--添加hibernate-validator依赖，作为数据校验使用-->
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.7.Final</version>
</dependency>
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
    <version>2.0.1.Final</version>
</dependency>
<dependency>
    <groupId>javax.el</groupId>
    <artifactId>javax.el-api</artifactId>
    <version>3.0.1-b04</version>
</dependency>
<dependency>
    <groupId>org.glassfish.web</groupId>
    <artifactId>javax.el</artifactId>
    <version>2.2.6</version>
</dependency>
<!--添加hibernate-validator依赖 END-->

<!-- Oracle驱动包 -->
<dependency>
    <groupId>com.oracle</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.4.0-atlassian-hosted</version>
</dependency>
```

### 配置application.yml

``` yml
###################  spring配置  ###################
spring:
  profiles:
    active: dev
  batch:
    job:
      enabled: false
  datasource:
    driver-class-name: oracle.jdbc.driver.OracleDriver
    url: jdbc:oracle:thin:@xxx.xxx.xxx.xxx:1521:orcl11g
    username: adm_real
    password: adm_real
```

真实csv数据，位于`src/main/resources/NT_BSC_BUDGETVTOLL.csv`中

表定义如下：

``` sql
CREATE TABLE "ADM_REAL"."NT_BSC_BUDGETVTOLL" (
    "F_ID" VARCHAR2(100 BYTE) NOT NULL ,
    "F_YEAR" VARCHAR2(4 BYTE) NULL ,
    "F_TOLLID" VARCHAR2(50 BYTE) NULL ,
    "F_BUDGETID" VARCHAR2(50 BYTE) NULL ,
    "F_CBUDGETID" VARCHAR2(50 BYTE) NULL ,
    "F_VERSION" VARCHAR2(1 BYTE) DEFAULT '1'  NULL ,
    "F_AUDITMSG" VARCHAR2(100 BYTE) NULL ,
    "F_TRIALSTATUS" VARCHAR2(1 BYTE) DEFAULT '0'  NULL ,
    "F_FIRAUDITER" VARCHAR2(50 BYTE) NULL ,
    "F_FIRAUDITTIME" VARCHAR2(20 BYTE) NULL ,
    "F_FINAUDITER" VARCHAR2(50 BYTE) NULL ,
    "F_FINAUDITTIME" VARCHAR2(64 BYTE) NULL ,
    "F_EDITTIME" VARCHAR2(64 BYTE) NULL ,
    "F_STARTDATE" VARCHAR2(8 BYTE) NULL ,
    "F_ENDDATE" VARCHAR2(8 BYTE) NULL ,
)
```

### 领域模型类

``` java
public class BudgetVtoll {
    private String id;
    private String year;
    private String tollid;
    private String budgetid;
    private String cbudgetid;
    private String version;
    /**
     * 使用JSR-303注解来校验数据
     */
    @Size(max = 100)
    private String auditmsg;
    private String trialstatus;
    private String firauditer;
    private String firaudittime;
    private String finauditer;
    private String finaudittime;
    private String edittime;
    private String startdate;
    private String enddate;
    
    // 下面省略get/set方法
}
```

### 数据处理及校验

定义处理器

``` java
public class CsvItemProcessor extends ValidatingItemProcessor<BudgetVtoll> {
    @Override
    public BudgetVtoll process(BudgetVtoll item) throws ValidationException {
        /*
         * 需要执行super.process(item)才会调用自定义校验器
         */
        super.process(item);
        /*
         * 对数据进行简单的处理和转换 todo
         */
        return item;
    }
}
```

校验器定义：

``` java
public class CsvBeanValidator<T> implements Validator<T>, InitializingBean {
    private javax.validation.Validator validator;

    @Override
    public void validate(T value) throws ValidationException {
        /*
         * 使用Validator的validate方法校验数据
         */
        Set<ConstraintViolation<T>> constraintViolations = validator.validate(value);
        if (constraintViolations.size() > 0) {
            StringBuilder message = new StringBuilder();
            for (ConstraintViolation<T> constraintViolation : constraintViolations) {
                message.append(constraintViolation.getMessage()).append("\n");
            }
            throw new ValidationException(message.toString());
        }
    }

    /**
     * 使用JSR-303的Validator来校验我们的数据，在此进行JSR-303的Validator的初始化
     */
    @Override
    public void afterPropertiesSet() {
        ValidatorFactory validatorFactory = Validation.buildDefaultValidatorFactory();
        validator = validatorFactory.usingContext().getValidator();
    }
}
```

### 任务监听器

``` java
public class CsvJobListener implements JobExecutionListener {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    private long startTime;
    private long endTime;

    @Override
    public void beforeJob(JobExecution jobExecution) {
        startTime = System.currentTimeMillis();
        logger.info("任务处理开始");
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        endTime = System.currentTimeMillis();
        logger.info("任务处理结束，总耗时=" + (endTime - startTime) + "ms");
    }
}
```

### 配置类

``` java
@Configuration
@EnableBatchProcessing
public class CsvBatchConfig {
    /**
     * ItemReader定义,用来读取数据
     * 1，使用FlatFileItemReader读取文件
     * 2，使用FlatFileItemReader的setResource方法设置csv文件的路径
     * 3，对此对cvs文件的数据和领域模型类做对应映射
     *
     * @return FlatFileItemReader
     */
    @Bean
    @StepScope
    public FlatFileItemReader<BudgetVtoll> reader(@Value("#{jobParameters['input.file.name']}") String pathToFile) {
        FlatFileItemReader<BudgetVtoll> reader = new FlatFileItemReader<>();
        // reader.setResource(new ClassPathResource(pathToFile));
        reader.setResource(new FileSystemResource(pathToFile));
        reader.setLineMapper(new DefaultLineMapper<BudgetVtoll>() {
            {
                setLineTokenizer(new DelimitedLineTokenizer(",") {
                    {
                        setNames(new String[]{
                                "id","year","tollid","budgetid", "cbudgetid", "version", "auditmsg", "trialstatus",
                                "firauditer", "firaudittime", "finauditer", "finaudittime", "edittime", "startdate", "enddate"
                        });
                    }
                });
                setFieldSetMapper(new BeanWrapperFieldSetMapper<BudgetVtoll>() {{
                    setTargetType(BudgetVtoll.class);
                }});
            }
        });
        return reader;
    }

    /**
     * ItemProcessor定义，用来处理数据
     *
     * @return
     */
    @Bean
    public ItemProcessor<BudgetVtoll, BudgetVtoll> processor() {
        //使用我们自定义的ItemProcessor的实现CsvItemProcessor
        CsvItemProcessor processor = new CsvItemProcessor();
        //为processor指定校验器为CsvBeanValidator()
        processor.setValidator(csvBeanValidator());
        return processor;
    }

    /**
     * ItemWriter定义，用来输出数据
     * spring能让容器中已有的Bean以参数的形式注入，Spring Boot已经为我们定义了dataSource
     *
     * @param dataSource
     * @return
     */
    @Bean
    public ItemWriter<BudgetVtoll> writer(DruidDataSource dataSource) {
        JdbcBatchItemWriter<BudgetVtoll> writer = new JdbcBatchItemWriter<>();
        //我们使用JDBC批处理的JdbcBatchItemWriter来写数据到数据库
        writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());

        String sql = "insert into BudgetVtoll " + " (f_id,f_year,f_tollid,f_budgetid,f_cbudgetid,f_version,f_auditmsg,f_trialstatus,f_firauditer,f_firaudittime,f_finauditer,f_finaudittime,f_edittime,f_startdate,f_enddate) "
                + " values(:id,:year,:tollid,:budgetid,:cbudgetid,:version,:auditmsg,:trialstatus,:firauditer,:firaudittime,:finauditer,:finaudittime,:edittime,:startdate,:enddate)";
        //在此设置要执行批处理的SQL语句
        writer.setSql(sql);
        writer.setDataSource(dataSource);
        return writer;
    }

    /**
     * JobRepository，用来注册Job的容器
     * jobRepositor的定义需要dataSource和transactionManager，Spring Boot已为我们自动配置了
     * 这两个类，Spring可通过方法注入已有的Bean
     *
     * @param dataSource
     * @param transactionManager
     * @return
     * @throws Exception
     */
    @Bean
    public JobRepository jobRepository(DruidDataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {

        JobRepositoryFactoryBean jobRepositoryFactoryBean = new JobRepositoryFactoryBean();
        jobRepositoryFactoryBean.setDataSource(dataSource);
        jobRepositoryFactoryBean.setTransactionManager(transactionManager);
        jobRepositoryFactoryBean.setDatabaseType(String.valueOf(DatabaseType.ORACLE));
        // 下面事务隔离级别的配置是针对Oracle的
        jobRepositoryFactoryBean.setIsolationLevelForCreate("ISOLATION_READ_COMMITTED");
        jobRepositoryFactoryBean.afterPropertiesSet();
        return jobRepositoryFactoryBean.getObject();
    }

    /**
     * JobLauncher定义，用来启动Job的接口
     *
     * @param dataSource
     * @param transactionManager
     * @return
     * @throws Exception
     */
    @Bean
    public SimpleJobLauncher jobLauncher(DruidDataSource dataSource, PlatformTransactionManager transactionManager) throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
        jobLauncher.setJobRepository(jobRepository(dataSource, transactionManager));
        return jobLauncher;
    }

    /**
     * Job定义，我们要实际执行的任务，包含一个或多个Step
     *
     * @param jobBuilderFactory
     * @param s1
     * @return
     */
    @Bean
    public Job importJob(JobBuilderFactory jobBuilderFactory, Step s1) {
        return jobBuilderFactory.get("importJob")
                .incrementer(new RunIdIncrementer())
                .flow(s1)//为Job指定Step
                .end()
                .listener(csvJobListener())//绑定监听器csvJobListener
                .build();
    }

    /**
     * step步骤，包含ItemReader，ItemProcessor和ItemWriter
     *
     * @param stepBuilderFactory
     * @param reader
     * @param writer
     * @param processor
     * @return
     */
    @Bean
    public Step step1(StepBuilderFactory stepBuilderFactory, ItemReader<BudgetVtoll> reader, ItemWriter<BudgetVtoll> writer,
                      ItemProcessor<BudgetVtoll, BudgetVtoll> processor) {
        return stepBuilderFactory
                .get("step1")
                .<BudgetVtoll, BudgetVtoll>chunk(65000)//批处理每次提交65000条数据
                .reader(reader)//给step绑定reader
                .processor(processor)//给step绑定processor
                .writer(writer)//给step绑定writer
                .build();
    }

    @Bean
    public CsvJobListener csvJobListener() {
        return new CsvJobListener();
    }

    @Bean
    public Validator<BudgetVtoll> csvBeanValidator() {
        return new MyBeanValidator<>();
    }
}
```

注意，在定义Job的时候，我通过`@StepScope`注解，可以通过传递参数的方式将csv文件路径传进去。

### 测试类

最后再让我们写个测试类看看能不能成功：

``` java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class BatchServiceTest {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    @Autowired
    private JobLauncher jobLauncher;
    @Autowired
    private Job importJob;

    @Test
    public void testBatch1() throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
                .addLong("time", System.currentTimeMillis())
                .addString("input.file.name", "E:\\NT_BSC_BUDGETVTOLL.csv")
                .toJobParameters();
        jobLauncher.run(importJob, jobParameters);
        logger.info("testBatch1执行完成");
    }
}
```

把csv文件放入resources/目录下面，然后执行测试。看看输出结果：

```
: Job: [FlowJob: [name=importJob]] launched with the following parameters: [{time=1517654976174, input.file.name=E:\NT_BSC_BUDGETVTOLL.csv}]
: 任务处理开始
: Executing step: [step1]
: 任务处理结束，总耗时=810ms
: Job: [FlowJob: [name=importJob]] completed with the following parameters: [{time=1517654976174, input.file.name=E:\NT_BSC_BUDGETVTOLL.csv}] and the following status: [COMPLETED]
: testBatch1执行完成
```

再去查看数据库里面的数据，已经正常写入了。

![](https://xnstatic-1253397658.file.myqcloud.com/sb-batch03.png)

## 并发执行

SpringBatch批处理框架默认使用单线程完成任务的执行，但是他提供了对线程池的支持。
使用tasklet的task-executor属性可以很容易的将普通的step转成多线程的step。

* task-executor:  任务执行处理器，定义后采用多线程执行任务，需要考虑线程安全问题。
* throttle-limit: 最大使用线程池数目。

Spring Core 为我们提供了多种执行器实现（包括多种异步执行器），我们可以根据实际情况灵活选择使用。

类名	                     | 描述                                        | 是否异步
-------------------------|---------------------------------------------|-------------------------
ThrottledTaskExecutor    | 该执行器为其他任意执行器的装饰类              | 视被装饰的执行器而定
SyncTaskExecutor         | 简单同步执行器                               | 否
SimpleAsyncTaskExecutor	 | 简单异步执行器，提供最基本的异步实现          | 是
WorkManagerTaskExecutor  | 该类作为通过 JCA 规范进行任务执行的实现       | 是
ThreadPoolTaskExecutor   | 线程池任务执行器                             | 是

在多线程step中为了保证代码处理的正确性，要求所有在多线程step中处理的对象和操作必须都是线程安全的。
但是SpringBatch框架提供的大部分的ItemReader、ItemWriter都不是线程安全的。所以需要自己保证多线程处理时候的线程安全。

最常见的需要并行处理情况是，job中不同的step没有先后顺序，可以在执行期间并行的执行。
SpringBatch提供了并行step的能力。可以通过split元素来定义并行的作业流。

## FAQ

在进行Oracle操作的时候报一个错误：

```
ORA-08177: 无法连续访问此事务处理
```

原因：Spring Batch 默认是 `ISOLATION_SERIALIZABLE`

官方文档说明：

```
The default is ISOLATION_SERIALIZABLE, which prevents accidental concurrent execution of the same job (ISOLATION_REPEATABLE_READ would work as well).
```

而Oracle默认的事务隔离级别是ISOLATION_READ_COMMITTED

解决方案就是修改`JobRepositoryFactoryBean`的定义，加一个配置：

``` java
jobRepositoryFactoryBean.setIsolationLevelForCreate("ISOLATION_READ_COMMITTED");
```

