#### Synopsis

This page describes guidelines to migrate the code written on top of ACS to support the latest libraries included in 6.0 release.

Content:
* [Hibernate](#hibernate)
* [Quartz](#quartz)
* [Spring](#spring)
* [Jackson](#jackson)

#### Hibernate

Hibernate was completely remove from the product. The dialect detection is now part of [alfresco-repository](https://github.com/Alfresco/alfresco-repository) project.
The dialect names have changed as well. The new names are:
* org.alfresco.repo.domain.dialect.MariaDBDialect
* org.alfresco.repo.domain.dialect.MySQLInnoDBDialect
* org.alfresco.repo.domain.dialect.PostgreSQLDialect
* org.alfresco.repo.domain.dialect.DB2Dialect (enterprise only, EOLed)
* org.alfresco.repo.domain.dialect.Oracle9Dialect (enterprise only)
* org.alfresco.repo.domain.dialect.SQLServerDialect (enterprise only)

They all extend `org.alfresco.repo.domain.dialect.Dialect`.
All of the hibernate related properties were removed from *repository.properties*.
`org.springframework.jdbc.datasource.DataSourceTransactionManager` is now used as the Transaction manager implementation.

#### Quartz

[Quartz library](http://www.quartz-scheduler.org) was updated to 2.3.0.

As the APIs has changed the trigger and jobs are described in a different way.

In conjunction with Spring 5 upgrade (see below for details) the jobs and triggers can be configured in as in the following examples.

Before:
```xml
    <bean id="feedCleanerJobDetail" class="org.springframework.scheduling.quartz.JobDetailBean">
        <property name="jobClass">
            <value>org.alfresco.repo.activities.feed.cleanup.FeedCleanupJob</value>
        </property>
        <property name="jobDataAsMap">
            <map>
                <entry key="feedCleaner">
                    <ref bean="feedCleaner" />
                </entry>
            </map>
        </property>
    </bean>

    <bean id="feedCleanerTrigger" class="org.alfresco.util.CronTriggerBean">
        <property name="jobDetail" ref="feedCleanerJobDetail" />
        <property name="scheduler" ref="schedulerFactory" />
        <property name="cronExpression" value="${activities.feed.cleaner.cronExpression}" />
        <property name="enabled" value="${activities.feed.cleaner.enabled}" />
        <property name="startDelayMinutes">
            <value>${activities.feed.cleaner.startDelayMins}</value>
        </property>
    </bean>
```
Now:
```xml
    <bean id="feedCleanerSchedulerAccessor" class="org.alfresco.schedule.AlfrescoSchedulerAccessorBean">
        <property name="scheduler" ref="schedulerFactory"/>
        <property name="triggers">
            <list>
                <bean id="feedCleanerTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
                    <property name="cronExpression" value="${activities.feed.cleaner.cronExpression}"/>
                    <property name="startDelay" value="${activities.feed.cleaner.startDelayMilliseconds}"/>
                    <property name="jobDetail" ref="feedCleanerJobDetail"/>
                </bean>
            </list>
        </property>
        <property name="enabled" value="${activities.feed.cleaner.enabled}" />
    </bean>

    <bean id="feedCleanerJobDetail" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
        <property name="jobClass" value="org.alfresco.repo.activities.feed.cleanup.FeedCleanupJob"/>
        <property name="jobDataAsMap">
            <map>
                <entry key="feedCleaner" value-ref="feedCleaner"/>
            </map>
        </property>
    </bean>
```

Please note that the job detail is injected inside the trigger and the trigger is passed to scheduler. Previously the scheduler could be injected in the trigger directly. Also note that neither the Spring's *org.springframework.scheduling.quartz.SchedulerAccessorBean* nor the trigger bean definition has *enabled* flag. If it is required to control the scheduling of the trigger via properties use *org.alfresco.schedule.AlfrescoSchedulerAccessorBean* instead of Spring's accessor. As before all of the triggers have to be scheduled through one scheduler (*schedulerFactory* bean) if those need to be exposed in JMX MBeans (see *org.alfresco.enterprise.scheduler.MonitoredRAMJobStore* and associated classes).

Java APIs of trigger and job configuration have changed as well.

Before:
```java
        JobDetail jobDetail = new JobDetail(jobName, Scheduler.DEFAULT_GROUP, HeartBeatJob.class);
        String cronExpression = collector.getCronExpression();
        jobDetail.getJobDataMap().put(HeartBeatJob.COLLECTOR_KEY, collector);
        jobDetail.getJobDataMap().put(HeartBeatJob.DATA_SENDER_SERVICE_KEY, hbDataSenderService);
        jobDetail.getJobDataMap().put(HeartBeatJob.JOB_LOCK_SERVICE_KEY, jobLockService);
        CronTrigger cronTrigger = new CronTrigger(triggerName , Scheduler.DEFAULT_GROUP, cronExpression);
        scheduler.scheduleJob(jobDetail, cronTrigger);
```
After:
```java
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put(HeartBeatJob.COLLECTOR_KEY, collector);
        jobDataMap.put(HeartBeatJob.DATA_SENDER_SERVICE_KEY, hbDataSenderService);
        jobDataMap.put(HeartBeatJob.JOB_LOCK_SERVICE_KEY, jobLockService);
        JobDetail jobDetail = JobBuilder.newJob()
                .withIdentity(jobName)
                .usingJobData(jobDataMap)
                .ofType(HeartBeatJob.class)
                .build();
        String cronExpression = collector.getCronExpression();
        CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                .withIdentity(triggerName)
                .withSchedule(CronScheduleBuilder.cronSchedule(cronExpression))
                .build();
        scheduler.scheduleJob(jobDetail, cronTrigger);
```

#### Spring

Spring was updated to version 5.

While the handling of transaction-local resources changed quite a bit in Spring, extensions that make use of the pre- and post-commit phase callbacks (`TransactionListener`) should not experience any change in functionality; thorough testing of the features is recommended to ensure that there are no use cases that have been missed.

There are a few things that *need* to be changed.

The context files need to have an updated xsd declaration. For example:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```

The local reference to beans (like *idref local*) is now an error. The chnage might be like this:
```xml
-             <idref local="BlogService_transaction" />
+             <idref bean="BlogService_transaction" />
```

Singleton declaration should be now done via scope. The default scope is singleton. For example:

```xml
-    <bean id="userBootstrap-mt" parent="userBootstrap-base" singleton="false" />
+    <bean id="userBootstrap-mt" parent="userBootstrap-base" scope="prototype" />
```

*BaseSpringTest* class is now different. The test classes that are extending it need to specify the configuration via annotations, like:
* *@ContextConfiguration* needs to specify all the context files to use, for example:

```java
@ContextConfiguration({"classpath:alfresco/application-context.xml", "classpath:subsystem-test-context.xml"})
public class SubsystemsTest extends BaseSpringTest
```

* *@Transactional* needs to be included if the tests should run in txn. Note that the control of the transaction can be done via *org.springframework.test.context.transaction.TestTransaction*. This utility class exposes start and end methods. Keep in mind that by default *@Transactional* annotation enables txn that is marked for rollback. If it is required to have it committed use *TestTransaction.flagForCommit()* or *@Commit* in front of the test method.

```java
@Transactional
public class RuleServiceImplTest extends BaseRuleTest
{    
    @Commit
    @Test
    public void testCyclicAsyncRules() throws Exception
    {
    }
}
```
* Test setup and teardown should be done via JUnit annotations like *@Before*, *@After*, *@BeforeClass* and *@AfterClass*.

* Use *applicationContext* variable from *BaseSpringTest* to get the beans or *@Autowired* annotation.

```java
@Transactional
public class ComparePropertyValueEvaluatorTest extends BaseSpringTest
{
    @Autowired
    private DictionaryDAO dictionaryDAO;
}
```

#### Jackson

ACS 6 has newer [Jackson libraries](https://github.com/FasterXML) - 2.8.11(.1).

The old code (Jackson 1.x) will not compile against the new version. The migration mostly needs package names changed in imports section. There might be some behaviour changes as there is no one-to-one matching configuration for the *ObjectMapper*.  
The behaviour is expected to change even more with upgrades to 2.9+.