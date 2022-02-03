---
title: "Acceptance testing using JBehave, Spring Framework and Maven"
date: 2014-05-29
categories: [Craftsmanship, JBehave, Spring]
---

This post documents acceptance testing best practices collected in regular projects I was working on.
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
 
But I found many traps when I was trying to apply acceptance tests automation in practice.

> Acceptance Testing is about collaboration not tools.

You will get much better results if you will collaborate closely with product owner, end users and customer. 
You could write test scenario only by yourself but perhaps you will fail. 
When you are able to work on test scenarios together, you could think about tools and automation.
Do not let that tools interfere in collaboration, all team members must be committed to acceptance tests contribution.

> Acceptance Testing needs to be done using user interface.

In most situation you don't need to implement tests using user interface. 

User interface tends to be changed frequently, business logic not so often. 
I don't want to change my tests when business logic stays unchanged, even if user interface has been changed significantly.

User interface tests are very fragile and slow. You will lost one of the automated tests advantages: fast and precise feedback loop.
It is really hard to setup and maintain the infrastructure for user interface testing.

> Everything should be tested.

Acceptance tests are mainly for happy path scenarios verification. 
Acceptance tests are expensive to maintain, so do not test corner cases, validation and error handling, on that level.
Focus only on the relevant assertions for the given scenario, do not verify everything only because you can.


## Project build organization

After bunch of theory it is time to show real code. Let's start with proper project organization. 
I found that acceptance testing is a cross cutting aspect of the application, and should be separated from the application code.
Acceptance tests build configuration is very specific and I don't want to clutter application build configuration. 
You can also utilize multi module project, to ensure that acceptance tests module is allowed to call application public API only.
This segregation applies only for acceptance testing, the best place for unit tests is still in an application module under `src/test` directory.

With _Maven_ (and other build tools like _Gradle_), application code and acceptance tests code can be located in separate modules.

Parent module 

``` xml 
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

Web application module

``` xml 
<project>
    <parent>
        <groupId>example</groupId>
        <artifactId>example-jbehave</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>example-jbehave-app</artifactId>
    <packaging>war</packaging>
</project>
``` 

Tests module

``` xml
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

The parent module is the best place to define common configuration properties inherited by child modules.

Configuration properties in parent module

