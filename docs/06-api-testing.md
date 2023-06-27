第 6 章 RESTful API 测试
========================

API 测试属于集成测试的范畴，手动进行 API 测试有很多方法，例如：

-   使用 Curl 命令。

-   使用 Postman。

-   使用 Swagger 附带的 UI （用户界面）工具。

这些方法都有一个共同的问题，就是实现自动化的反复执行比较困难。比较好的做法是通过编码的方式进行 API 测试，最好放到代码仓库中以自动化的形式完成，并集成在 CI
（持续集成）流水线中。

在一些企业中有专门的 API 测试平台管理相关测试。这种方式的灵活性会稍弱并与企业内部的基础设施强关联，因此本章选用的工具都是基于开源软件实现的。

本章的目标是介绍如何基于源代码编写可以维护的 API 测试，涵盖的内容有：

-   RESTful API 测试工具集。

-   第三方 API 的处理。

6.1 RESTful API 测试工具集
--------------------------

Spring Boot 的 spring-boot-starter-test
Starter 项目已经为我们准备了一套测试工具，不过为了让 API 测试更有效和便利，除了 Spring 测试套件以外，还需要准备一些额外的基础设施。例如，REST
Assured 可以作为 API 测试的基本框架，DbUnit 可以用来在多个测试之间隔离数据库的状态，WireMock 可以用来模拟第三方依赖的 API。这些工具可以帮助我们更好地完成测试用例的编写和运行，但在某些场景下不是必选的。

在 Spring 的测试体系下，MockMvc 也是 API 自动化测试的一个重要工具，但是我们常常并没有选用它作为 API 测试的工具。这是因为 API 测试已经是一个 E2E 测试了，API 测试关注的是服务是否能提供完整的功能，我们会希望启动完整的上下文来进行端到端的测试。

MockMvc 在前面章节介绍过，用来模拟 Web 服务器，以便让 Controller 上的注解可以被测试到，所以常常在分层的单元测试中用于验证 Controller 部分的逻辑。

更多的情况下，我们会选用专业的 API 自动化测试工具，比如 REST
Assured、Karate DSL 等。本书基于市场占有量选择了以 REST
Assured 为例详细讲解 API 测试。实际上，MockMvc 也可以配合 RESTAssured 使用。

### 6.1.1 REST Assured

REST Assured 由 Johan Haleby 在 Jayway 公司创建。与 MockMvo
不同的是，REST Assured
更像是 Java 中关于测试的领域特定语言（DSL）。它提供了一套链式的 API 来编写 Given、When、Then 风格的测试。示例代码如下：

```java
given().
    param("key1", "value1").
    param("key2", "value2").
when().
    post("/somewhere").
then().
    body(containsString("OK"));
```

在上述代码中，通过流式的风格声明了 Given、When、Then 这三种语句块，让 API 测试具有与单元测试类似的风格。

REST
Assured 的优秀之处还在于提供了 xml-path、json-path 这两个强大的断言库，它可以直接断言返回的数据结构，使用起来非常方便。json-path 也被吸收到 spring-boot-starter-test 中了，基本已成为断言 JSON 数据的必备工具。

REST
Assured 只是一个单独的测试套件，内含一个 HTTP 客户端，测试时需要启动全量的 Web 服务，且需要依赖 @SpringBootTest 注解启动完整的应用上下文，并创建一个随机端口，避免测试被干扰。如果觉得 Web 服务器启动得太慢，RESTAssured 也可以结合 MockMvc 来使用。

创建一个 Maven 项目或者模块，除了要引人 Spring 生态下基本的依赖外，还需要引入下面的依赖包：

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>4.4.0</version>
    <scope>test</scope>
</dependency>
```

RESTAssured 的配置非常简单，使用下面的脚本即可完成初始化：

```java
@SpringBootTest(webEnvironment = RANDOM_PORT, classes = {Application.class})

....

@LocalServerPort
private int port;

@BeforeEach
public void setup() {
  System.out.println("port:" + port);

  RestAssured.port = port;
  RestAssured.basePath = "/api";
  RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
}
```

@LocalServerPort 注解可以读取 @SpringBootTest 在启动时创建的随机端口，并将其用于后续的测试中。在 setup
方法中，我们为 REST
Assured 配置了端口、API 前缀，并打开了一个特性开关，以便测试失败时打印出请求和返回的详情。

上面的代码就是启用 REST
Assured 的全部配置，后面会在测试实例中给出完整的使用示例。如果需要实现
MockMvc 的效果，可以使用 spring-mock-mve 库中的 RestAssuredMockMvc
类进行桥接，加快测试速度。

### 6.1.2 MariaDB

上一章讨论过单元测试中关于基础设施的问题，我们知道，涉及数据库的测试可以使用内嵌的数据库比如 H2 来实现相应的模拟。但是 H2 的功能比较有限，有时候不一定能满足需要。比如，在做 E2E 测试时，H2 无法模拟出与 MySQL 同样的功能，SQL 语法上也与其存在一定的差异。这时，可以使用 MariaDB 来模拟 MySQL，并通过 MariaDB4j 这个工具从测试中启动 MariaDB。

MariaDB 是数据库管理系统 MySQL 的一个分支，主要由开源社区维护，在一定程度上可以将其看作 MySQL。

MariaDB4j 实际上只是一个启动器，真正的数据库还是会由与操作系统相关的二进制包启动。为了使用随机的数据库端口，我们不再使用 Spring
Boot 的自动配置，而是直接给上下文配置一个 DataSource。来看个示例，老规矩，先引入一个依赖：

```java
<dependency>
    <groupId>ch.vorburger.mariaDB4j</groupId>
    <artifactId>mariaDB4j</artifactId>
    <version>2.4.0</version>
</dependency>
```

然后在测试模块中引入一个配置类。在测试模块中定义的相关 Bean 也会被加载到 Spring
Boot 的上下文中，但是不会对业务代码产生影响。

```java
@Configuration
public class MariaDB4jSpringConfiguration {

    @Autowired
    private DataSourceProperties dataSourceProperties;

    @Bean
    public MariaDB4jSpringService mariaDB4j() {
        MariaDB4jSpringService mariaDB4jSpringService = new MariaDB4jSpringService();
        mariaDB4jSpringService.getConfiguration().addArg("--user=root");
        mariaDB4jSpringService.getConfiguration().addArg("--character-set-server=utf8");
        return mariaDB4jSpringService;
    }

