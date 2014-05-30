---
layout: post
title: "Acceptance testing using JBehave, Spring Framework and Maven"
date: 2014-05-29
comments: true
categories: [jbehave, spring, maven]
---

This post documents acceptance testing best practices collected in regular project I was working on.
Best practices materialized into working [project](https://github.com/mkuthan/example-jbehave), using _Jbehave_, _Spring Framework_ and _Maven_. 

After the lecture you will know:

* How to implement automated acceptance tests and avoid common traps.
* How to organize project build using _Maven_.
* How to configure project and glue everything together using _Spring Framework_.
* How to write test scenarios using _JBehave_.
* Finally how to run tests from command line and from your favourite IDE.

## Automated acceptance tests

Automated acceptance test suite is a system documentation, the real single source of truth. 
The best documentation I've ever seen: always up-to-date, unambiguous and precise.
 
But I found many traps when I was trying to apply acceptance tests automation n practice.

> Acceptance Testing is about collaboration not tools

You will get much better results if you will collaborate closely with product owner, users and customer. 
You could write test scenario only by yourself but perhaps you will fail. 
When you are able to work on test scenarios together, you could think about tools and automation.
Do not let that tools interfere in collaboration.

> Acceptance Testing needs to be done using user interface

In most situation you don't need to implement tests using user interface. 

User interface tends to be changed frequently, business logic not so often. 
I don't want to change my tests when business logic stays unchanged even if user interface has been changed significantly.

User interface tests are very fragile and slow. You will lost one of the automated tests advantages: fast and precise feedback loop.
It is really hard to setup and maintain the infrastructure for user interface testing.

> Everything should be tested

Acceptance tests are mainly for happy path scenarios verification. 
Acceptance tests are expensive to maintain so do not test corner cases, validation and error handling on that level.
Focus only on the relevant assertions for the given scenario, do not verify everything only because you can.

## Project build organization

After bunch of theory it is time to show real code. Let's start with proper project organization. 
I found that acceptance testing is a cross cutting aspect to the application, and should be separated from the application code.
Tests configuration is very specific and I don't want to clutter application configuration.

With _Maven_ (and other build tools like _Gradle_) application code and acceptance tests code can be located in separate modules:
 
``` xml Parent module
<project>
    <groupId>example</groupId>
    <artifactId>example-jbehave</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>example-jbehave-app</module>
        <module>example-jbehave-tests</module>
    </modules>
</project>
```

``` xml Application module
<project>
    <parent>
        <groupId>example</groupId>
        <artifactId>example-jbehave</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>example-jbehave-app</artifactId>
    <packaging>jar</packaging>
</project>
``` 

``` xml Acceptance tests module
<project>
    <parent>
        <groupId>example</groupId>
        <artifactId>example-jbehave</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>example-jbehave-tests</artifactId>
    <packaging>jar</packaging>
</project>
``` 

The parent module is the best place to define common configuration properties:

``` xml Configuration properties in parent module
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>

    <jbehave.version>3.9.2</jbehave.version>
    <logback.version>1.1.1</logback.version>
    <slf4j.version>1.7.6</slf4j.version>
    <spring.version>4.0.5.RELEASE</spring.version>
</properties>
```

In the parent module you could also define _Spring Framework_ BOM (Bill Of Materials) to ensure consistent dependency management.
This is quite new _Spring Framework_ ecosystem feature.

``` xml Dependency management in parent module
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-framework-bom</artifactId>
            <version>${spring.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
``` 

In the application module declare all needed dependencies.

``` xml Dependency management in application module
<dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
            <scope>runtime</scope>
        </dependency>
    </dependencies>
</project>
```

In the tests module declare all needed dependencies. 
There is a dependency to the application module, the tests module needs to load application context.
The last two dependencies of `zip` type are needed to generate _JBehave_ tests execution report.

``` xml Dependency management in tests module
<dependencies>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>example-jbehave-app</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
        </dependency>

        <dependency>
            <groupId>org.jbehave</groupId>
            <artifactId>jbehave-core</artifactId>
            <version>${jbehave.version}</version>
        </dependency>

        <dependency>
            <groupId>org.jbehave</groupId>
            <artifactId>jbehave-spring</artifactId>
            <version>${jbehave.version}</version>
        </dependency>

        <dependency>
            <groupId>org.jbehave.site</groupId>
            <artifactId>jbehave-site-resources</artifactId>
            <version>3.1.1</version>
            <type>zip</type>
        </dependency>

        <dependency>
            <groupId>org.jbehave</groupId>
            <artifactId>jbehave-core</artifactId>
            <version>${jbehave.version}</version>
            <classifier>resources</classifier>
            <type>zip</type>
        </dependency>
    </dependencies>
```

Two _Maven_ plugins must be configured specifically in the tests module: `maven-surefire-plugin` and `jbehave-maven-plugin`.

Because we separated tests into it's own module, test classes might be located under `src/main` as first class citizen.
_Surefire_ will be executing test scenarios under `example/jbehave/tests/stories` package.

```xml Surefire configuration in tests module
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.17</version>
    <configuration>
        <testSourceDirectory>${basedir}/src/main/java/</testSourceDirectory>
        <testClassesDirectory>${project.build.directory}/classes/</testClassesDirectory>
        <includes>
            <include>example/jbehave/tests/stories/**/*.java</include>
        </includes>
    </configuration>
</plugin>
```

In my setup _JBehave_ plugin will be responsible only for unpacking resources used by tests report. 
I do not use plugin to run stories at all, I found better way to do that. It will be described later in the post.

``` xml JBehave configuration in tests module
<plugin>
    <groupId>org.jbehave</groupId>
    <artifactId>jbehave-maven-plugin</artifactId>
    <version>${jbehave.version}</version>
    <executions>
        <execution>
            <id>unpack-view-resources</id>
            <phase>generate-resources</phase>
            <goals>
                <goal>unpack-view-resources</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## Spring Framework configuration

The application implements shopping basket simplified functionality.
Do not use my shopping basket implementation on production, it is only for this post educational purposes.
 
The application is composed from two main packages: `domain` and `infrastructure`. 
This convention comes from Domain Driven Design, you can read more in my post DDD Architecture Summary.

Each package is configured using _Spring Framework_ annotation support. 
In general you should keep the configuration as modular as possible. 
It is very important for testing, with modular configuration you can load only needed context and speed up tests execution.

``` java DomainConfiguration.java
@Configuration
@ComponentScan
public class DomainConfiguration {
}
```

``` java InfrastructureConfiguration.java
@Configuration
@ComponentScan
public class InfrastructureConfiguration {
}
```

If you are interested in application functionality, go to the [source code](https://github.com/mkuthan/example-jbehave/tree/master/example-jbehave-app/src/main/java/example/jbehave/app).
The application is really simple, just old plain Java.

Much more interesting is _Spring Framework_ configuration in tests module. 
First the meta annotation for acceptance tests is defined. 
This is a new way to avoid repetition in tests definition introduced in _Spring Framework_ recently.

``` java AcceptanceTest.java
@ContextConfiguration(classes = AcceptanceTestsConfiguration.class)
@ImportResource({"classpath:/application.properties", "classpath:/tests.properties"})
@ActiveProfiles("tests")
@DirtiesContext
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface AcceptanceTest {
}
```

1. Tests configuration is loaded, again using Java config.
2. Load application properties and overwrite defaults using tests properties if needed.
3. Activate some special _Spring Framework_ profile(s). Another way to customize tests configuration.
4. Acceptance tests have side effects typically. Dirty context before every story execution. 

The `AcceptanceTestsConfiguration` class is again very simple. 
It imports application configurations: domain and infrastructure.

``` java AcceptanceTestsConfiguration.class
@Configuration
@Import({DomainConfiguration.class, InfrastructureConfiguration.class})
@ComponentScan
public class AcceptanceTestsConfiguration {
}
```

Meta annotation support is also used to define very specific annotations, one for _JBehave_ test steps, second for _JBehave_ converters.
Well crafted annotations are better than generic `@Component` even if they do not provide additional functionality. 

``` java Steps.java
@Target(value = ElementType.TYPE)
@Retention(value = RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Steps {
}
```

``` java Converter.java
@Target(value = ElementType.TYPE)
@Retention(value = RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Converter {
}
```

## JBehave configuration

The last tests infrastructure element is a base class for stories. 
_JBehave_ provides plenty of integration methods with _Spring Framework_ and I spent a lot of time to select the best one.

I have following requirements:

* The ability to run single story from my IDE.
* Meet Open Close Principle. When I add new story I do not want to modify any existing file. I want to add new one(s).
* Have a full control over _JBehave_ configuration.

To meet my requirements some base class for all tests must be defined.
I do not like the idea to use inheritance here but I did not find better way. 

Let me describe `AbstractSpringJBehaveStory` step by step:

``` java
public abstract class AbstractSpringJBehaveStory extends JUnitStory {
...
}
```

`JUnitStory` is a _JBehave_ class with single test to run single story. 
It means that any subclass of this class can be executed as regular _JUnit_ test.

``` java
private static final int STORY_TIMEOUT = 120;

public AbstractSpringJBehaveStory() {
    Embedder embedder = new Embedder();
    embedder.useEmbedderControls(embedderControls());
    embedder.useMetaFilters(Arrays.asList("-skip"));
    useEmbedder(embedder);
}

private EmbedderControls embedderControls() {
    return new EmbedderControls()
            .doIgnoreFailureInView(true)
            .useStoryTimeoutInSecs(STORY_TIMEOUT);
}
```

The constructor initialize _JBehave_ embedder, a fascade to embed _JBehave_ functionality in JUnit runner.

``` java
@Autowired
private ApplicationContext applicationContext;

@Override
public InjectableStepsFactory stepsFactory() {
    return new SpringStepsFactory(configuration(), applicationContext);
}
```

Configure _JBehave_ to load steps and converters from _Spring Framework_ context. 
What is also important, the steps and converters are managed by _Spring Framework_, you can inject whatever you want.

``` java
@Override
public Configuration configuration() {
    return new MostUsefulConfiguration()
            .useStoryPathResolver(storyPathResolver())
            .useStoryLoader(storyLoader())
            .useStoryReporterBuilder(storyReporterBuilder())
            .useParameterControls(parameterControls());
}
```

The `configuration` method is surprisingly responsible for _JBehave_ configuration. 
The most useful configuration is used with some customizations defined later.

``` java
private StoryPathResolver storyPathResolver() {
    return new UnderscoredCamelCaseResolver();
}
```

The story path resolver is reponsible for resolving story based on test class name. 
With `UnderscoredCamelCaseResolver` implementation, story `learn_jbehave_story.story` will be correlated with `LearnJbehaveStory.java` class.

``` java
private StoryLoader storyLoader() {
    return new LoadFromClasspath();
}
```

Stories will be resolved and loaded from the current classpath (from `src/main/resources` to be more specific).

``` java
private StoryReporterBuilder storyReporterBuilder() {
    return new StoryReporterBuilder()
            .withCodeLocation(CodeLocations.codeLocationFromClass(this.getClass()))
            .withPathResolver(new ResolveToPackagedName())
            .withFailureTrace(true)
            .withDefaultFormats()
            .withFormats(IDE_CONSOLE, TXT, HTML);
}
```

The configuration how the reports will look like. 
Nothing special, please refer to _JBehave_ reference documentation for more details.

``` java
private ParameterControls parameterControls() {
    return new ParameterControls()
            .useDelimiterNamedParameters(true);
}
```

The configuration how the steps parameters will be handled.

## Test scenarios definition

Test scenarios are rather straightforward, if you are familiar with BDD and Gherkin like syntax. If not please read 
[BDD Concepts](http://jbehave.org/reference/stable/concepts.html) short definition.

Look, in the scenarios there is nothing specific to the application user interface. 
It is not important how product price editor looks like, and how the shopping basket is presented.

```
Narrative:
In order to learn JBehave
As a tester
I want to define sample story for shopping cart

Lifecycle:
Before:
Given product Domain Driven Design with SKU 1234
And product Domain Driven Design price is 35 EUR

Given product Specification By Example with SKU 2345
And product Specification By Example price is 30 EUR

Scenario: Empty shopping cart

Given empty shopping cart
Then shopping cart is empty

Scenario: Products are added to empty shopping cart

Given empty shopping cart
When products are added to the shopping cart:
|PRODUCT                 |QTY|
|Domain Driven Design    |  1|
|Specification By Example|  2|

Then the number of products in shopping cart is 2
And total price is 95 EUR
```

## Test steps implementation

Test steps are implemented in Java classes annotated with `@Steps`. 
The common mistake is to develop steps only for single story. 
The steps should be reusable across many user stories if feasible. 
With reusable steps you will find that writing next user stories are much cheaper. 
You can just use existing steps implementation to define new user story.

For example steps for product catalog and product prices are defined in `SharedSteps` class.
The repositories are used to manage products and prices. 
In real application, perhaps you should use application service and it's public API.
Please imagine steps implementation complexity using user interface. Even with design (anti-)pattern like Page Objects.
 
``` java
@Steps
public class SharedSteps {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private PriceRepository priceRepository;

    @Given("product $name with SKU $sku")
    public void product(String name, StockKeepingUnit sku) {
        productRepository.save(new Product(sku, name));
    }

    @Given("product $name price is $price")
    public void price(String name, Money price) {
        Product product = productRepository.findByName(name);
        priceRepository.save(product.getSku(), price);
    }
}
```

You could ask, how does _JBehave_ know about `StockKeepingUnit` and `Money` classes?
You will have provide converters but it is much more convenient to use well defined API instead of dozen of `String` based values.

``` java MoneyConverter.class
@Converter
public class MoneyConverter {

    @AsParameterConverter
    public Money convertPercent(String value) {
        if (StringUtils.isEmpty(value)) {
            return null;
        }

        String[] tokens = value.split("\\s");
        if (tokens.length != 2) {
            throw new ParameterConverters.ParameterConvertionFailed("Expected 2 tokens (amount and currency) but got " + tokens.length + ", value: " + value + ".");
        }

        return new Money(tokens[0], tokens[1]);
    }
}
```

The class `MoneyConverter` is annotated with `@Converter` annotation defined before. 
`StringUtils` is a utility class from _Spring Framework_, look at the API documentation how many helpful utils classes are implemented in the framework.
If the value cannot be converted, _JBehave_ `ParameterConvertionFailed` exception is thrown.

## Running tests

For every user story definition, one test class is defined. The test class is only the marker and does not define any logic.
I could move `@RunWith` and `@AcceptanceTest` to the superclass but I want to define them directly in the test class. 
I don't want to look into superclass to understand the current test class.

``` java LearnJBehaveStory.java
@RunWith(SpringJUnit4ClassRunner.class)
@AcceptanceTest
public class LearnJbehaveStory extends AbstractSpringJBehaveStory {
}
```

The test can be executed directly from your favourite IDE, at least if the IDE provides support for JUnit runner.

The second way to execute tests is to use _Maven_ from command line. 
As long as tests are executed by regular _Maven Surefire Plugin_ the tests are executed exactly the same way like any other tests.

``` console
mvn -pl example-jbehave-tests -am test
[INFO] Scanning for projects...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO] 
[INFO] example-jbehave
[INFO] example-jbehave-app
[INFO] example-jbehave-tests
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building example-jbehave 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building example-jbehave-app 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:2.5:resources (default-resources) @ example-jbehave-app ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] 
[INFO] --- maven-compiler-plugin:2.3.2:compile (default-compile) @ example-jbehave-app ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-resources-plugin:2.5:testResources (default-testResources) @ example-jbehave-app ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marcin/projects/example-jbehave/example-jbehave-app/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:2.3.2:testCompile (default-testCompile) @ example-jbehave-app ---
[INFO] No sources to compile
[INFO] 
[INFO] --- maven-surefire-plugin:2.10:test (default-test) @ example-jbehave-app ---
[INFO] No tests to run.
[INFO] Surefire report directory: /home/marcin/projects/example-jbehave/example-jbehave-app/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------

Results :

Tests run: 0, Failures: 0, Errors: 0, Skipped: 0

[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building example-jbehave-tests 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- jbehave-maven-plugin:3.9.2:unpack-view-resources (unpack-view-resources) @ example-jbehave-tests ---
[INFO] Unpacked /home/marcin/.m2/repository/org/jbehave/jbehave-core/3.9.2/jbehave-core-3.9.2-resources.zip to /home/marcin/projects/example-jbehave/example-jbehave-tests/target/jbehave/view
[INFO] Unpacked /home/marcin/.m2/repository/org/jbehave/site/jbehave-site-resources/3.1.1/jbehave-site-resources-3.1.1.zip to /home/marcin/projects/example-jbehave/example-jbehave-tests/target/jbehave/view
[INFO] 
[INFO] --- maven-resources-plugin:2.5:resources (default-resources) @ example-jbehave-tests ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 3 resources
[INFO] 
[INFO] --- maven-compiler-plugin:2.3.2:compile (default-compile) @ example-jbehave-tests ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-resources-plugin:2.5:testResources (default-testResources) @ example-jbehave-tests ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/marcin/projects/example-jbehave/example-jbehave-tests/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:2.3.2:testCompile (default-testCompile) @ example-jbehave-tests ---
[INFO] No sources to compile
[INFO] 
[INFO] --- maven-surefire-plugin:2.17:test (default-test) @ example-jbehave-tests ---
[INFO] Surefire report directory: /home/marcin/projects/example-jbehave/example-jbehave-tests/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running example.jbehave.tests.stories.LearnJbehaveStory
09:40:44.511 [main] INFO  o.s.test.context.TestContextManager - Could not instantiate TestExecutionListener [org.springframework.test.context.web.ServletTestExecutionListener]. Specify custom listener classes or make the default listener classes (and their required dependencies) available. Offending class: [javax/servlet/ServletContext]
09:40:44.523 [main] INFO  o.s.test.context.TestContextManager - Could not instantiate TestExecutionListener [org.springframework.test.context.transaction.TransactionalTestExecutionListener]. Specify custom listener classes or make the default listener classes (and their required dependencies) available. Offending class: [org/springframework/transaction/interceptor/TransactionAttribute]
09:40:44.918 [main] INFO  o.s.c.s.GenericApplicationContext - Refreshing org.springframework.context.support.GenericApplicationContext@3234e239: startup date [Fri May 30 09:40:44 CEST 2014]; root of context hierarchy
Processing system properties {}
Using controls EmbedderControls[batch=false,skip=false,generateViewAfterStories=true,ignoreFailureInStories=false,ignoreFailureInView=true,verboseFailures=false,verboseFiltering=false,storyTimeoutInSecs=120,failOnStoryTimeout=false,threads=1]
Running story example/jbehave/tests/stories/learn_jbehave_story.story
Generating reports view to '/home/marcin/projects/example-jbehave/example-jbehave-tests/target/jbehave' using formats '[stats, ide_console, txt, html]' and view properties '{navigator=ftl/jbehave-navigator.ftl, views=ftl/jbehave-views.ftl, reports=ftl/jbehave-reports-with-totals.ftl, nonDecorated=ftl/jbehave-report-non-decorated.ftl, decorated=ftl/jbehave-report-decorated.ftl, maps=ftl/jbehave-maps.ftl}'
Reports view generated with 1 stories (of which 1 pending) containing 2 scenarios (of which 1 pending)
09:40:46.980 [main] INFO  o.s.c.s.GenericApplicationContext - Closing org.springframework.context.support.GenericApplicationContext@3234e239: startup date [Fri May 30 09:40:44 CEST 2014]; root of context hierarchy
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.619 sec - in example.jbehave.tests.stories.LearnJbehaveStory

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] example-jbehave ................................... SUCCESS [0.005s]
[INFO] example-jbehave-app ............................... SUCCESS [2.637s]
[INFO] example-jbehave-tests ............................. SUCCESS [6.153s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 9.173s
[INFO] Finished at: Fri May 30 09:40:47 CEST 2014
[INFO] Final Memory: 8M/116M
[INFO] ------------------------------------------------------------------------ 
```

The convenient way to execute single test from command line is to use _Surefire_ `test` property.

``` console
mvn -pl example-jbehave-tests -am test -Dtest=LearnJbehaveStory
...
```
