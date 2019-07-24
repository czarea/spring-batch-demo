## 1.简介
批处理场景在很多系统中都存在，大部分的场景为：从数据库，文件或队列中读取大量记录，然后以某种方式处理数据，最终将处理结果持久化到数据库或者磁盘文件。比如：从内部和外部系统接收的信息，并对信息做一些校验，业务规则处理，最终记录系统中。

批处理程序一般存在以下需求：
- 定期提交批处理
- 并发批处理：并行处理作业
- 分阶段的企业消息驱动处理
- 大规模并行批处理
- 失败后手动或预定重启
- 依赖步骤的顺序处理（使用扩展的toworkflow驱动批次）
- 部分处理：跳过记录（例如，回滚时）
- 整批交易，适用于批量较小或现有存储过程/脚本的情况

spring batch的特点：
- 事务管理，让您专注于业务处理，实现批处理机制，你可以引入平台事务机制或其他事务管理器机制
- 基于块Chunk的处理，通过将一大段大量数据分成一段段小数据来处理，。
- 启动/停止/重新启动/跳过/重试功能，以处理过程的非交互式管理。
- 基于Web的管理界面（Spring Batch Admin），它提供了一个用于管理任务的API。
- 基于Spring框架，因此它包括所有配置选项，包括依赖注入。
- 符合JSR 352：Java平台的批处理应用程序。
- 基于数据库管理的批处理，可与Spring Cloud Task结合，适合分布式集群下处理。
- 能够进行多线程并行处理，分布式系统下并行处理，变成一种弹性Job分布式处理框架。


spring batch一般结合Quartz等定时调度框架使用。

spring batch4.1的新特性：
- 一个新的@SpringBatchTest注释，用于简化测试批处理组件
- 一个新的@EnableBatchIntegration注释，用于简化远程分块和分区配置
- 一个新的JsonItemReader并JsonFileItemWriter支持JSON格式
- 添加对使用Bean Validation API验证项目的支持
- 添加对JSR-305注释的支持
- FlatFileItemWriterBuilderAPI的增强功能


## 2.框架
spring batch框架图：
![框架](https://docs.spring.io/spring-batch/4.1.x/reference/html/images/spring-batch-reference-model.png)

从上图可以看到Job就是一次批处理任务，并且批处理任务可以拆分为多个步骤，每个步骤有输入，处理，输出三个步骤。

![job层级](https://docs.spring.io/spring-batch/4.1.x/reference/html/images/jobHeirarchyWithSteps.png)

spring batch 给每个运行的Job一个JobInstance实例对象，Job只有一个，每次运行根据JobParameters来区分，比如每天运行一次的以日期，每个Job有自己的向下文对象JobExecution，每个步骤也可以有StepExecution对象。

### 2.1 Hello World


**Configuration**

```
@Configuration
@EnableBatchProcessing
public class BatchConfiguration {


    @Autowired
    public JobBuilderFactory jobBuilderFactory;

    @Autowired
    public StepBuilderFactory stepBuilderFactory;

    @Bean
    public FlatFileItemReader<Person> reader() {
        return new FlatFileItemReaderBuilder<Person>()
            .name("personItemReader")
            .resource(new ClassPathResource("sample-data.csv"))
            .delimited()
            .names(new String[]{"firstName", "lastName"})
            .fieldSetMapper(new BeanWrapperFieldSetMapper<Person>() {{
                setTargetType(Person.class);
            }})
            .build();
    }

    @Bean
    public PersonItemProcessor processor() {
        return new PersonItemProcessor();
    }

    @Bean
    public JdbcBatchItemWriter<Person> writer(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<Person>()
            .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
            .sql("INSERT INTO people (first_name, last_name) VALUES (:firstName, :lastName)")
            .dataSource(dataSource)
            .build();
    }

    @Bean
    public Job importUserJob(JobCompletionNotificationListener listener, Step step1) {
        return jobBuilderFactory.get("importUserJob")
            .incrementer(new RunIdIncrementer())
            .listener(listener)
            .flow(step1)
            .end()
            .build();
    }

    @Bean
    public Step step1(JdbcBatchItemWriter<Person> writer) {
        return stepBuilderFactory.get("step1")
            .<Person, Person> chunk(10)
            .reader(reader())
            .processor(processor())
            .writer(writer)
            .build();
    }
}
```

**Processor**

```
public class PersonItemProcessor implements ItemProcessor<Person, Person> {

    private static final Logger log = LoggerFactory.getLogger(PersonItemProcessor.class);

    @Override
    public Person process(final Person person) throws Exception {
        final String firstName = person.getFirstName().toUpperCase();
        final String lastName = person.getLastName().toUpperCase();

        final Person transformedPerson = new Person(firstName, lastName);

        log.info("Converting (" + person + ") into (" + transformedPerson + ")");

        return transformedPerson;
    }
}
```

**Listener**

```
@Component
public class JobCompletionNotificationListener extends JobExecutionListenerSupport {

    private static final Logger log = LoggerFactory.getLogger(JobCompletionNotificationListener.class);

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public JobCompletionNotificationListener(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if(jobExecution.getStatus() == BatchStatus.COMPLETED) {
            log.info("!!! JOB FINISHED! Time to verify the results");

            jdbcTemplate.query("SELECT first_name, last_name FROM people",
                (rs, row) -> new Person(
                    rs.getString(1),
                    rs.getString(2))
            ).forEach(person -> log.info("Found <" + person + "> in the database."));
        }
    }
}
```


[代码地址](https://github.com/czarea/spring-batch-demo)


## 3.总结
spring batch 如果存在统计，二次使用中间数据，不是很友好，[可参考](https://stackoverflow.com/questions/55803553/in-spring-batch-processor-wait-one-process-to-finish-handler-all-data-then-execu)


>参考一：[spring batch reference](https://docs.spring.io/spring-batch/4.1.x/reference/html/index-single.html)

>参考二：[spring batch service](https://spring.io/guides/gs/batch-processing/)