    @Bean
    @Primary
    public DataSource dataSource() throws ManagedProcessException {
        String dbname = UUID.randomUUID().toString().substring(0, 8);
        mariaDB4j().getDB().createDB(dbname);
        return DataSourceBuilder.create()
                .driverClassName(dataSourceProperties.getDriverClassName())
                .url(mariaDB4j().getConfiguration().getURL(dbname))
                .username(dataSourceProperties.getUsername())
                .password(dataSourceProperties.getPassword())
                .build();
    }
}
```

MariaDB4jSpringService 继承了 MariaDB4jService，并实现了 Lifecycle 接口，它是启动 MariaDB4jService 的原生 API，可以启动和关停数据库。MariaDB4jSpringService 实现了 Spring 的生命周期钩子，让数据库随着 Spring 的生命周期启动和关停。

DataSource
Bean 自定义了数据库名称，并获得了 MariaDB4j 中的默认配置，然后以此来创建数据源。在 DataSource 的配置中使用了 DataSourceProperties 的配置，该配置来源于我们在 YAML 文件（application.yml）中配置的数据库连接信息，这样可以将原本需要连接真实
MySQL 的数据源切换成内置的临时数据库。

### 6.1.3 DbUnit

通常情况下，我们会结合数据库来进行 E2E 测试，每执行一个测试用例，测试数据就有可能会污染数据库。对于这种情况，DbUnit 可以帮助我们在测试前暂存数据库的状态，并在测试后恢复，避免测试之间产生干扰。

如果我们的数据库中没有初始数据，可以直接做清空操作，则不需要 DbUnit 的帮助。但是，一般来说我们会使用 Flyway 管理数据库的表结构，而且为了开发方便，也会将一些初始数据同步写入。系统中总是需要初始化一些默认数据，例如超级管理员的账号和密码等，通过 Flyway 管理会非常方便。如果测试运行前数据库中有一些默认数据（而且是通过 Flyway 管理的），就不能通过简单地清空数据库来重置数据库状态，这时 DbUnit 就派上用场了。

按照惯例，将 DbUnit 加入项目中时，先引入需要的依赖包：

```xml
<dependency>
    <groupId>org.dbunit</groupId>
    <artifactId>dbunit</artifactId>
    <version>2.7.0</version>
</dependency>
```

上述代码中，DbUnit 的实现方式比较朴素，主要提供了一个 IDataSet 接口，它通过 CSV、XML、SQL 等实现方式把数据临时存储下来，如图 6-1 所示。

![图 6-1 DBUnit
暂存数据集的几种方式](./06-api-testing/media/image1.png)

图 6-1 DbUnit 暂存数据集的几种方式

DbUnit 还提供了一个工具类 DatabaseOperation，可以通过它来操作数据库和数据集，实现备份、还原和清空等功能。

下面是一个封装好的服务，将其加载到测试之前和之后的方法中即可。

```java
@Service
public class ResetDbService {

    private static IDatabaseConnection conn;

    @Autowired
    private DataSource dataSource;
    private File tempFile;
    
    // 备份数据状态的方法
    public void backup() throws Exception {
        this.getConnection();
        this.backupCustom();
    }

    // 恢复数据状态的方法
    public void rollback() throws Exception {
        this.reset();
        this.closeConnection();
    }

