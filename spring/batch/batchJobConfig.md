# Spring Batch Configuring Job

## Restartability

> if you want to restart the Job with new JobInstance. you have to setting option like below

```xml
<job id="footballJob" restartable="false">
    ...
</job>
```
```java
Job job = new SimpleJob();
job.setRestartable(false);
```

## Java Config

> There are two components for the java based configuration: the @EnableBatchConfiguration annotation and two builders. @EnableBatchProcessing provides a base configuration for building batch jobs. Within this base configuration, an instance of StepScope is created in addition to a number of beans made available to be autowired:

- JobRepository : bean name "jobRepository"
- JobLauncher : bean name "jobLauncher"
- JobRegistry : bean name "jobRegistry"
- JobExplorer : bean name "jobExplorer"
- PlatformTransactionManager : bean name "transactionManager"
- JobBuilderFactory : bean name "jobBuilders"
- StepBuilderFactory : bean name "stepBuilders"

> Only one configuration class needs to have the @EnableBatchProcessing annotation. Once you have a class annotated with it, you will have all of the above available. -> Each Configuration class has @EnableBatchProcessing annotation.

## Configuring a JobRepository

 > As described in earlier, the JobRepository is used for basic CRUD operations of the various persisted domain objects within Spring Batch, such as JobExecution and StepExecution. It is required by many of the major framework features, such as the JobLauncher, Job, and Step. The batch namespace abstracts away many of the implementation details of the JobRepository implementations and their collaborators. However, there are still a few configuration options available:
```xml
<job-repository id="jobRepository"
    data-source="dataSource"
    transaction-manager="transactionManager"
    isolation-level-for-create="SERIALIZABLE"
    table-prefix="BATCH_"
	max-varchar-length="1000"/>
```
> None of the configuration options listed above are required except the id. If they are not set, the defaults shown above will be used. They are shown above for awareness purposes. The max-varchar-length defaults to 2500, which is the length of the long VARCHAR columns in the sample schema scripts

## In-Memeory Repository

> There are scenarios in which you may not want to persist your domain objects to the database. One reason may be speed; storing domain objects at each commit point takes extra time. Another reason may be that you just don't need to persist status for a particular job. For this reason, Spring batch provides an in-memory Map version of the job repository:
```java
@Configuration
@EnableBatchProcessing
public class JobConfig {

  @Bean(name = "mapJobRepositoryFactoryBean")
  public MapJobRepositoryFactoryBean createMapJobRepositoryFactory(PlatformTransactionManager transactionManager) {
    MapJobRepositoryFactoryBean mapJobRepositoryFactoryBean = new MapJobRepositoryFactoryBean();
    mapJobRepositoryFactoryBean.setTransactionManager(transactionManager);
  }

}
```
> Note that the in-memory repository is volatile and so does not allow restart between JVM instances. It also cannot guarantee that two job instances with the same parameters are launched simultaneously, and is not suitable for use in a multi-threaded Job, or a locally partitioned Step. So use the database version of the repository wherever you need those features. However it does require a transaction manager to be defined because there are rollback semantics within the repository, and because the business logic might still be transactional (e.g. RDBMS access). For testing purposes many people find the ResourcelessTransactionManager useful.

## Configuring a JobLauncher

> The most basic implementation of the JobLauncher interface is the SimpleJobLauncher. Its only required dependency is a JobRepository, in order to obtain an execution:
```java
@Bean
public SimpleJobLauncher jobLauncher(JobRepository jobRepository) {
  SimpleJobLauncher launcher = new SimpleJobLauncher();
  launcher.setJobRepository(jobRepository);
  return launcher;
}
```
> Once a JobExecution is obtained, it is passed to the execute method of Job, ultimately returning the JobExecution to the caller: The sequence is straightforward and works well when launched from a scheduler. However, issues arise when trying to launch from an HTTP request. In this scenario, the launching needs to be done asynchronously so that the SimpleJobLauncher returns immediately to its caller. This is because it is not good practice to keep an HTTP request open for the amount of time needed by long running processes such as batch. The SimpleJobLauncher can easily be configured to allow for this scenario by configuring a TaskExecutor:
```java
@Bean
publid SimpleJobLauncher jobLauncher(JobRegistry jobRepository) {
  SimpleJobLauncher launcher = new SimpleJobLauncher();
  launcher.setJobRepository(jobRepository);
  launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());
  return launcher;
}
```
> Any implementation of the spring TaskExecutor interface can be used to control how jobs are asynchronously executed.

## Aborting a Job

> A job execution which is FAILED can be restarted (if the Job is restartable). A job execution whose status is BatchStatus.ABANDONED will not be restarted by the framework. The ABANDONED status is also used in step executions to mark them as skippable in a restarted job execution: if a job is executing and encounters a step that has been marked ABANDONED in the previous failed job execution, it will move on to the next step