``` xml
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

In the parent module you could also define _Spring Framework_ BOM (Bill Of Materials), to ensure consistent dependency management.
This is quite new _Spring Framework_ ecosystem feature.

Dependency management in parent module

``` xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-framework-bom</artifactId>
            <version>${spring.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        (...)
    </dependencies>    
<dependencyManagement>
```

Because I prefer _SLF4J_ over _Apache Commons Logging_, unwanted dependency is excluded globally from `spring-core` artifact.

Dependency management in parent module

``` xml
<dependencyManagement>
    <dependencies>
        (...)    
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
</dependencyManagement>
``` 

In the application module declare all application dependencies.
In real application the list will be much longer.

Dependency management in application module

``` xml
<dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
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

_Maven War Plugin_ must be configured specifically, classes (the content of the WEB-INF/classes directory) must be attached to the project as an additional artifact. 
Acceptance test module depends on this additional artifact. Set `attachClasses` property to `true`. 

War plugin configuration in application module

``` java
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <attachClasses>true</attachClasses>
    </configuration>
</plugin>
```

Alternatively you can use two separate modules for the application.
One _jar_ type with domain and infrastructure and separate _war_ type with web layer.
Then you would declare dependency to your _jar_ application module only. 
In my example I would keep it simple, and use single _war_ type module for all layers in the application.

In the tests module declare all dependencies as well. 
There is also an extra dependency to the application module, additional _jar_ type artifact generated by _Maven War Plugin_.
The last two dependencies of `zip` type are needed to generate _JBehave_ tests report.

Dependency management in tests module

``` xml
<dependencies>
        <dependency>
            <groupId>${project.groupId}</groupId>
            <artifactId>example-jbehave-app</artifactId>
            <version>${project.version}</version>
            <classifier>classes</classifier>
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
_Surefire_ is configured to execute test scenarios under `example/jbehave/tests/stories` package.

Surefire plugin configuration in tests module

```xml
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

JBehave plugin configuration in tests module

``` xml
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
Do not use my shopping basket implementation on production, it is only for this post educational purposes :-)
 
The application is composed from three main packages: `domain`, `infrastructure` and `web`. 
This convention comes from Domain Driven Design, you can read more in my post [DDD Architecture Summary](http://mkuthan.github.io/blog/2013/11/04/ddd-architecture-summary/).

Each package is configured using _Spring Framework_ annotation support. 
In general you should keep the configuration as modular as possible. 
It is very important for testing, with modular configuration you can load only needed context and speed up tests execution.

DomainConfiguration.java

``` java 
@Configuration
@ComponentScan
public class DomainConfiguration {
}
```

InfrastructureConfiguration.java

``` java
@Configuration
@ComponentScan
public class InfrastructureConfiguration {
}
```

WebConfiguration.java

``` java
@Configuration
@ComponentScan
public class WebConfiguration {
}
```

If you are interested in application functionality, go to the [source code](https://github.com/mkuthan/example-jbehave/tree/master/example-jbehave-app/src/main/java/example/jbehave/app).
The application is really simple, just old plain Java.

Much more interesting is _Spring Framework_ configuration in tests module. 
First the meta annotation for acceptance tests is defined. 
This is a new way to avoid repetition in tests definition, introduced in _Spring Framework_ recently.

AcceptanceTest.java

``` java
@ContextConfiguration(classes = AcceptanceTestsConfiguration.class)
@ImportResource({"classpath:/application.properties", "classpath:/tests.properties"})
@ActiveProfiles("tests")
@DirtiesContext
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface AcceptanceTest {
}
```

1. Tests configuration is loaded, again using Java config instead of XML.
2. Load application properties and overwrite defaults using tests properties if needed.
3. Activate some special _Spring Framework_ profile(s). Another way to customize tests configuration.
4. Acceptance tests have side effects typically. Reload context before every story execution. 

The `AcceptanceTestsConfiguration` class is again very simple. 
It imports application configurations: domain and infrastructure. 
Because we will implement acceptance tests using service layer, we don't need to load web module or run web container.

AcceptanceTestsConfiguration.java

``` java
@Configuration
@Import({DomainConfiguration.class, InfrastructureConfiguration.class})
@ComponentScan
public class AcceptanceTestsConfiguration {
}
```

Meta annotation support is also used to define very specific annotations, one for _JBehave_ test steps, second for _JBehave_ converters.
Well crafted annotations are better than generic `@Component`, even if they do not provide additional features. 

Steps.java

``` java
@Target(value = ElementType.TYPE)
@Retention(value = RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Steps {
}
```

Converter.java

``` java
@Target(value = ElementType.TYPE)
@Retention(value = RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Converter {
}
```

## JBehave configuration

The last infrastructure element in tests module is a base class for stories. 
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
The most useful configuration is used with some customizations. 
Let's check what kind of customization are applied.

``` java
private StoryPathResolver storyPathResolver() {
    return new UnderscoredCamelCaseResolver();
}
```

The story path resolver is responsible for resolving story based on test class name. 
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
With reusable steps you will find, that writing next user stories are much easier and faster. 
You can just use existing steps implementation to define new user story.

For example steps for product catalog and product prices are defined in `SharedSteps` class.
The repositories are used to manage products and prices. 
In real application, you should use application service and it's public API instead of direct access to the repositories.
Please think about steps implementation complexity, if we would need to use user interface, instead of repositories or service API.
 
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
You will have to implement custom converters but it is much more convenient to use well defined API, instead of dozen of `String` based values.

``` java 
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

The shopping cart related steps are implemented in `ShoppingCartSteps` class. 

``` java
@Steps
public class ShoppingCartSteps {

    @Autowired
    private ShoppingCartService shoppingCartService;

    @Autowired
    private ProductDao productRepository;

    @Given("empty shopping cart")
    public void emptyShoppingCart() {
        shoppingCartService.createEmptyShoppingCart();
    }

    @When("products are added to the shopping cart: $rows")
    public void addProducts(List<ShoppingCartRow> rows) {
        for (ShoppingCartRow row : rows) {
            Product product = productRepository.findByName(row.getProductName());
            shoppingCartService.addProductToShoppingCart(product.getSku(), row.getQuantity());
        }
    }

    @Then("shopping cart is empty")
    public void isEmpty() {
        ShoppingCart shoppingCart = shoppingCartService.getShoppingCart();
        assertEquals(0, shoppingCart.numberOfItems());
    }

    @Then("the number of products in shopping cart is $numberOfItems")
    public void numberOfItems(int numberOfItems) {
        ShoppingCart shoppingCart = shoppingCartService.getShoppingCart();
        assertEquals(numberOfItems, shoppingCart.numberOfItems());
    }

    @Then("total price is $price")
    @Pending
    public void totalPrice(Money price) {
        // TODO: implement missing functionality and enable step
    }
}
```

There are two interesting elements: 

* Last step annotated with `@Pending` annotation.
* `ShoppingCartRow` class used to defined products added to the cart.

Typically user story is prepared before implementation. 
In this situation you will have several pending steps, slowly implemented during the sprint.
Pending step does not mean that acceptance tests have failed, it only means that functionality has not been implemented yet.

`ShoppingCartRow` is a simple bean prepared for tabular parameters definition in the story. Do you remember this step?

```
When products are added to the shopping cart:
|PRODUCT                 |QTY|
|Domain Driven Design    |  1|
|Specification By Example|  2|
```

Basket presented in tabular form is much easier to read than if it would be defined line by line. 
To use this kind of parameter you have to prepare a class with a few annotations.

``` java
@AsParameters
public static class ShoppingCartRow {

    @Parameter(name = "PRODUCT")
    private String productName;

    @Parameter(name = "QTY")
    private Integer quantity;

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }
}
```

_JBehave_ uses this annotated class to convert table row from user story to Java object.

## Running tests

The last part of this post is about running tests. 

For every user story definition, one test class is defined. 
The test class is only the marker and does not define any logic.

``` java
@RunWith(SpringJUnit4ClassRunner.class)
@AcceptanceTest
public class LearnJbehaveStory extends AbstractSpringJBehaveStory {
}
```

The test can be executed directly from your favourite IDE, at least if the IDE provides support for JUnit runner.

The second way to execute tests is to use _Maven_ from command line. 
As long as tests are executed by regular _Maven Surefire Plugin_, the tests are executed exactly the same way like any other tests.
Run the following command from the project parent directory. 
_Maven_ builds the application module first, add the classes to the reactor classpath and then execute acceptance tests from tests module. 

``` console
mvn -pl example-jbehave-tests -am test
```

The convenient way to execute single test from command line is to use regular Java property `-Dtest` recognized by _Surefire_.

``` console
mvn -pl example-jbehave-tests -am test -Dtest=LearnJbehaveStory
...
```

## Summary

In the post I presented the most important elements from example project.
The complete project is hosted on [GitHub](https://github.com/mkuthan/example-jbehave), you can clone/fork the project and do some experiments by yourself.
I configure [Travis](https://travis-ci.org/mkuthan/example-jbehave) continuous integration build to ensure that the project really works.
