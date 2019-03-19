# Configuring a Step

## TaskletStep

> Chunk-oriented processing is not the only way to process in a Step. What if a Step must consist as a simple stored procedure call? You could implement the call as an ItemReader and return null after the procedure finishes, but it is a bit unnatural since there would need to be a no-op ItemWriter. Spring Batch provides the TaskletStep for this scenario.

> The Tasklet is a simple interface that has one method, execute, **which will be a called repeatedly by the TaskletStep until it either returns RepeatStatus.FINISHED or throws an exception to signal a failure.** Each call to the Tasklet is wrapped in a transaction. Tasklet implementors might call a stored procedure, a script, or a simple SQL update statement

- example code
```java
//Setting Tasklet calss
@Component
public class TaskletStepConfig {

    private StepBuilderFactory stepBuilderFactory;
    private TakoyakiFoodTruckRepository repository;
    private PlatformTransactionManager txManager;

    @Autowired
    public UpdateStep(StepBuilderFactory stepBuilderFactory,
                      TakoyakiFoodTruckRepository repository,
                      @Qualifier("demoTransactionManager") PlatformTransactionManager txManager) {
        this.stepBuilderFactory = stepBuilderFactory;
        this.repository = repository;
        this.txManager = txManager;
    }

    private Tasklet taskletStep() {
        return (StepContribution contribution, ChunkContext chunkContext) -> {
            repository.findAll().forEach(
                    truck -> {
                        truck.setName(truck.getName() + "_takoyaki_truck");
                        repository.save(truck);
                    }
            );
            return RepeatStatus.FINISHED;
        };
    }

    @Bean
    public Step secondStep() {
        return stepBuilderFactory.get("secondStep")
                .transactionManager(txManager)
                .tasklet(taskletStep())
                .build();
    }

    @Bean
    public Step finalStep() {
        return stepBuilderFactory.get("finalStep")
                .transactionManager(txManager)
                .tasklet((StepContribution contribution, ChunkContext chunkContext) -> {
                    repository.deleteAll();
                    return RepeatStatus.FINISHED;
                })
                .build();
    }

}

//Job class
@Configuration
@EnableBatchProcessing
public class JobConfig {

    private static final String JOB_NAME = "demoJop";

    private JobBuilderFactory jobBuilderFactory;


    @Autowired
    public JobConfig(JobBuilderFactory jobBuilderFactory) {
        this.jobBuilderFactory = jobBuilderFactory;
    }

    @Bean
    public Job job(Step firstStep, Step secondStep, Step finalStep) {
        return jobBuilderFactory.get(JOB_NAME)
                .start(firstStep) //firstStep is skipped
                .next(secondStep)
                .next(finalStep)
                .build();
    }

}
```

## Controlling Step Flow

> The failure of a Step doesn't necessarily mean that the Job should fail. Furthermore, there may be more than one type of 'success' which determines which Step should be executed next. Depending upon how a group of Steps is configured, certain steps may not even be processed at all.

- example code
```java
@Bean
public Job job(Step firstStep, Step secondStep, Step finalStep) {
    return jobBuilderFactory.get(JOB_NAME)
            .start(firstStep)
            .next(secondStep)
            .next(finalStep)
            .build();
}
```

## @StepScope
> A spring batch StepScope object is one which is unique to a specific step and not a singleton. As you probably know, the default bean scope in Spring is a singleton. But by specifying a spring batch component being StepScope means that Spring Batch will use the spring container to instantiate a new instance of that component for each step execution.

> This is often useful for doing parameter late binding where a parameter may be specified either at the StepContext or the JobExecutionContext level and needs to be substituted for a placeholder, much like your example with the filename requirement.

> Another useful reason to use StepScope is when you decide to reuse the same component in parallel steps. If the component manages any internal state, its important that it be StepScope based so that one thread does not impair the state managed by another thread (e.g, each thread of a given step has its own instance of the StepScope component).