    protected void backupCustom() {
        try {
            QueryDataSet qds = new QueryDataSet(conn);
          
            // 指定需要备份的数据库表，也可以通过编写一个方法来获得数据库中的所有表
            qds.addTable("user");
            tempFile = new File("temp.xml");
            // 写入到 XML 文件中
            FlatXmlDataSet.write(qds, new FileWriter(tempFile), "UTF-8");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    void getConnection() throws DatabaseUnitException {
        conn = new DatabaseConnection(DataSourceUtils.getConnection(dataSource));
    }

    protected void reset() throws FileNotFoundException, DatabaseUnitException, SQLException {
        FlatXmlDataSetBuilder builder = new FlatXmlDataSetBuilder();
        builder.setColumnSensing(true);
        IDataSet ds = builder.build(new FileInputStream(tempFile));

        DatabaseOperation.CLEAN_INSERT.execute(conn, ds);
    }

    protected void closeConnection() throws SQLException {
        if (conn != null) {
            conn.close();
        }
    }
}
```

ResetDbService 服务提供了 backup、rollback 这两个方法。backup 会在测试启动时将当前数据库的状态暂存下来，测试完成时再调用 rollback 方法恢复，这样就能始终为每个测试提供一个一致的环境，也不需要手动对特定的数据进行清理，避免遗忘。

我们可以在测试的基类中组装这些准备、清理的工作，下面再介绍另外一种更灵活的组装方法。

6.2 API 测试实例
----------------

前面介绍了几种工具的使用，这里通过最小的实例将它们组装起来。为了减少篇幅，下面只贴上必要的代码，完整的代码可以在 GitHub 仓库中找到：[https://github.com/java-self-testing/java-self-testing-example/tree/master/api](https://github.com/java-self-testing/java-self-testing-example/tree/master/api)。

示例代码的目录结构如图 6-2 所示。

![图 6-2 API
测试示例](./06-api-testing/media/image2.png)

图 6-2 API 测试示例

下面是示例项目的依赖的包：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project
    xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.12</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>api</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.25</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>ch.vorburger.mariaDB4j</groupId>
            <artifactId>mariaDB4j</artifactId>
            <version>2.4.0</version>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>4.4.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.dbunit</groupId>
            <artifactId>dbunit</artifactId>
            <version>2.7.0</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <finalName>${project.artifactId}</finalName>
                    <outputDirectory>../package</outputDirectory>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

源码下面的包中实现了 2 个简单的 API，便于管理员添加和列出用户：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    UserService userService;

    @GetMapping("")
    public ResponseEntity<List<User>> getAll() {
        return ResponseEntity.ok(userService.listAll());
    }

    @PostMapping("")
    public ResponseEntity addUser(@RequestBody AddUserRequest addUserRequest) {
        User savedUser = userService.add(addUserRequest);
        return ResponseEntity.created(
                URI.create("/api/users/" + savedUser.getId())
        ).build();
    }
}
```

在测试目录中，有一个 TestBase
，是为所有测试类的基类，用于初始化测试上下文。示例代码如下：

```java
@SpringBootTest(webEnvironment = RANDOM_PORT, classes = {Application.class})
@TestExecutionListeners({
        DependencyInjectionTestExecutionListener.class,
        ResetDbListener.class,
})
public abstract class TestBase {

    @LocalServerPort
    private int port;
  
    @BeforeEach
    public void setUp() {
        System.out.println("port:" + port);

        RestAssured.port = port;
        RestAssured.basePath = "/api/";
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();
    }
}
```

@TestExecutionListeners 注解是 JUnit 的一种高级用法，可以注册测试生命周期函数，类似于 @BeforeEach 和 @AfterEach 的作用，但是更加简洁，使用该注解时需要实现 TestExecutionListener 接口。

下面介绍两个常用的 TestExecutionListener 接口的实现。

-   DependencylnjectionTestExecutionListener：可以对测试类中的依赖进行注入，如果没有这个类，就会加载不到 @LocalServerPort，因为它实际上是一个 @Value 注解的包装。

-   TransactionalTestExecutionListener：用于保证插入数据库中的测试数据在测试完成之后会进行事务回滚。也就是说，这个监听器可以将插入的数据删除，保证数据库干净。实际上，这个类能起到类似 DbUnit 的作用。但是，它俩又有所不同，DbUnit 可以完整地验证事务的结果，然后进行清理：TransactionalTestExecutionListener 则是让测试中的事务统统不提交。由于 API 测试是一个 E2E 测试，为了提高测试的准确性，需要尽量使用完整的测试方式，因此在某些场景下可以酌情使用 TransactionalTestExecutionListener。

下面定义一个 ResetDbListener，用来暂存和恢复数据库的状态：

```java
public class ResetDbListener extends AbstractTestExecutionListener {
    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public void beforeTestMethod(TestContext testContext) throws Exception {
        ResetDbService resetDbService =
                testContext.getApplicationContext().getBean(ResetDbService.class);
        resetDbService.backup();
    }

    @Override
    public void afterTestMethod(TestContext testContext) throws Exception {
        ResetDbService resetDbService =
                testContext.getApplicationContext().getBean(ResetDbService.class);
        resetDbService.rollback();
    }
}
```

由于 ResetDbListener 中不能使用依赖注入，因此要使用测试上下文来加载 ResetDbService。前面已经展示过 ResetDbService 的代码，我们也可以自己实现一个其他版本的 ResetDbService。

接下来就可以创建测试了，在 UserController 对应的测试目录下创建两个测试，让它们分别对应列出用户和添加用户这两个操作。示例代码如下：

```java
class UserControllerTest extends TestBase {
    @Test
    void should_list_all_users() {
        given()
                .contentType("application/json")
                .when()
                .get("/users")
                .then().statusCode(200);    
    }

    @Test
    void should_add_user() {
        given()
                .contentType("application/json")
                .body(Maps.of(
                        "name", "test-user",
                        "email", "test@email.com",
                        "password", "123456"
                ))
                .when()
                .post("/users")
                .then().statusCode(201);
    }
}
```

这是两个最基本的测试，由于只是演示，我们没有编写更多的测试用例。在这两个测试中，状态码的断言比较简单，例如在第二个测试中，验证了最终返回的是 201 状态码。但是，对返回值的断言稍显复杂，后面会专门讨论如何实现复杂的断言。

到目前为止，基于 REST
Assured 的 API 测试就完成了，这里面的每个测试里都只有一个非常简洁的链式结构，可维护性和阅读性都非常高。下面我们来展开看一下
API 测试的各个阶段。

以测试用例 should_add_user 为例，其测试的逻辑是提交一组数据给 / users
API，用于创建用户。given 方法是门面类 RestAssured 的一个方法，它创建了一个测试规格对象 RequestSpecification 的实例。基于这个实例，在每次调用新的方法后，都会返回一个新的实例用于链式调用。

这个调用链有以下几个明显的阶段，每个阶段返回的对象不同。

（1）Given 阶段

返回 RequestSpecification 实例，用于组织请求的数据。它提供了设置端口、提交数据、Query 参数、Session、请求头等一系列方法，方便构建 HTTP 数据体。

（2）When 阶段

when 方法与 given 方法所做的事情一样，只是它的语法更符合 BDD 风格。真正的 When
阶段是通过与 HTTP 动词（Method）同名的方法 get、put、post 等实现的。例如，调用 get 方法后会返回 Response 对象。

（3）Then 阶段

得到返回值后可以通过 Response 上的 getBody、getHeader 等方法直接获得数据并进行断言，然后中断链式调用。也可以使用 then 方法获取 ValidatableResponseOptions 对象，并将其传入各种校验器校验 Response 中的数据，校验的方法有 statusCode、body、header 等。

下面是一个获取返回结果的示例，这里基于从列表接口获得的数据进行断言：

```java
String body =  given()
         .contentType("application/json")
         .when()
         .get("/users").getBody().print();
 Assertions.assertEquals("[{"id":"admin-id","name":"admin","role":"ADMIN","email":"admin@test.com","password":"$2a$10$Q9xt3B2Ixe0tGnbCjVWAbunD4lYdf5PpMSYGyLNrD4S38FGUt4NMC","createdAt":"2020-12-16T21:45:41.147Z","updatedAt":"2020-12-16T21:45:41.147Z"}]", body);
```

但是如果要进一步对结果中的属性进行断言，需要先使用 JSON 解析库将字符串解析为 Java 对象，然后再对解析后的对象属性进行断言。如此一来，依赖代码就是冗长的。虽然 ValidatableResponseOptions 提供了直接在调用链上断言的方法，比如在 then 方法后面调用 statusCode(201)
方法来判断状态码，但若面对的是 body 方法中复杂的数据结构，就不能这么简单地完成校验了。

为了让断言更容易，可以借用 JsonPath 和 XmIPath 这两种工具，通过传入一个 JSON
路径表达式来访问数据中的特定节点，从而完成断言。在 then 方法后，可以基于 body 方法的各种重载配合 JSON
路径表达式完成简洁和高效的断言。

6.3 使用 JsonPath 断言
----------------------

由于越来越多的应用都开始使用 RESTful API（RESTful
API 的数据格式为 JSON），因此可以使用一种表达式（JsonPath）来进行断言。这里为了演示 JsonPath 的使用，将用户列表的接口修改为 Page 类型了，这样可以让返回结果变得更复杂一些。示例代码如下：

```java
// UserController
@GetMapping("")
public ResponseEntity<Page<User>> getAll() {
  return ResponseEntity.ok(userService.listAll());
}
// UserService
public Page<User> listAll() {
    return userRepository.findAll();
}
```

这时返回的数据结构会带上分页信息，相对复杂了一些，具体如下：

```java
{
    "content": [
        {
            "id": "admin-id",
            ...
        }
    ],
    "totalPages": 1,
    "totalElements": 1,
    "size": 15,
    "number": 0,
    "numberOfElements": 1
}
```

REST
Assured 提供的 body 方法可接收一个 JsonPath 和一个 Matcher。大部分情况下，将不同的 JsonPath 和 Matcher 组合使用就可以完成需要的断言。

下面是常见的断言示例：

-   断言属性。例如 totalPages 属性的值为 1，可以写为. body("$.totalPages",
    is(1))。

-   断言数组长度。例如 content 属性的内容长度为 1，可以写为. body("$.content.
    size()", is(1))。

-   断言数组中具体的属性。例如 content 属性中第一个元素的 name 属性值为 admin，可以写为. body("$.content[0].name",is("admin"))。

在 JsonPath 中提供了一些特殊的符号来匹配部分数据结构，可以把这些符号叫作操作符。比如，点号用于匹配对象的属性。表 6-1 列出了常用的操作符。

表 6-1 常用 JsonPath 操作符

| 操作符                                                       | 用途                               |
| ------------------------------------------------------------ | ---------------------------------- |
| $                                                            | 要查询的根元素，一般置于表达式之前 |
| @                                                            | 迭代器的当前元素                   |
| *                                                            | 通配符操作符，用于模糊匹配         |
| ..                                                           | 深度匹配，跨多层级匹配             |
| .<name>                                                      | 子属性匹配                         |
| ['<name>' (,'<name>')]                                      | 一般用于匹配对象的多个属性         |
| [<number> (, <number>)]  | 根据数组索引访问          | |[start:end]` | 对数组切片访问                     |

6.4 鉴权的处理
--------------

API 测试的鉴权处理常用如下方法：

-   针对 API 使用 Basic 鉴权。现实中，对于暴露给用户的 API 几乎都不会使用这种鉴权方式，但是服务之间鉴权使用得较为广泛。

-   针对 API 使用 Token 鉴权。每一个测试可以生成一次鉴权 Token，并在 Header 中使用。

-   针对 API 使用 Cookie 鉴权：使用 Session 方式传递鉴权信息，RESTAssured 提供的 SessionFilter 可以作为一种更方便的方法。

### 6.4.1 Basic 鉴权

认证信息都是使用 HTTP 头部来传递的，Basic 认证就是将用户名和口令拼接，并进行 Base64 编码，然后放到
Authorization 头信息中。

例如，用户名 "admin"，口令 "123456"，使用 "：" 拼接并编码后变为 YWRtaW46
MTIzNDU2。在 HTTP 包中发送的数据如下：

```java
Authorization: Basic YWRtaW46MTIzNDU2
```

可以手动使用 header 来完成鉴权：

```java
given().header("Authorization", "Basic YWRtaW46MTIzNDU2")
```

也可以使用快捷方法：

```java
given().auth().basic("admin", "123456")
```

如果需要全局增加鉴权，可以直接使用
RestAssured 类的全局属性，不过这会降低灵活性。示例代码如下：

```java
RestAssured.authentication = basic("admin", "123456");
```

### 6.4.2 Token 鉴权

Token
鉴权是最灵活的方式，与 Basic 鉴权类似，都是在 HTTP 请求的头部传入认证信息。一般使用 Token 鉴权时都会使用 Bearer 前缀，示例代码如下：

```java
Authorization: Bearer a1e1eb29-2733-4ce3-b2cc-4569df7fdf0e
```

与 Token 鉴权类似的还有 JWT 鉴权、OAuth 鉴权等。这些鉴权方式都是通过生成一种简单的凭证实现的，不同之处在于 JWT 是一种自包含凭证，它会将鉴权信息编码到 Token
中，而 OAuth 为分布式鉴权，需要使用 access_token 和 refresh
token 这两种 Token。

使用 Token 鉴权时，可以在测试中提前准备一个 Token，并在每个测试中复用。这里因为是在同一个项目中编写的测试用例，所以会在测试中直接使用源代码中的方法，这样更方便。示例代码如下：

```java
@Autowired
private AuthorizeService authorizeService;
private String token;

...  

@BeforeEach
public void setup() {  
    // 调用父类的初始化方法  
    super.setup();  
    // 设置一个默认测试用户  
    ...   
    this.token = authorizeService.login(User user);
}

@Test
public void test() {    
    given().contentType("application/json").header("Authorization","Bearer " + token);
    ...
}
```

### 6.4.3 Cookie 鉴权

对于一些使用 Cookie 鉴权的 API，可以通过暂存 Cookie 的方法来实现测试中的鉴权。Cookie 鉴权的原理是用户在初次访问页面或 API 时，服务器会生成一个 Session，此 Session 由 Session
ID 标识，并返回给用户在 Cookie 中存储。用户登录后，关联用户凭证信息到这个 Session
ID，只要用户每次请求都带上这个 ID，服务器就能识别。

由于 Cookie 的自动附加是浏览器的默认功能，因此使用这种方式时开发者不需要做太多工作。

我们可以在登录后抽取出 Cookie，然后在下次请求时带上，示例代码如下：

```java
// 1. 登录并留存 Cookies
Map<String, String> cookies = given()
  .contentType("application/json")
  .body(Maps.of("email", "test@email.com", "password", "123456"))
  .when().post("/authorizes")
  .then().statusCode(201).extract().cookies();

// 2. 使用 cookies 获取用户个人信息
given()
  .contentType("application/json")
  .cookies(cookies)
  .when().post("/authorizes/me")
  .then().statusCode(200);
```

RESTAssured 在 2.0.0 版本之后提供了一个快捷方式，即可以使用 SessionFilter 来保持会话，示例代码如下：

```java
SessionFilter sessionFilter = new SessionFilter();
given()
  .contentType("application/json")
  .filter(sessionFilter)
  .body(Maps.of("email", "test@email.com", "password", "123456"))
  .when().post("/authorizes")
  .then().statusCode(201);
```

当然，还有一种方法是在测试中关闭鉴权，不过这会让测试的价值降低，因此并不推荐。

6.5 文件处理
------------

有一些 API 需要上传文件，比如导入、上传图片等，因此对其进行测试时就需要处理相应的文件。下面介绍几种可以从测试目录的资源文件夹中读取文件的方法，这里不需要使用绝对路径。

假设在测试资源目录中有一个文件用于测试，其文件地址为 "classpath:file/test.pdf"。第一种读取文件的方法是使用 ClassLoader。这时可以直接使用类加载器来加载相应的文件。示例代码如下：

```java
ClassLoader classLoader = getClass().getClassLoader();
File file = new File(classLoader.getResource("file/test.pdf").getFile());
```

第二种方法是使用 Google 的 Guava。如果项目中引入了 Google 的 Guava，那么可以使用 Resources 工具类来实现文件的加载，非常方便。示例代码如下：

```java
Resources.getResource("file/test.pdf").getFile();
```

如果是文本文件，还可以使用 Resources.toString 来读取文本的内容。示例代码如下：

```java
Resources.toString(Resources.getResource("file/test.txt"));
```

获取文件或者文件内容后，可以将其用于后续的测试，在 given 方法后可以使用 multiPart
方法来构建表单中的文件数据。示例代码如下：

```java
given()
  .multiPart(Resources.getResource("file/test.pdf").getFile())
  .when().post("/upload");
```

不过采用这种写法时，默认文件表单控件的 name 属性为 "file"，某些 API 会自定义 name 属性需要注意增加相应的参数。示例代码如下：

```java
given()
  .multiPart("custom_file_name", Resources.getResource("file/test.pdf").getFile())
  .when().post("/upload");
```

6.6 模拟第三方 API
------------------

如果我们需要测试的应用依赖于第三方 API，那么我们要先模拟所依赖的 API 才能完成测试，尤其是在微服务架构下，否则会让测试变得比较困难。

除了搭建真实的服务以外，我们还可以使用一些工具和方法来进行模拟。在实际工作中，根据自己的需要从形形色色的方法中寻找一个成本和效果都能达到设计目标的方法即可。

### 6.6.1 使用适配器模式模拟 

一般来说，我们在和第三方 API 对接时，需要先定义一些客户端对象，以便根据场景的不同把与第三方对接的这部分逻辑封装起来，这种模式称为适配器（Adapter）模式。如果适配器的设计良好，只需要模拟对应的对象或者 Bean
即可。

xkcd.com 是一个程序员四格漫画网站，我在 https://any-api.com 中发现了它的 API。这个看上去极其简单的 API 可用来发布一些漫画信息，下面就用它来演示 API 模拟的方法。API 的地址为 https://xkcd.com/info.0.json。访问此地址，可得到 JSON 格式的返回值：

```json
{
  "month": "6",
  "num": 2472,
  "link": "",
  "year": "2021",
  "news": "",
  "safe_title": "Fuzzy Blob",
  "transcript": "",
  "alt": "If there's no dome, how do you explain the irregularities the board discovered in the zoning permits issued in that area!?",
  "img": "https://imgs.xkcd.com/comics/fuzzy_blob.png",
  "title": "Fuzzy Blob",
  "day": "4"
}
```

在 Spring
Boot 的生态中，可以选择的 HTTP 客户端挺多，比如 RestTemplate、Feign 或者简单的 OkHttp 等，在这个例子中，使用的是 RestTemplate。

如图 6-3 所示，在项目中添加一个 DailyComicController
和相关的类，用于调用远程的 API。

![图 6-3
更新后的代码结构](./06-api-testing/media/image3.png)

图 6-3 更新后的代码结构

Controller 类会调用 XkcdClient，并返回 DailyComic-Response
对象，这个对象目前只有一个 imageLink
属性，从 API 中返回的对应字段为 img。示例代码如下：

```java
@RestController
@RequestMapping("/api/daily-comic")
public class DailyComicController {
    @Autowired
    private XkcdClient xkcdClient;

    @GetMapping("")
    public ResponseEntity<DailyComicResponse> getCurrentComic() {
        XkcdVO xkcdVO = xkcdClient.getXkcdResponse();
        return ResponseEntity.ok(DailyComicResponse.builder().imageLink(xkcdVO.getImg()).build());
    }
}
```

XkcdClient 类使用 RestTemplate 的 getForObject 方法来获取远程 API 的数据。Spring
框架已经帮我们封装好了 HTTP 客户端，我们只需要定义一下 RestTemplate 的 Bean。示例代码如下：

```java
@Service
public class XkcdClient {
    @Autowired
    private RestTemplate restTemplate;

    public XkcdVO getXkcdResponse() {
        return restTemplate.getForObject("https://xkcd.com/info.0.json", XkcdVO.class);
    }
}
```

下面来编写一个测试，验证这个接口是否能返回需要的内容：

```java
class DailyComicControllerTest extends TestBase {
    @Test
    void get_current_comic() {
        given()
                .contentType("application/json")
                .when()
                .get("/daily-comic")
                .then().statusCode(200)
                .body("imageLink", is("https://imgs.xkcd.com/comics/fuzzy_blob.png"));
    }
}
```

如果代码编写无误，这个测试可以通过。这时还没有使用任何模拟工具，测试运行时会有如下两个问题。

-   每次运行测试时都需要依赖外部的 API，并且要保证能访问到服务目标。

-   如果对方的 API 发生了变化，测试就会失败。

接下来模拟 XkcdClient 的 Bean，首先在 TestBase 中添加如下代码，实现对 Bean 的模拟，这部分在前面章节已经讲过。

```java
@MockBean
private XkcdClient xkcdClient;
```

然后在测试中使用 @Autowired 获得这个模拟对象，并给予特定的返回值。这里需要注意的是，@MockBean 必须先在 TestBase 类中定义好，然后再使用 @Autowired。下面就是完整的带有 Mock 的测试。

```java
class DailyComicControllerTest extends TestBase {
    @Autowired
    private XkcdClient xkcdClient;

    @Test
    void get_current_comic() {
        BDDMockito.given(xkcdClient.getXkcdResponse()).willReturn(new XkcdVO() {{
            setImg("https://imgs.xkcd.com/comics/fuzzy_blob.png");
        }});

        given()
                .contentType("application/json")
                .when()
                .get("/daily-comic")
                .then().statusCode(200)
                .body("imageLink", is("https://imgs.xkcd.com/comics/fuzzy_blob.png"));
    }
}
```

通过 @MockBean 的方法模拟第三方 API，对源代码的侵入较强，对实现代码有一些要求。一些比较复杂的系统访问第三方 API 时，相关的实现可能没那么规范，使用 MockBean
也就没那么方便了，这时可以使用外部的 API Mock 工具。这些 Mock
工具的原理是通过启动一个服务来加载相应的 JSON 文件，从而实现定制的返回，它们也会提供简单的规则实现不同的响应需求。

开源的 Mock 工具非常多，比如 WireMock、Moco、json-server。下面介绍一个最具有代表性的 Mock 工具：WireMock。

### 6.6.2 WireMock

WireMock 是一个 HTTPMock 服务器，它的核心是一个 Web 服务器，可以为特定的请求规则返回相应的内容。另外，它还提供了验证功能，可保证测试的有效性。

WireMock 还可以实现录制请求的功能，这项功能可大大减少测试的工作量。当一个请求过来时，WireMock
工具会先把请求转发到服务目标上，或者在返回相应的值后将其缓存下来，并为测试生成对应的文件。另外一个比较有意思的功能是返回错误或延迟的数据，这用于测试一些特殊的场景，以便覆盖更多的测试用例。

书中讨论的是 Java
API 测试，刚好 WireMock 构建于 JVM 平台之上，它可以单独启动，也可以内嵌在项目中启动，稍后会演示这两种启动方式。由于可以独立启动，因此只要安装了 JRE
环境就可以运行，故而它不限于用于 Java 项目中。

#### 1. 独立启动

根据官网的介绍可知，WireMock 通过 Jar 包直接启动时，与命令行程序没有区别。可以在 Maven 仓库中找到 Jar 文件的下载地址 https://repol.maven.org/maven2/com/github/tomakehurst/wiremock-jre8-standalone/2.28.0/wiremock-jre8-standalone-2.28.0.jar。

启动的命令如下：

```shell
java -jar wiremock-jre8-standalone-2.28.0.jar
```

官网给出的参数比较多，这里选择几个常用的进行说明。

-   --port:
    设置一个 HTTP 的端口号。这是为了与 HTTPS 的端口区分开来，因为在 Web 服务器中，两种端口是不同的实现。

-   --https-port:
    设置一个 HTTPS 端口。需要注意的是，WireMock 依然默认开启 HTTP 端口（默认值为 8080），如果开启了多个 WireMock 实例，最好也同时指定 HTTP
    端口。大部分情况下，应用依赖的远程服务地址都是可以配置的。可以让生产使用 HTTPS，测试依然配置为 HTTP。如果需要配置 HTTPS，则需要相应的证书和密钥。

-   --bind-address:
    主机鉴定绑定，默认绑定的是本地网络适配器（0.0.0.0），如果没有多个网卡，可以忽略。

-   --root-dir:
    设置模拟数据文件的存放目录，用于存放 mappings 和_files 这两个子目录，默认为当前程序的工作目录。

-   --record-mappings:
    开启记录模式，把请求到后方的数据记录下来，并存放到目录中。

-   --match-headers: 开启记录模式后，把一些请求头中的值也记录下来。

-   --enable-stub-cors:
    允许自动发送 CORS 响应头，可以解决前端调试时存在的跨域问题，这里默认为关闭状态。

-   --help: 显示命令的帮助提示。

此外，WireMock 还提供了 HTTPS 证书、代理和线程控制等相关参数，但是用得不多，如需使用，查询手册即可。

这里还是以 xkcd.com 的访问值为例进行说明，我们不使用任何参数启动 WireMock，而是通过 API 动态地添加 Mock 数据。为了减少代码，只输出一个 "Hello
World"。服务启动后，组织下面的 API 请求。

```shell
curl -X POST --data '{ "request": { "url": "/info.0.json", "method":"GET" }, "response": {"status": 200, "body":"{"hello":"world"}" }}' 
http://localhost:8080/__admin/mappings/new
```

其中，/_admin/mappings/new
API 是 WireMock 的管理 API，可以使用它设置特定的模拟内容。我们来看一下设置是否成功：

```shell
curl http://localhost:8080/info.0.json

{"hello":"world"}
```

大家可能注意到了，每次使用模拟数据时，都需要调用一次 API 先设置模拟数据，这样做比较麻烦，可以直接编写配置文件来实现模拟数据的准备工作。

实例启动时会自动创建_files 和 mappings 这两个目录，编写的配置文件可以放到这两个目录中，下面介绍一下这两个目录的使用方法。

-   __files:
    当 WireMock 启动时，这个目录会被当作一个静态服务器的根路径。例如，在这个目录下编写 hello.html，则可以通过 http://localhost:8080/hello.html 访问。通常，这个目录用于一些没有特定规则的场景，也可以用于大的文件 API，比如需要存放一张图片时。

-   mappings:
    这个目录可以用于请求映射返回需要的内容，比如匹配请求中的 URL、Header，返回特定的数据。可以用于定义更为复杂的交互场景，适合较为动态的 API。另外，它也具有比__files 更高的优先级。

mappings 中不能像 __files
一样直接放静态文件（如果这样就不能发挥动态模拟的价值了），需要使用规定的 DSL 语法来实现。依然使用 xkcd 的例子，不过这次是通过定义配置文件的方式来实现模拟数据的准备。

在 mappings 目录的 info.0.json 中放入下面的文件内容，注意 body 方法接收的还是 String 类型，因此需要转义为 JSON 的内容。

```json
{
    "request": {
        "method": "GET",
        "url": "/info.0.json"
    },
    "response": {
        "status": 200,
      "body":"{"month":"6","num":2472,"link":"","year":"2021","news":"","safe_title":"Fuzzy Blob","transcript":"","alt":"If there's no dome, how do you explain the irregularities the board discovered in the zoning permits issued in that area!?","img":"https://imgs.xkcd.com/comics/fuzzy_blob.png","title":"Fuzzy Blob","day":"4"}"
    }
}
```

#### 2. 内嵌在 JUnit 平台中使用

WireMock 不仅提供了独立启动的方式，还可以使用依赖包在测试中启动。通过 WireMock
暴露出来的 API 可实现灵活的配置、验证、重置、录制等功能，这就为我们的测试带来无限的可能。下面演示如何使用
WireMock 在测试中动态地模拟第三方服务。

首先，依然是在 POM 文件中添加依赖项，这里参考官网的例子使用了 wiremock-jre8
这个版本，建议大家根据 Java 版本选择合适的 WireMock 发行版。

下面创建一个测试类，直接启动 WireMockServer 即可，这与前面介绍的 RedisServer
类似，都是需要启动一个实例。

```java
public class UserMockServerTest extends TestBase {
    private WireMockServer wireMockServer;

    @Autowired
    RestTemplate restTemplate;

    @BeforeEach
    public void setUp() {
        super.setUp();
        wireMockServer = new WireMockServer(options().port(9090));
        wireMockServer.start();
    }

    @AfterEach
    void tearDown() {
        wireMockServer.stop();
    }

    @Test
    void test_mock_server() {
        String response = restTemplate.getForObject("http://localhost:9090/users", String.class);
        System.out.println(response);
    }
}
```

为了简化演示，这里没实现源代码中的客户端代码，而是直接使用 RestTemplate 请求启动了 WireMock 的模拟服务。在 test_mock_server 测试中，我们访问 http://localhost:9090/users 会得到一个 404 的错误，因为这里还没有设置任何模拟数据。

接下来，编写一个最基本的动态模拟代码，并说明对应方法的用途。

```java
configureFor("localhost", 9090);
stubFor(get(urlEqualTo("/users"))
        .willReturn(aResponse()
                .withHeader("Content-Type", "text/json")
                .withBody("{"name":"john"}")));
```

configureFor(String host, int port)
方法用于给后续的方法 stubFor 配置一个 WireMock 的地址和端口，而 stubFor 的原理又与调用 WireMock 的 / admin 端口一样，因此也需要为其配置对应的端口。

stubFor(MappingBuilder mappingBuilder)
方法用于配置模拟的数据，由于 get 方法的重名方法太多（Mockito、JDK
等包中也有 get 方法），建议将 com.github.tomakehurst.wiremock.
client.WireMock
这个包使用 "*" 引进来。get、urlEqualTo、aResponse 都是用于构建匹配规则和返回值的相关方法。

添加以上的方法后，再次运行测试。顺利的话，测试会通过并打印出需要返回的数据内容。上面通过 Java 语言描述的模拟数据等价于下面的 JSON 文件：

```json
{
    "request": {
        "method": "GET",
        "url": "/users"
    },
    "response": {
        "status": 200,
        "body": "{\"name\":\"john\"}",
        "headers": {
            "Content-Type": "text/json"
        }
    }
}
```

WireMock 类中提供了非常多的 Builder 方法，这样就可以更加灵活地构建模拟 API，这对于构建一些异常返回特别有用。比如，我们可以省略 urlEqualTo 方法，直接使用 okJson 方法构建 JSON 的返回内容。示例代码如下：

```java
@Test
void test_mock_json() {
    configureFor("localhost", 9090);
    stubFor(get("/users")
            .willReturn(okJson("{\"name\":\"john\"}")));

    String response = restTemplate.getForObject("http://localhost:9090/users", String.class);
    Assertions.assertEquals("{\"name\":\"john\"}", response);
}
```

当测试服务异常时，也可以使用 unauthorized、forbidden、notFound 等方法构建异常返回。示例代码如下：

```java
@Test
void mock_status() {
    configureFor("localhost", 9090);
    stubFor(get("/limited-resources")
            .willReturn(unauthorized()));

    Assertions.assertThrows(HttpClientErrorException.Unauthorized.class, () -> {
        restTemplate.getForEntity("http://localhost:9090/limited-resources", String.class);
    });
}
```

还可以使用 temporaryRedirect 构建跳转方法，比如测试 302、301 跳转。默认的 RestTemplate
开启了自动获取跳转后的数据的功能，这里为了演示方便，断言了跳转到 /redirect-to 后的返回值。示例代码如下：

```java
@Test
void mock_redirect() {
    configureFor("localhost", 9090);
    stubFor(get("/redirect-to")
            .willReturn(ok("new-url")));
    stubFor(get("/redirect")
            .willReturn(temporaryRedirect("/redirect-to")));

    String response = restTemplate.getForObject("http://localhost:9090/redirect-to", String.class);
    Assertions.assertEquals(response, "new-url");
}
```

如果想关闭 RestTemplate 的自动跳转，可以自己定义一个 Bean 进行配置。关闭 RestTemplate
的自动跳转（也可以使用前面介绍的 TestRestTemplate）后，可以使用 getForEntity
获取返回的 HTTP 包实体，并获取真实的状态码。示例代码如下：

```java
@Bean
public RestTemplate restTemplate () {
    RestTemplate restTemplate = new RestTemplate();
    final HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
    CloseableHttpClient build =
            HttpClientBuilder.create().disableRedirectHandling().build();
    factory.setHttpClient(build);
    restTemplate.setRequestFactory(factory);
    return restTemplate;
}
```

#### 3. 调用验证

WireMock 实例在启动后会记录所有的测试请求，并且在重置之前都不会清理掉。它的用处就是获取请求的实际情况，从而判定测试结果。同时，它也可以获取所有请求的详情，为调试 Bug 带来便利。

通过 Jar 包独立启动的 WireMock 服务可以通过管理 API 获取匹配结果，但是使用独立启动的 WireMock 服务时，很少会去验证模拟数据的匹配情况。

通过内嵌启动的 WireMock 服务使用 Java
API 来实现对 API 的验证就比较简单，只需要调用 verify 方法即可：

```java
@Test
void verify_mock_server() {
    // 准备 Mock 数据
    configureFor("localhost", 9090);
    stubFor(get(urlEqualTo("/users"))
            .willReturn(aResponse()
                    .withHeader("Content-Type", "text/json")
                    .withBody("{\"name\":\"john\"}")));

    // 调用被测试内容
    String response = restTemplate.getForObject("http://localhost:9090/users", String.class);
    System.out.println(response);

    // 验证被依赖 API 是否被调用
    verify(getRequestedFor(urlEqualTo("/users"))
            .withHeader("Content-Type", equalTo("text/json")));
}
```

示例代码中的 verify 方法与 Mockito
中的相关方法重名，请注意引入的包是否正确。这里的
getRequestedFor 获得 Builder 对象的语法与准备
Mock 数据时非常相似。verify 方法没有别的参数时，会验证匹配的规则是否至少调用了一次。

在这个例子中，restTemplate 对象发出去的请求没有带上 Content-Type，因此测试会失败。测试失败后，可以在控制台查看错误信息，如图 6-4 所示。使用 IntelliJIDEA 运行测试，单击错误信息可以查看差异，从而分析问题产生的原因。

![图 6-4
对比分析错误原因](./06-api-testing/media/image4.png)

图 6-4 对比分析错误原因

当然，与 Mockito 类似，verify 方法除了默认执行至少一次的校验外，还提供了多种验证操作：

```java
verify(lessThan(5), postRequestedFor(urlEqualTo("/many")));
verify(lessThanOrExactly(5), postRequestedFor(urlEqualTo("/many")));
verify(exactly(5), postRequestedFor(urlEqualTo("/many")));
verify(moreThanOrExactly(5), postRequestedFor(urlEqualTo("/many")));
verify(moreThan(5), postRequestedFor(urlEqualTo("/many")));
```

#### 4. 自定义 JUnit 5 Extension

我们知道 JUnit4 提供了 Rule，WireMock 也提供了一个 WireMockRule 来集成原生的 API。回顾之前介绍的 Rule，我们知道，Rule 是 JUnit 的一种拓展方式，可以让测试更为灵活，它能起到类似插件的作用。Rule 最基本的使用方式就是 @Rule 注解在测试类中声明。示例代码如下：

```java
@Rule
public WireMockRule wireMockRule = new WireMockRule();
```

也可以增加一些参数：

```java
@Rule
public WireMockRule wireMockRule = new WireMockRule(options().port(8888).httpsPort(8889));
```

但是，我们的例子现在已经升级到了 JUnit5 版本，在 Junit5 中使用 Extension 代替了 Rule，让拓展的编写更容易理解。这里使用 WireMock 的原生 API 编写一个 JUnit
5 的 Extension。编写 Extension 的方法比较简单，因为我们可以直接使用 WireMockServer 对象作为父类，然后实现相应的接口。

```java
public class WireMockExtension extends WireMockServer implements BeforeEachCallback, AfterEachCallback {
    public WireMockExtension() {
    }

    @Override
    public void afterEach(ExtensionContext context) {
        stop();
        resetAll();
    }

    @Override
    public void beforeEach(ExtensionContext context) {
        start();
    }
}
```

一个极其简单的 Extension 就开发完成了，使用 @RegisterExtension 可以注人 Extension
代替每次手动编写的 start、stop 方法。示例代码如下：

```java
public class WireMockExtensionTest extends TestBase {
    @RegisterExtension
    WireMockExtension wireMock = new WireMockExtension();
...
    }
}
```

WireMockServer 提供了根据参数构建模拟服务的方法，在 Extension 中也可以提供相应的方法。示例代码如下：

```java
// 根据配置构建 WireMockServer
public WireMockExtension(Options options) {
    super(options);
}
// 根据端口构建 WireMockServer
public WireMockExtension(int port, Integer httpsPort) {
    super(port, httpsPort);
}
```

6.7 API 自动化测试策略
----------------------

在编写 API 测试时，测试策略是一件比较令人头疼的事情，相对来说，测试的一些技术问题反而比较好解决。原因是应用架构的水平参差不齐，API 的职责不是那么清晰，从而导致测试变得困难。

良好的 API 设计会尽可能地考虑我们之前所说的 MECE 原则（Mutually Exclusive
Collectively
Exhaustive），也就是说，在功能穷尽的情况下，还会设法做到彼此独立。这样一来，API 的职责就比较单一了，自然我们的测试工作也会相对简单。

对于不同的 API 类型，测试策略也会不一样。这里的测试策略与通常意义上的测试策略略有不同，因为已经限定了是 API 测试，所以它的测试目标、类型基本都是确定的，那么我们讨论的重点也就变成了如何更好地为 API 组织测试、如何提高测试有效性，以及如何让测试的编写变得更加便利。

在实际工作中，测试策略往往与架构有关，API 测试受架构的制约。例如，在微服务架构中，面向应用的 API 在一些公司叫作 Experience
API、前台、BFF 等，而面向内部服务的 API 则叫作领域服务或核心服务 API。

下面以餐饮系统为例，说明上述两种 API 的细微差异。用户可以在手机上点餐，通过浏览器或者 APP 访问用户端 API，管理员可以通过管理端 API 管理用户等系统中的资源。这些请求最终都会通过不同的应用 API 到达领域 API 中，如图 6-5 所示。

![](./06-api-testing/media/image5.png)

图 6-5 不同类型的 API

### 6.7.1 面向应用的 API

应用 API 只是对领域 API 的组织和编排，没有特别多的业务逻辑，因此我们的测试重点是验证这套
API 能否完成端到端的验证。这类
API 的测试工作最好按照场景来组织，比较推荐的做法是根据场景来测试，可使用用户旅程（User
Journey）测试的风格来组织测试。

由于应用 API 依赖于用户服务、产品服务等，因此需要通过模拟这些依赖让场景测试顺利进行。又因为应用 API 中的业务逻辑和分支条件较少，所以不用考虑异常分支就能让测试的覆盖率较高。这类
API 测试也不需要关注后端 API 的业务逻辑中发生了哪些业务异常。

#### 1. 测试目的

针对面向应用的
API 进行测试时，目标是测试 API 的业务场景，验证字段是否正常返回，以及最终的后端 API 是否被合理调用。

#### 2. 测试准备

主要使用 WireMock 对依赖的后端 API 进行模拟。在实践中，可以把常用的模拟方法组合成通用的方法，避免重复编写测试代码。另外，这类测试往往与用户的认证有关，因此需要模拟与用户认证相关的行为，具体请参考前面的鉴权处理。

#### 3. 测试组织方式

使用用户旅程来组织测试在 JUnit 中有两种风格。

第一种是通过类的方式来组织、然后在类的级别中设置通用的 Mock 行为，并通过设置测试方法的运行序列整体运行测试。在 JUnit5 中，可以使用 @Order 注解让测试顺序执行。示例代码如下：

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class BuyScenarioTest {
    @BeforeAll
    static void beforeAll() {
        // 这里进行通用的准备
    }

    @AfterAll
    static void afterAll() {
        // 这里进行通用的清理
    }

    @Test
    @Order(1)
    void should_list_products() {
        System.out.println("should_list_products");
    }

    @Test
    @Order(2)
    void should_get_product_detail() {
        System.out.println("should_get_product_detail");
    }

    @Test
    @Order(3)
    void should_add_product_to_shopping_cart() {
        System.out.println("should_add_product_to_shopping_cart");
    }
    ...
}
```

由于在这一层 API 中，我们放弃了对异常（指局部异常，对分支流程的异常还是会测试）的测试，因此让整个测试变得非常容易理解和维护。只有低成本的自动化测试才会让团队有动力持续地维护下去。

但习惯了采用这种方式组织测试之后，很多开发者会对那些无意义的 @Order 注解感到厌烦 ------------ 既然我们是使用场景组织 API 测试的，何不就以一个场景作为单位划分测试呢？这就是第二种风格，直接以场景为粒度进行测试，使用一个测试编排多个测试方法。在其他的 BDD 测试框架中，对此是有专门的语法来支持的。示例代码如下：

```java
public class ScenarioTest {
    @Test
    void buy_scenario_test() {
        // 这里进行通用的准备
        shouldListProducts();
        shouldGetProductDetail();
        shouldAddProductToShoppingCart();
        ...
        // 这里进行通用的清理
    }
    void shouldListProducts() {
        System.out.println("should_list_products");
    }
    void shouldGetProductDetail() {
        System.out.println("should_get_product_detail");
    }
    void shouldAddProductToShoppingCart() {
        System.out.println("should_add_product_to_shopping_cart");
    }
}
```

### 6.7.2 面向领域服务的 API

面向领域服务的 API 不是为最终的场景服务的，而是提供通用的业务能力。以产品服务为例，对于产品服务来说，它提供的是基本的服务能力，例如添加产品、删除产品。领域服务的各个 API 之间的关联性相对较弱，因此可以独立地组织测试数据，不过，需要验证异常测试。

#### 1. 测试目标

针对面向领域服务 API 进行测试时，目标是测试
API 的业务能力，包括单元测试不能覆盖的异常行为。

#### 2. 测试准备

领域服务除了会依赖少量的第三方 API 之外，还会依赖数据库、文件等基础设施。这里参考之前的内容为不同的基础设施准备测试内容：

-   启动内嵌的数据库等基础设施。

-   直接使用源码中的 Service、Repository 等 Bean 来准备数据库中的内容。

-   使用 WireMock 来处理外部系统。

#### 3. 测试组织方式

由于面向领域服务的 API 提供的基本是原子性的 API，因此其与单元测试的组织风格类似，即保持一个 Controller 对应一个测试类。若为每个 API 单独准备、执行、验证测试用例，那么一个测试类可能会有很多个测试。想让测试结构更加清晰，可以把正向、异常测试组织到一个嵌套测试中。示例代码如下：

```java
public class ProductControllerTest {
    @Nested
    @DisplayName("query product list suite")
    class QueryProduct {
        @Test
        void should_list_product_list_with_default_page() {
            System.out.println("should_list_product_list_with_default_page");
        }
        @Test
        void should_list_product_list_with_specify_page() {
            System.out.println("should_list_product_list_with_specify_page");
        }
    }
    @Nested
    @DisplayName("add product test suite")
    class AddProduct {
        @Test
        void should_add_product_success() {
            System.out.println("should_add_product_success");
        }
        @Test
        void should_add_product_failed_when_exceed_product_limitation() {
            System.out.println("should_add_product_failed_when_exceed_product_limitation");
        }
    }
}
```

嵌套类尤其适合这种场景，在 IntelliJIDEA 中运行测试时，可以在测试结果中看到非常清晰的测试结构。图 6-6 展示了嵌套测试执行后的样子。

![图 6-6
使用嵌套类组织测试](./06-api-testing/media/image6.png)

图 6-6 使用嵌套类组织测试

6.8 小结
--------

在前后端分离和微服务盛行的技术趋势下，API 测试变得越来越重要。本章介绍了如何使用 MockMvc、RESTAssured 等工具实现 API 测试，以及一些相关的注意事项和技巧。API 测试不再是单元测试的范畴，它的测试工具、Mock 工具、测试策略等都与单元测试有所不同。通过 WireMock 可以隔离依赖的第三方 API，让处于错综复杂的调用关系下的 API 更容易被测试。

对于微服务来说，不同类型的微服务会提供不同的 API，这些 API 需要使用不同的测试策略，为场景定制测试策略而不是死板地使用同一套方法尤为重要。
