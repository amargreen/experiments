here's a step-by-step plan and the code for a basic Spring Batch application that reads from a database, processes the data, and writes to a file.
  Create a new Spring Boot project and add the necessary dependencies.
  Define a model class that represents the database table.
  Define a RowMapper to map the database rows to the model class.
  Define a Processor to process the data.
  Define a Writer to write the processed data to a file.
  Define a Job that uses the reader, processor, and writer.
  Configure a DataSource to connect to the database.
// Step 1: Add Spring Boot and Spring Batch dependencies in your build.gradle file
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    runtimeOnly 'com.h2database:h2'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

// Step 2: Define a model class
public class MyModel {
    private String field1;
    private String field2;
    // getters and setters
}

// Step 3: Define a RowMapper
public class MyModelRowMapper implements RowMapper<MyModel> {
    @Override
    public MyModel mapRow(ResultSet rs, int rowNum) throws SQLException {
        MyModel model = new MyModel();
        model.setField1(rs.getString("field1"));
        model.setField2(rs.getString("field2"));
        return model;
    }
}

// Step 4: Define a Processor
public class MyModelProcessor implements ItemProcessor<MyModel, MyModel> {
    @Override
    public MyModel process(MyModel item) throws Exception {
        // process item
        return item;
    }
}

// Step 5: Define a Writer
public class MyModelWriter implements ItemWriter<MyModel> {
    @Override
    public void write(List<? extends MyModel> items) throws Exception {
        // write items to file
    }
}

// Step 6: Define a Job
@Configuration
@EnableBatchProcessing
public class BatchConfiguration {
    @Autowired
    public JobBuilderFactory jobBuilderFactory;
    @Autowired
    public StepBuilderFactory stepBuilderFactory;

    @Bean
    public JdbcCursorItemReader<MyModel> reader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyModel>()
                .dataSource(dataSource)
                .name("myModelReader")
                .sql("SELECT field1, field2 FROM my_table")
                .rowMapper(new MyModelRowMapper())
                .build();
    }

    @Bean
    public MyModelProcessor processor() {
        return new MyModelProcessor();
    }

    @Bean
    public MyModelWriter writer() {
        return new MyModelWriter();
    }

    @Bean
    public Step myStep(JdbcCursorItemReader<MyModel> reader, MyModelProcessor processor, MyModelWriter writer) {
        return stepBuilderFactory.get("myStep")
                .<MyModel, MyModel> chunk(10)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job myJob(Step myStep) {
        return jobBuilderFactory.get("myJob")
                .incrementer(new RunIdIncrementer())
                .flow(myStep)
                .end()
                .build();
    }
}

// Step 7: Configure a DataSource in your application.properties file
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
This is a basic example and you'll need to modify it to suit your specific needs. For example, you'll need to replace MyModel, my_table, and the SQL query with your actual model class, table name, and query. You'll also need to implement the process method in MyModelProcessor and the write method in MyModelWriter to do the actual processing and writing.
-------------------------------------------------------------------------------------------------
To write an entry to a file and a database after processing a record, you can use Spring Batch's FlatFileItemWriter and JdbcBatchItemWriter respectively.

Here's how you can do it:

Define a FlatFileItemWriter to write to a file.
Define a JdbcBatchItemWriter to write to a database.
Define a CompositeItemWriter to use both writers.
Update the Job to use the CompositeItemWriter.
Here's the code:
// Step 1: Define a FlatFileItemWriter
@Bean
public FlatFileItemWriter<MyModel> fileWriter() {
    FlatFileItemWriter<MyModel> writer = new FlatFileItemWriter<>();
    writer.setResource(new FileSystemResource("output.txt"));
    writer.setLineAggregator(new DelimitedLineAggregator<MyModel>() {{
        setDelimiter(",");
        setFieldExtractor(new BeanWrapperFieldExtractor<MyModel>() {{
            setNames(new String[] {"field1", "field2"});
        }});
    }});
    return writer;
}

// Step 2: Define a JdbcBatchItemWriter
@Bean
public JdbcBatchItemWriter<MyModel> databaseWriter(DataSource dataSource) {
    JdbcBatchItemWriter<MyModel> writer = new JdbcBatchItemWriter<>();
    writer.setItemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>());
    writer.setSql("INSERT INTO my_table (field1, field2) VALUES (:field1, :field2)");
    writer.setDataSource(dataSource);
    return writer;
}

// Step 3: Define a CompositeItemWriter
@Bean
public CompositeItemWriter<MyModel> compositeWriter(FlatFileItemWriter<MyModel> fileWriter, JdbcBatchItemWriter<MyModel> databaseWriter) {
    CompositeItemWriter<MyModel> writer = new CompositeItemWriter<>();
    writer.setDelegates(Arrays.asList(fileWriter, databaseWriter));
    return writer;
}

// Step 4: Update the Job to use the CompositeItemWriter
@Bean
public Step myStep(JdbcCursorItemReader<MyModel> reader, MyModelProcessor processor, CompositeItemWriter<MyModel> writer) {
    return stepBuilderFactory.get("myStep")
            .<MyModel, MyModel> chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
}

This will write each processed record to both a file and a database. The FlatFileItemWriter writes to a file named output.txt in the current directory, and the JdbcBatchItemWriter writes to a table named my_table in the database. You'll need to replace these with your actual file name and table name. The SQL query in the JdbcBatchItemWriter also needs to be replaced with your actual query.
-------------------------------------------------------------------------------------------------------
To conditionally write to a file or a database, you can create a custom ItemWriter that delegates to either the FlatFileItemWriter or the JdbcBatchItemWriter based on a condition.

Here's how you can do it:

  Define a ConditionalItemWriter that writes to either a file or a database based on a condition.
  Update the Job to use the ConditionalItemWriter.
Here's the code:
// Step 1: Define a ConditionalItemWriter
public class ConditionalItemWriter implements ItemWriter<MyModel> {
    private FlatFileItemWriter<MyModel> fileWriter;
    private JdbcBatchItemWriter<MyModel> databaseWriter;

    public ConditionalItemWriter(FlatFileItemWriter<MyModel> fileWriter, JdbcBatchItemWriter<MyModel> databaseWriter) {
        this.fileWriter = fileWriter;
        this.databaseWriter = databaseWriter;
    }

    @Override
    public void write(List<? extends MyModel> items) throws Exception {
        List<MyModel> fileItems = new ArrayList<>();
        List<MyModel> databaseItems = new ArrayList<>();

        for (MyModel item : items) {
            if (/* your condition here */) {
                fileItems.add(item);
            } else {
                databaseItems.add(item);
            }
        }

        fileWriter.write(fileItems);
        databaseWriter.write(databaseItems);
    }
}

// Step 2: Update the Job to use the ConditionalItemWriter
@Bean
public Step myStep(JdbcCursorItemReader<MyModel> reader, MyModelProcessor processor, FlatFileItemWriter<MyModel> fileWriter, JdbcBatchItemWriter<MyModel> databaseWriter) {
    return stepBuilderFactory.get("myStep")
            .<MyModel, MyModel> chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(new ConditionalItemWriter(fileWriter, databaseWriter))
            .build();
}

This will write each error record to a file named error.txt in the current directory. You'll need to replace this with your actual file name. The MySkipListener writes to the error file when an error occurs during writing or processing. If you also want to handle errors during reading, you can implement the onSkipInRead method.
------------------------------------------------------------------------------
To write error records to a separate file, you can use a FlatFileItemWriter in a SkipListener. The SkipListener will be called when an error occurs during reading, processing, or writing.

Here's how you can do it:

Define a FlatFileItemWriter to write to an error file.
Define a SkipListener that writes error records to the error file.
Update the Job to use the SkipListener.
Here's the code:
// Step 1: Define a FlatFileItemWriter
@Bean
public FlatFileItemWriter<MyModel> errorFileWriter() {
    FlatFileItemWriter<MyModel> writer = new FlatFileItemWriter<>();
    writer.setResource(new FileSystemResource("error.txt"));
    writer.setLineAggregator(new DelimitedLineAggregator<MyModel>() {{
        setDelimiter(",");
        setFieldExtractor(new BeanWrapperFieldExtractor<MyModel>() {{
            setNames(new String[] {"field1", "field2"});
        }});
    }});
    return writer;
}

// Step 2: Define a SkipListener
public class MySkipListener implements SkipListener<MyModel, MyModel> {
    private FlatFileItemWriter<MyModel> errorFileWriter;

    public MySkipListener(FlatFileItemWriter<MyModel> errorFileWriter) {
        this.errorFileWriter = errorFileWriter;
    }

    @Override
    public void onSkipInRead(Throwable t) {
    }

    @Override
    public void onSkipInWrite(MyModel item, Throwable t) {
        writeError(item);
    }

    @Override
    public void onSkipInProcess(MyModel item, Throwable t) {
        writeError(item);
    }

    private void writeError(MyModel item) {
        try {
            errorFileWriter.open(new ExecutionContext());
            errorFileWriter.write(Collections.singletonList(item));
            errorFileWriter.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// Step 3: Update the Job to use the SkipListener
@Bean
public Step myStep(JdbcCursorItemReader<MyModel> reader, MyModelProcessor processor, ConditionalItemWriter writer, FlatFileItemWriter<MyModel> errorFileWriter) {
    return stepBuilderFactory.get("myStep")
            .<MyModel, MyModel> chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skip(Exception.class)
            .skipLimit(Integer.MAX_VALUE)
            .listener(new MySkipListener(errorFileWriter))
            .build();
}

This will write each error record to a file named error.txt in the current directory. You'll need to replace this with your actual file name. The MySkipListener writes to the error file when an error occurs during writing or processing. If you also want to handle errors during reading, you can implement the onSkipInRead method.
----------------------------------------------------

Here's how you can implement the same functionality in a legacy Spring application using XML configuration:

<!-- Step 1: Define a FlatFileItemWriter -->
<bean id="errorFileWriter" class="org.springframework.batch.item.file.FlatFileItemWriter">
    <property name="resource" value="file:./error.txt" />
    <property name="lineAggregator">
        <bean class="org.springframework.batch.item.file.transform.DelimitedLineAggregator">
            <property name="delimiter" value="," />
            <property name="fieldExtractor">
                <bean class="org.springframework.batch.item.file.transform.BeanWrapperFieldExtractor">
                    <property name="names" value="field1,field2" />
                </bean>
            </property>
        </bean>
    </property>
</bean>

<!-- Step 2: Define a SkipListener -->
<bean id="mySkipListener" class="com.example.MySkipListener">
    <constructor-arg ref="errorFileWriter" />
</bean>

<!-- Step 3: Update the Job to use the SkipListener -->
<job id="myJob" xmlns="http://www.springframework.org/schema/batch">
    <step id="myStep">
        <tasklet>
            <chunk reader="reader" processor="processor" writer="writer" commit-interval="10">
                <skippable-exception-classes>
                    <include class="java.lang.Exception" />
                </skippable-exception-classes>
            </chunk>
            <listeners>
                <listener ref="mySkipListener" />
            </listeners>
        </tasklet>
    </step>
</job>

You'll need to replace com.example.MySkipListener with the actual fully qualified class name of your SkipListener. The MySkipListener class should look like this:

public class MySkipListener implements SkipListener<MyModel, MyModel> {
    private FlatFileItemWriter<MyModel> errorFileWriter;

    public MySkipListener(FlatFileItemWriter<MyModel> errorFileWriter) {
        this.errorFileWriter = errorFileWriter;
    }

    @Override
    public void onSkipInRead(Throwable t) {
    }

    @Override
    public void onSkipInWrite(MyModel item, Throwable t) {
        writeError(item);
    }

    @Override
    public void onSkipInProcess(MyModel item, Throwable t) {
        writeError(item);
    }

    private void writeError(MyModel item) {
        try {
            errorFileWriter.open(new ExecutionContext());
            errorFileWriter.write(Collections.singletonList(item));
            errorFileWriter.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

This will write each error record to a file named error.txt in the current directory. You'll need to replace this with your actual file name. The MySkipListener writes to the error file when an error occurs during writing or processing. If you also want to handle errors during reading, you can implement the onSkipInRead method.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
This BatchJobTest class uses Mockito to create mock implementations of the ItemReader, ItemProcessor, and ItemWriter. The @Mock annotation creates the mock objects, and the @InjectMocks annotation injects the mock objects into the BatchJobRunner.

The testBatchJob method sets up the mock objects to return specific values when their methods are called, runs the batch job, and then verifies that the methods were called the expected number of times with the expected arguments.

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;

import static org.mockito.Mockito.*;

public class BatchJobTest {

    @Mock
    private ItemReader<Customer> reader;

    @Mock
    private ItemProcessor<Customer, Customer> processor;

    @Mock
    private ItemWriter<Customer> writer;

    @InjectMocks
    private BatchJobRunner batchJobRunner;

    @BeforeEach
    public void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
    public void testBatchJob() throws Exception {
        Customer customer = new Customer();
        customer.setId(1L);
        customer.setName("Test Customer");
        customer.setEmail("test@example.com");
        customer.setSubscriptionEndDate(new Date());
        customer.setAccessEnabled(true);

        when(reader.read()).thenReturn(customer, (Customer) null);
        when(processor.process(any(Customer.class))).thenReturn(customer);
        doNothing().when(writer).write(anyList());

        batchJobRunner.run();

        verify(reader, times(2)).read();
        verify(processor, times(1)).process(any(Customer.class));
        verify(writer, times(1)).write(anyList());
    }
}


import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import static org.mockito.Mockito.*;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring-batch-config.xml"})
public class BatchJobTest {

    @Mock
    private ItemReader<Customer> reader;

    @Mock
    private ItemProcessor<Customer, Customer> processor;

    @Mock
    private ItemWriter<Customer> writer;

    @InjectMocks
    private BatchJobRunner batchJobRunner;

    @Before
    public void setUp() {
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testBatchJob() throws Exception {
        Customer customer = new Customer();
        customer.setId(1L);
        customer.setName("Test Customer");
        customer.setEmail("test@example.com");
        customer.setSubscriptionEndDate(new Date());
        customer.setAccessEnabled(true);

        when(reader.read()).thenReturn(customer, (Customer) null);
        when(processor.process(any(Customer.class))).thenReturn(customer);
        doNothing().when(writer).write(anyList());

        batchJobRunner.run();

        verify(reader, times(2)).read();
        verify(processor, times(1)).process(any(Customer.class));
        verify(writer, times(1)).write(anyList());
    }
}

If you're using a legacy Spring version with XML context, you can use the SpringJUnit4ClassRunner and ContextConfiguration annotations to load the Spring context. Here's how you can modify the previous test to work with XML context:

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import static org.mockito.Mockito.*;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring-batch-config.xml"})
public class BatchJobTest {

    @Mock
    private ItemReader<Customer> reader;

    @Mock
    private ItemProcessor<Customer, Customer> processor;

    @Mock
    private ItemWriter<Customer> writer;

    @InjectMocks
    private BatchJobRunner batchJobRunner;

    @Before
    public void setUp() {
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testBatchJob() throws Exception {
        Customer customer = new Customer();
        customer.setId(1L);
        customer.setName("Test Customer");
        customer.setEmail("test@example.com");
        customer.setSubscriptionEndDate(new Date());
        customer.setAccessEnabled(true);

        when(reader.read()).thenReturn(customer, (Customer) null);
        when(processor.process(any(Customer.class))).thenReturn(customer);
        doNothing().when(writer).write(anyList());

        batchJobRunner.run();

        verify(reader, times(2)).read();
        verify(processor, times(1)).process(any(Customer.class));
        verify(writer, times(1)).write(anyList());
    }
}

In this example, SpringJUnit4ClassRunner is used to provide Spring testing functionality to standard JUnit tests. The @ContextConfiguration annotation is used to specify the XML configuration file that defines the Spring context. The MockitoAnnotations.initMocks(this) method is used to initialize the mock and spy objects.

@Query(value = "{ $or: [ { 'Date1': { $lt: ?0 }, 'Date2': { $gt: ?0 } }, { 'SubInquiries': { $elemMatch: { $or: [ { 'Date3': null }, { 'Date3': { $lt: ?0 } } ], $or: [ { 'Date4': null }, { 'Date4': { $gt: ?0 } } ] } } } ] }")

@Query(value = "{ $and: [ { 'Date1': { $lt: ?0 } }, { 'Date2': { $gt: ?0 } }, { $or: [ { 'SubInquiries': { $exists: false } }, { 'SubInquiries': { $size: 0 } }, { 'SubInquiries': { $elemMatch: { $or: [ { 'Date3': null }, { 'Date3': { $lt: ?0 } } ], $or: [ { 'Date4': null }, { 'Date4': { $gt: ?0 } } ] } } } ] } ] }", 
       fields = "{ 'name': 1, 'Date1': 1, 'Date2': 1, 'SubInquiries.$': 1 }")
List<InquiryType> findMatchingInquiriesWithSubInquiries(Date passedDate);

@Aggregation(pipeline = {
        "{ $match: { effectiveDate: { $lt: ?0 }, termiDate: { $gt: ?0 } } }",
        "{ $addFields: { subInquiries: { $filter: { input: '$subInquiries', as: 'sub', cond: { $and: [ { $lt: ['$$sub.effectiveDate', ?0] }, { $gt: ['$$sub.termiDate', ?0] } ] } } } } }"
    })


