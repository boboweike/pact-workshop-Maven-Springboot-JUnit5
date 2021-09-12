# 跟波波学 Pact(基于 Maven + Springboot + JUnit5)

## 介绍

本向导旨在演示 Pact 的核心功能、以及契约驱动测试的好处。

本向导演示[消费者驱动契约 consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)方法 - 服务的消费方和提供方并行开发，基于契约不断迭代演化服务，尤其在开发之初，双方对要开发功能的细节还不是非常清楚。

学习本课程耗时约 1~2 个小时，具体看你学习的深入程度。

**课程大纲**:

- [step 1: **创建消费方**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step1#step-1---simple-consumer-calling-provider): 在提供方 API 还不就绪前先创建消费方
- [step 2: **单元测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step2#step-2---client-tested-but-integration-fails): 为消费方编写单元测试
- [step 3: **Pact 测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step3#step-3---pact-to-the-rescue): 为消费方编写 Pact 测试
- [step 4: **Pact 校验**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step4#step-4---verify-the-provider): 利用提供方 API 校验消费方的 Pact
- [step 5: **修复消费方**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step5#step-5---back-to-the-client-we-go): 修复消费方对提供方的错误假设
- [step 6: **Pact 测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step6#step-6---consumer-updates-contract-for-missing-products): 在消费方编写一个 404(用户不存在)的 Pact 测试用例
- [step 7: **提供方状态(states)**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step7#step-7---adding-the-missing-states): 更新提供方的 API 来处理 404 情况
- [step 8: **Pact 测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step8#step-8---authorization): 在消费方写一个 401 的 Pact 测试用例
- [step 9: **Pact 测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step9#step-9---implement-authorisation-on-the-provider): 更新提供方的 API 来处理 401 情况
- [step 10: **请求过滤器**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step10#step-10---request-filters-on-the-provider) : 修复提供方以支持 401 情况
- [step 11: **Pact 中介(broker)**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step11#step-11---using-a-pact-broker): 实现 Pact 中介工作流，集成 CI/CD

_注意：每一步都和一个 git 分支相关联，这样你可以依次检出分支，逐步学习每一个步骤。例如，如果要学习 step 2，那么请运行：`git checkout step2`_。

## 学习目标

[这里](./LEARNING.md)有更详细的关于每一步的学习目标。

## 需求

- JDK 8 or 更高版本
- Maven 3
- step 11 需要 Docker

## 场景

本课程的源码包含两个主要的服务：

1. Product Catalog 网站。Product Service 的消费方，它通过查询 Product service 获取产品信息。
1. Product Service(提供方)。提供关于产品的有用信息，包括列出所有产品，获取每个产品的详情。

## Step 1 - 消费方简单调用提供方

首先，我们需要 Consumer 端创建一个 HTTP client，它可以调用我们的 provider service：

![Simple Consumer](diagrams/workshop_step1.svg)

消费方已经实现了调用 product service 的 client，它支持：

- `GET /products` - 获取所有产品
- `GET /products/{id}` - 通过 ID 获取一个产品

下图展示了获取 ID 为 10 的产品时，消费方和提供方之间的交互：

![Sequence Diagram](diagrams/workshop_step1_class-sequence-diagram.svg)

你可以在`consumer/src/main/java/io/pact/workshop/product_catalogue/clients/ProductServiceClient.java`这个文件中看到 client 实现:

```java
@Service
public class ProductServiceClient {
  @Autowired
  private RestTemplate restTemplate;

  @Value("${serviceClients.products.baseUrl}")
  private String baseUrl;

  public ProductServiceResponse fetchProducts() {
    return restTemplate.getForObject(baseUrl + "/products", ProductServiceResponse.class);
  }

  public Product fetchProductById(long id) {
    return restTemplate.getForObject(baseUrl + "/products/" + id, Product.class);
  }
}
```

在克隆了本仓库以后，你需要构建项目下载依赖。在`consume`目录中，运行如下命令：

```console
consumer ❯ mvn verify
```

现在你可以运行项目：

```console
consumer ❯ java -jar target/product-catalogue-0.0.1-SNAPSHOT.jar
```

现在在浏览器中[访问应用](http://localhost:8080)，访问页面上的[here](http://localhost:8080/catalogue)链接，你会得到 500 错误页面，因为下游的服务提供方还不存在。

在 Springboot 控制台输出中，你也会看到一个异常：

```
 I/O error on GET request for "http://localhost:9000/products": Connection refused
```

_现在可以进入[step 2](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step2#step-2---client-tested-but-integration-fails)_

## Step 2 - 客户端已经测试过，但是集成时失败了

现在我们给 API client 创建一个基本的测试。我们想要检查两个东西：

1. 客户端代码命中了正确的端点
1. 响应可以被正确地反序列为一个对象，并且其中的数据是正确的

从`consumer/src/test/java/io/pact/workshop/product_catalogue/clients/ProductServiceClientTest.java`这个文件中你可以看到相应的测试代码:

```java
  @Test
  void getProductById(@Wiremock WireMockServer server, @WiremockUri String uri) {
      productServiceClient.setBaseUrl(uri);
      server.stubFor(
          get(urlPathEqualTo("/products/10"))
              .willReturn(aResponse()
              .withStatus(200)
              .withBody("{\n" +
                  "            \"id\": 50,\n" +
                  "            \"type\": \"CREDIT_CARD\",\n" +
                  "            \"name\": \"28 Degrees\",\n" +
                  "            \"version\": \"v1\"\n" +
                  "        }\n")
              .withHeader("Content-Type", "application/json"))
      );

      Product product = productServiceClient.getProductById(10);
      assertThat(product, is(equalTo(new Product(10L, "28 Degrees", "CREDIT_CARD", "v1"))));
  }
```

![Unit Test With Mocked Response](diagrams/workshop_step2_unit_test.svg)

运行测试，校验测试都通过：

```console
consumer ❯ mvn verify
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------< io.pact.workshop:product-catalogue >-----------------
[INFO] Building product-catalogue 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-resources-plugin:3.2.0:resources (default-resources) @ product-catalogue ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO]
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ product-catalogue ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:3.2.0:testResources (default-testResources) @ product-catalogue ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] skip non existing resourceDirectory /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ product-catalogue ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/test-classes
[INFO]
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ product-catalogue ---
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running io.pact.workshop.product_catalogue.clients.ProductServiceClientTest

<<< Omitted lots of logs >>>

[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.952 s - in io.pact.workshop.product_catalogue.clients.ProductServiceClientTest
2021-02-25 13:37:46.510  INFO 25640 --- [extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-catalogue ---
[INFO]
[INFO] --- spring-boot-maven-plugin:2.4.3:repackage (repackage) @ product-catalogue ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.535 s
[INFO] Finished at: 2021-02-25T13:37:47+11:00
[INFO] ------------------------------------------------------------------------
```

如果在运行`mvn verify`后显示测试失败，请确认你的当前分支是`step2`。

期间，我们的服务提供方 provider 团队开始并行开发了他们的 API。现在，我们可以同时运行 consumer 和 provider(你需要在两个终端中分别运行)：

```console
# Terminal 1
consumer ❯ mvn spring-boot:run

<<< Omitted >>>

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.3)

2021-02-25 13:46:36.835  INFO 26565 --- [           main] i.p.w.product_catalogue.Application      : Starting Application using Java 1.8.0_265 on ronald-P95xER with PID 26565 (/home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/classes started by ronald in /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer)
2021-02-25 13:46:36.836  INFO 26565 --- [           main] i.p.w.product_catalogue.Application      : No active profile set, falling back to default profiles: default
2021-02-25 13:46:37.274  INFO 26565 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-02-25 13:46:37.279  INFO 26565 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-02-25 13:46:37.279  INFO 26565 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2021-02-25 13:46:37.303  INFO 26565 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-02-25 13:46:37.303  INFO 26565 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 441 ms
2021-02-25 13:46:37.405  INFO 26565 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-02-25 13:46:37.481  INFO 26565 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-02-25 13:46:37.486  INFO 26565 --- [           main] i.p.w.product_catalogue.Application      : Started Application in 0.866 seconds (JVM running for 1.051)

```

```console
# Terminal 2
provider ❯  mvn spring-boot:run

<<< Omitted >>>

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.3)

2021-02-25 16:06:08.671  INFO 40129 --- [           main] i.p.w.product_service.Application        : Starting Application using Java 1.8.0_265 on ronald-P95xER with PID 40129 (/home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/provider/target/classes started by ronald in /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/provider)
2021-02-25 16:06:08.673  INFO 40129 --- [           main] i.p.w.product_service.Application        : No active profile set, falling back to default profiles: default
2021-02-25 16:06:08.990  INFO 40129 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2021-02-25 16:06:09.013  INFO 40129 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 18 ms. Found 1 JPA repository interfaces.
2021-02-25 16:06:09.159  INFO 40129 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.ws.config.annotation.DelegatingWsConfiguration' of type [org.springframework.ws.config.annotation.DelegatingWsConfiguration$$EnhancerBySpringCGLIB$$c6a6464a] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-02-25 16:06:09.189  INFO 40129 --- [           main] .w.s.a.s.AnnotationActionEndpointMapping : Supporting [WS-Addressing August 2004, WS-Addressing 1.0]
2021-02-25 16:06:09.344  INFO 40129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 9000 (http)
2021-02-25 16:06:09.349  INFO 40129 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-02-25 16:06:09.349  INFO 40129 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2021-02-25 16:06:09.382  INFO 40129 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-02-25 16:06:09.382  INFO 40129 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 684 ms
2021-02-25 16:06:09.469  INFO 40129 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2021-02-25 16:06:09.525  INFO 40129 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2021-02-25 16:06:09.549  INFO 40129 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2021-02-25 16:06:09.569  INFO 40129 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.4.28.Final
2021-02-25 16:06:09.621  INFO 40129 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.2.Final}
2021-02-25 16:06:09.702  INFO 40129 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.H2Dialect
2021-02-25 16:06:09.954  INFO 40129 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2021-02-25 16:06:09.958  INFO 40129 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2021-02-25 16:06:10.093  WARN 40129 --- [           main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2021-02-25 16:06:10.149  INFO 40129 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-02-25 16:06:10.259  INFO 40129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 9000 (http) with context path ''
2021-02-25 16:06:10.265  INFO 40129 --- [           main] i.p.w.product_service.Application        : Started Application in 1.812 seconds (JVM running for 2.0)
2021-02-25 16:06:11.833  INFO 40129 --- [nio-9000-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-02-25 16:06:11.833  INFO 40129 --- [nio-9000-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-02-25 16:06:11.834  INFO 40129 --- [nio-9000-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
```

现在请[访问网站](http://localhost:8080)，页面上应该会显示 3 个不同的产品。上面各有一个`Details ...`链接应该可以展示产品的详情。

但是当你点击详情链接时候，看看发生了什么！

![Failed page](diagrams/workshop_step2_failed_page.png)

结果显示每次点击详情链接都会得到 500 错误。经过排查，发现 Consumer 端的日志中有访问 Producer Service 时产生的 404 错误。

```
HttpClientErrorException$NotFound: 404 : [{"timestamp":"2021-02-25T05:27:51.264+00:00","status":404,"error":"Not Found","message":"","path":"/products/9"}]]
```

问题出在 provider 提供的端点是`/product/{id}`和`/products`，而 consumer 请求的是`products/{id}`。

所以，消费方需要先和提供方确认端点！

_现在转到[step 3](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step3#step-3---pact-to-the-rescue)_

## Step 3 - Pact 来帮忙

单元测试主要用于测试独立的组件/服务(独立于其它的服务)。当你对和其它服务交互的代码进行测试时(例如对 Consumer 访问 Producer 的 HTTP client 进行测试)，你是基于某种假设契约进行测试的。你无法直接校验这个契约在消费方和提供方之间是否正确。

> 契约集成测试是指一种对服务边界进行的测试，它用于校验服务提供方的接口满足消费方的期望 ~~ [Martin Fowler](https://martinfowler.com/bliki/IntegrationContractTest.html)

如果添加了基于 Pact 的契约测试，那么就可以提前发现`/product/{id}`这个端点是不正确的。

下面我们在项目中添加 Pact 依赖，然后为`GET /products/{id}`端点编写一个契约测试。

*提供方状态(Provider states)*是我们需要介绍的 Pact 中的一个重要概念。这些状态定义了在某个具体的交互中，provider 必须处于某种状态。目前，我们先测试下面的状态：

- `ID为10的产品存在`
- `所有产品存在`

消费者可以使用`given`属性来定义一个交互的状态。

Pact 测试代码 `consumer/src/test/java/io/pact/workshop/product_catalogue/clients/ProductServiceClientPactTest.java`:

```java
@SpringBootTest
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ProductService")
class ProductServiceClientPactTest {
  @Autowired
  private ProductServiceClient productServiceClient;

  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact allProducts(PactDslWithProvider builder) {
    return builder
      .given("products exists")
      .uponReceiving("get all products")
      .path("/products")
      .willRespondWith()
      .status(200)
      .body(
        new PactDslJsonBody()
          .minArrayLike("products", 1, 2)
          .integerType("id", 9L)
          .stringType("name", "Gem Visa")
          .stringType("type", "CREDIT_CARD")
          .closeObject()
          .closeArray()
      )
      .toPact();
  }

  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact singleProduct(PactDslWithProvider builder) {
    return builder
      .given("product with ID 10 exists", "id", 10)
      .uponReceiving("get product with ID 10")
      .path("/products/10")
      .willRespondWith()
      .status(200)
      .body(
        new PactDslJsonBody()
          .integerType("id", 10L)
          .stringType("name", "28 Degrees")
          .stringType("type", "CREDIT_CARD")
          .stringType("code", "CC_001")
          .stringType("version", "v1")
      )
      .toPact();
  }

  @Test
  @PactTestFor(pactMethod = "allProducts", port="9999")
  void testAllProducts(MockServer mockServer) throws IOException {
    productServiceClient.setBaseUrl(mockServer.getUrl());
    List<Product> products = productServiceClient.fetchProducts().getProducts();
    assertThat(products, hasSize(2));
    assertThat(products.get(0), is(equalTo(new Product(9L, "Gem Visa", "CREDIT_CARD", null, null))));
  }

  @Test
  @PactTestFor(pactMethod = "singleProduct", port="9999")
  void testSingleProduct(MockServer mockServer) throws IOException {
    Product product = productServiceClient.getProductById(10L);
    assertThat(product, is(equalTo(new Product(10L, "28 Degrees", "CREDIT_CARD", "v1", "CC_001"))));
  }
}
```

![Test using Pact](diagrams/workshop_step3_pact.svg)

这些测试会在随机端口上启动一个 mock server，它扮演 provider service 的角色。为了让这个 mock server 生效，我们将`ProductServiceClient`的 URL 更新为指向 Pact 所提供的这个 mock server。

运行这些测试的话都会通过，同时还会生成一个 pact 契约文件，后面我们可以利用这个 pack 文件，来校验在交互时 provider 端是否能满足 consumer 端的契约假设。

```console
consumer ❯ mvn verify
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------< io.pact.workshop:product-catalogue >-----------------
[INFO] Building product-catalogue 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- maven-resources-plugin:3.2.0:resources (default-resources) @ product-catalogue ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] Copying 1 resource
[INFO] Copying 6 resources
[INFO]
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ product-catalogue ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:3.2.0:testResources (default-testResources) @ product-catalogue ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] skip non existing resourceDirectory /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ product-catalogue ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 2 source files to /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/test-classes
[INFO]
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ product-catalogue ---
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-catalogue ---
[INFO] Building jar: /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/product-catalogue-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.4.3:repackage (repackage) @ product-catalogue ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.013 s
[INFO] Finished at: 2021-02-26T09:43:03+11:00
[INFO] ------------------------------------------------------------------------
```

测试完毕，会生成一个 Pact 契约文件*consumer/target/pacts/ProductCatalogue-ProductService.json*

_注意_：即便 Provider 团队已经给我们提供了 API client，我们仍然需要编写契约测试 ~~ 因为这个 client 版本和已经部署的 API 可能并不同步，同时，我们需要根据我们提供的具体数据进行测试。

_现在可以转到[step 4](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step4#step-4---verify-the-provider)_

## Step 4 - 校验提供方 provider

我们需要将 consumer test 中生成的 Pact 契约文件，拷贝到 Provider 项目中。这样我们才能校验 provider 满足 consumer 所预期的契约需求。

将契约文件 `consumer/target/pacts/ProductCatalogue-ProductService.json` 拷贝到 `provider/pacts/ProductCatalogue-ProductService.json`.

现在我们可以在 provider 端编写一个用于校验 consumer 契约的 Pact 测试：

在 `provider/src/test/java/io/pact/workshop/product_service/PactVerificationTest.java` 文件中:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("ProductService")
@PactFolder("pacts")
@IgnoreMissingStateChange
public class PactVerificationTest {
  @LocalServerPort
  private int port;

  @Autowired
  private ProductRepository productRepository;

  @BeforeEach
  void setup(PactVerificationContext context) {
    context.setTarget(new HttpTestTarget("localhost", port));
  }

  @TestTemplate
  @ExtendWith(PactVerificationInvocationContextProvider.class)
  void pactVerificationTestTemplate(PactVerificationContext context) {
    context.verifyInteraction();
  }
}
```

这是一个 Springboot 测试，它在一个随机端口上启动应用，然后将端口注入到测试类中。

然后我们通过在测试类上添加一些注解来设置测试上下文，这些注解告诉 Pact 框架 provider 是哪个，pact 文件在哪里。我们也将测试目标指向运行的应用。

现在，通过运行测试，我们就可以校验 consumer 所生成的 pact 契约是否合法，但是目前这个测试会失败：

```console
provider ❯ mvn verify

<<< Omitted >>>

1) Verifying a pact between ProductCatalogue and ProductService - get product with ID 10: has status code 200

    1.1) status: expected status of 200 but was 404

    1.2) body: $ Expected code='CC_001' but was missing

        {
        -  "code": "CC_001",
        -  "id": 10,
        -  "name": "28 Degrees",
        -  "type": "CREDIT_CARD",
        -  "version": "v1"
        +  "timestamp": "2021-02-26T02:50:41.293+00:00",
        +  "status": 404,
        +  "error": "Not Found",
        +  "message": "",
        +  "path": "/products/10"
        }

<<< Omitted >>>

```

![Pact Verification](diagrams/workshop_step4_pact.svg)

这个测试会失败，因为预期的 path `/products/{id}`返回了 404。消费端预期 provider 提供的接口遵循 RESTful 设计，但是实际上这个预期是错误的 🤷🏻‍♂️。

消费者应该调用的正确端点是`/product/{id}`。

现在可以转移到 [step 5](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step5#step-5---back-to-the-client-we-go)

## Step 5 - 回到客户端修复问题

现在我们需要更新消费端的测试代码，让 consumer client 访问正确的 product path。

首先，我们需要更新 client 中的 GET 路由：

在文件`consumer/src/main/java/io/pact/workshop/product_catalogue/clients/ProductServiceClient.java`中:

```java
  public Product getProductById(long id) {
    return restTemplate.getForObject(baseUrl + "/product/" + id, Product.class);
  }
```

然后我们需要更新`ID 10 exists`这个 Pact 测试，在`path`部分使用正确的端点。

在这个文件 `consumer/src/test/java/io/pact/workshop/product_catalogue/clients/ProductServiceClientPactTest.java`中:

```java
  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact singleProduct(PactDslWithProvider builder) {
    return builder
      .given("product with ID 10 exists", "id", 10)
      .uponReceiving("get product with ID 10")
        .path("/product/10")
      .willRespondWith()
        .status(200)
        .body(
          new PactDslJsonBody()
            .integerType("id", 10L)
            .stringType("name", "28 Degrees")
            .stringType("type", "CREDIT_CARD")
            .stringType("code", "CC_001")
            .stringType("version", "v1")
        )
      .toPact();
  }
```

下面我们运行 consumer 端测试，生成一个新的 pact 契约文件：

```console
consumer ❯ mvn verify

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-catalogue ---
[INFO] Building jar: /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/product-catalogue-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.4.3:repackage (repackage) @ product-catalogue ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

现在运行 providers 端的测试，利用更新过的契约。

![Pact Verification](diagrams\workshop_step5_pact.svg)

请将 `consumer/target/pacts/ProductCatalogue-ProductService.json` 拷贝到 `provider/pacts`目录中。

运行命令：

```console
provider ❯ mvn verify

<<< Omitted >>>

Verifying a pact between ProductCatalogue and ProductService
  [Using File pacts/ProductCatalogue-ProductService.json]
  Given product with ID 10 exists
  get product with ID 10
2021-02-26 13:11:45.229  INFO 96199 --- [o-auto-1-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-02-26 13:11:45.229  INFO 96199 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-02-26 13:11:45.230  INFO 96199 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
    returns a response which
      has status code 200 (OK)
      has a matching body (OK)
2021-02-26 13:11:45.356  WARN 96199 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   : Not all of the 2 were verified. The following were missing:
2021-02-26 13:11:45.356  WARN 96199 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   :     get all products
2021-02-26 13:11:45.366  INFO 96199 --- [           main] p.j.PactVerificationStateChangeExtension : Invoking state change method 'products exists':SETUP

Verifying a pact between ProductCatalogue and ProductService
  [Using File pacts/ProductCatalogue-ProductService.json]
  Given products exists
  get all products
    returns a response which
      has status code 200 (OK)
      has a matching body (OK)
2021-02-26 13:11:45.485  WARN 96199 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   : Skipping publishing of verification results as it has been disabled (pact.verifier.publishResults is not 'true')
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.573 s - in io.pact.workshop.product_service.PactVerificationTest
2021-02-26 13:11:45.527  INFO 96199 --- [extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
2021-02-26 13:11:45.527  INFO 96199 --- [extShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2021-02-26 13:11:45.528  INFO 96199 --- [extShutdownHook] .SchemaDropperImpl$DelayedDropActionImpl : HHH000477: Starting delayed evictData of schema as part of SessionFactory shut-down'
2021-02-26 13:11:45.532  INFO 96199 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2021-02-26 13:11:45.534  INFO 96199 --- [extShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
[INFO]
```

Yes，现在 Provider 端的测试变成绿色了 ✅!

现在可以转到 [step 6](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step6#step-6---consumer-updates-contract-for-missing-products)

## Step 6 - 更新消费者契约，测试产品不存在情况

现在我们要给契约添加 2 个新的场景：

- 对于获取某个产品的调用，如果对应的产品不存在，会发生什么情况？我们假定会得到`404`错误。
- 对于获取所有产品的调用，如果当前 Provider 端还没有产品，会发生什么情况？我们假定 Provider 端会返回`200`代码和一个空数组。

让我们来给这些新的场景编写测试，并生成更新的 pact 契约文件。

在文件 `consumer/src/test/java/io/pact/workshop/product_catalogue/clients/ProductServiceClientPactTest.java`中：

```java
  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact noProducts(PactDslWithProvider builder) {
    return builder
      .given("no products exists")
      .uponReceiving("get all products")
        .path("/products")
      .willRespondWith()
        .status(200)
        .body(
          new PactDslJsonBody().array("products")
        )
      .toPact();
  }

  @Test
  @PactTestFor(pactMethod = "noProducts")
  void testNoProducts(MockServer mockServer) {
    productServiceClient.setBaseUrl(mockServer.getUrl());
    ProductServiceResponse products = productServiceClient.fetchProducts();
    assertThat(products.getProducts(), hasSize(0));
  }

  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact singleProductNotExists(PactDslWithProvider builder) {
    return builder
      .given("product with ID 10 does not exist", "id", 10)
      .uponReceiving("get product with ID 10")
        .path("/product/10")
      .willRespondWith()
        .status(404)
      .toPact();
  }

  @Test
  @PactTestFor(pactMethod = "singleProductNotExists")
  void testSingleProductNotExists(MockServer mockServer) {
    productServiceClient.setBaseUrl(mockServer.getUrl());
    try {
      productServiceClient.getProductById(10L);
      fail("Expected service call to throw an exception");
    } catch (HttpClientErrorException ex) {
      assertThat(ex.getMessage(), containsString("404 Not Found"));
    }
  }
```

注意新加的测试和之前的测试几乎是相同的，差别只是在对`response`的预期数据不同 ~~ HTTP 请求部分的预期数据是完全相同的。

```console
consumer ❯ mvn verify

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-catalogue ---
[INFO] Building jar: /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/consumer/target/product-catalogue-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.4.3:repackage (repackage) @ product-catalogue ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

对于这些消费端新加的测试，provider 端测试会展现什么样的行为呢？我们再次将 pact 文件拷贝到 provider 的 pact 目录中，并运行命令：

```console
provider ❯ mvn verify

<<< Omitted >>>

Verifying a pact between ProductCatalogue and ProductService
  [Using File pacts/ProductCatalogue-ProductService.json]
  Given no products exists
  get all products
2021-02-26 13:45:27.930  INFO 100121 --- [o-auto-1-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-02-26 13:45:27.932  INFO 100121 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-02-26 13:45:27.934  INFO 100121 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
    returns a response which
      has status code 200 (OK)
      has a matching body (FAILED)

Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get all products has a matching body

    1.1) body: $.products Expected an empty List but received [{"code":"CC_01_VISA","id":9,"name":"Gem Visa","type":"CREDIT_CARD","version":"v1"},{"code":"CC_02_OTH","id":10,"name":"28 Degrees","type":"CREDIT_CARD","version":"v1"},{"code":"LN_PERS_01","id":11,"name":"MyFlexiPay","type":"PERSONAL_LOAN","version":"v2"}]

        [
        -
        +  {
        +    "id": 9,
        +    "name": "Gem Visa",
        +    "type": "CREDIT_CARD",
        +    "version": "v1",
        +    "code": "CC_01_VISA"
        +  },
        +  {
        +    "id": 10,
        +    "name": "28 Degrees",
        +    "type": "CREDIT_CARD",
        +    "version": "v1",
        +    "code": "CC_02_OTH"
        +  },
        +  {
        +    "id": 11,
        +    "name": "MyFlexiPay",
        +    "type": "PERSONAL_LOAN",
        +    "version": "v2",
        +    "code": "LN_PERS_01"
        +  }
        ]



2021-02-26 13:45:28.460  WARN 100121 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   : Not all of the 4 were verified. The following were missing:
2021-02-26 13:45:28.460  WARN 100121 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   :     get product with ID 10
2021-02-26 13:45:28.461  WARN 100121 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   :     get product with ID 10
2021-02-26 13:45:28.461  WARN 100121 --- [           main] a.c.d.p.p.DefaultTestResultAccumulator   :     get all products
2021-02-26 13:45:28.463  WARN 100121 --- [           main] p.j.PactVerificationStateChangeExtension : Did not find a test class method annotated with @State("no products exists")
for Interaction "get all products"
with Consumer "ProductCatalogue"
2021-02-26 13:45:28.530  WARN 100121 --- [           main] p.j.PactVerificationStateChangeExtension : Did not find a test class method annotated with @State("product with ID 10 does not exist")
for Interaction "get product with ID 10"
with Consumer "ProductCatalogue"

Verifying a pact between ProductCatalogue and ProductService
  [Using File pacts/ProductCatalogue-ProductService.json]
  Given product with ID 10 does not exist
  get product with ID 10
    returns a response which
      has status code 404 (FAILED)
      has a matching body (OK)

Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get product with ID 10: has status code 404

    1.1) status: expected status of 404 but was 200

```

我们预期测试会失败，因为我们所请求的产品在 provider 是存在的！实际上我们想要测试的是 Provider 的不同状态，也就是所谓的"提供方状态(Provider states)"。

我们可以更新消费端的测试代码，让它查询一个已知的不存在的产品，这样可以修复这个测试失败。但是实际上，我们真正的目标是让大家理解"Provider states"是如何工作的。

_请转到 [step 7](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step7#step-7---adding-the-missing-states)_

## Step 7 - 在提供方增加缺失的状态

我们需要更新 provider 端的代码来处理找不到产品的场景，并发送`404`响应。但是，在我们的数据库中有 product ID 是 10 和 11 的测试数据。

所以在本步骤中，我们会在 provider 的 Pact 验证测试中添加一些状态处理器(state handlers)，它们会根据 consumer 端的状态需求来更新数据存储中的状态。

在实际的测试功能被调用之前，状态处理器会先被调用，执行顺序如下：

```
BeforeEach -> StateHandler (setup) -> RequestFilter -> Execute Provider Test -> StateHandler (teardown) -> AfterEach
```

我们需要为 4 种状态添加状态处理器：

- 产品存在
- 产品不存在
- ID 为 10 的产品存在
- ID 为 10 的产品不存在

Provider 的 Pact 测试校验代码在这个文件 provider/src/test/java/io/pact/workshop/product_service/PactVerificationTest.java`中：

```java
  @State(value = "products exists", action = StateChangeAction.SETUP)
  void productsExists() {
    productRepository.deleteAll();
    productRepository.saveAll(Arrays.asList(
      new Product(100L, "Test Product 1", "CREDIT_CARD", "v1", "CC_001"),
      new Product(200L, "Test Product 2", "CREDIT_CARD", "v1", "CC_002"),
      new Product(300L, "Test Product 3", "PERSONAL_LOAN", "v1", "PL_001"),
      new Product(400L, "Test Product 4", "SAVINGS", "v1", "SA_001")
    ));
  }

  @State(value = "no products exists", action = StateChangeAction.SETUP)
  void noProductsExist() {
    productRepository.deleteAll();
  }

  @State(value = "product with ID 10 exists", action = StateChangeAction.SETUP)
  void productExists(Map<String, Object> params) {
    long productId = ((Number) params.get("id")).longValue();
    Optional<Product> product = productRepository.findById(productId);
    if (!product.isPresent()) {
      productRepository.save(new Product(productId, "Product", "TYPE", "v1", "001"));
    }
  }

  @State(value = "product with ID 10 does not exist", action = StateChangeAction.SETUP)
  void productNotExist(Map<String, Object> params) {
    long productId = ((Number) params.get("id")).longValue();
    Optional<Product> product = productRepository.findById(productId);
    if (product.isPresent()) {
      productRepository.deleteById(productId);
    }
  }
```

现在可以校验 Provider 端的 Pact 测试：

```console
provider ❯ mvn verify

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-service ---
[INFO] Building jar: /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/provider/target/product-service-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

_注意_：Provider 端的状态未必和 consumer 中的契约测试一一对应。你可以在不同的测试间重用状态。例如在该场景中，我们可以在两个测试中重用`no products exist`。

_现在请转到 [step 8](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step8#step-8---authorization)_

## Step 8 - 授权(Authorization)

不是每一个人都可以访问我们的 GraphQL API。经过和团队讨论，我们认为使用一个有时间限制的 bearer token 就够了。这个 token 是一个 base64 编码的时间戳，表示从当前开始到 1 小时之内这段时间。

在本例中，如果请求中没有带 bearer token，那么我们预期一个`401`错误。现在我们来更新消费方，让它传递 bearer token，并且捕获这个`401`场景。

In `consumer/src/main/java/io/pact/workshop/product_catalogue/clients/ProductServiceClient.java`:

```java
@Service
public class ProductServiceClient {
  @Autowired
  private RestTemplate restTemplate;

  @Value("${serviceClients.products.baseUrl}")
  private String baseUrl;

  public ProductServiceResponse fetchProducts() {
    return callApi("/products", ProductServiceResponse.class);
  }

  public Product getProductById(long id) {
    return callApi("/product/" + id, Product.class);
  }

  public String getBaseUrl() {
    return baseUrl;
  }

  public void setBaseUrl(String baseUrl) {
    this.baseUrl = baseUrl;
  }

  private <T> T callApi(String path, Class<T> responseType) {
    HttpHeaders headers = new HttpHeaders();
    ByteBuffer buffer = ByteBuffer.allocate(Long.BYTES);
    buffer.putLong(System.currentTimeMillis());
    headers.setBearerAuth(Base64.getEncoder().encodeToString(buffer.array()));
    HttpEntity<?> requestEntity = new HttpEntity<>(headers);
    return restTemplate.exchange(baseUrl + path, HttpMethod.GET, requestEntity, responseType).getBody();
  }
}
```

在 Consumer 的 HTTP client 代码 `consumer/src/test/java/io/pact/workshop/product_catalogue/clients/ProductServiceClientPactTest.java`中，将下列代码：

```java
    .matchHeader("Authorization", "Bearer [a-zA-Z0-9=\\+/]+", "Bearer AAABd9yHUjI=")
```

添加到如下 4 个交互契约中：

```java
  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact noAuthToken(PactDslWithProvider builder) {
    return builder
      .uponReceiving("get all products with no auth token")
        .path("/products")
      .willRespondWith()
        .status(401)
      .toPact();
  }

  @Test
  @PactTestFor(pactMethod = "noAuthToken")
  void testNoAuthToken(MockServer mockServer) {
    productServiceClient.setBaseUrl(mockServer.getUrl());
    try {
      productServiceClient.fetchProducts();
      fail("Expected service call to throw an exception");
    } catch (HttpClientErrorException ex) {
      assertThat(ex.getMessage(), containsString("401 Unauthorized"));
    }
  }

  @Pact(consumer = "ProductCatalogue")
  public RequestResponsePact noAuthToken2(PactDslWithProvider builder) {
    return builder
      .uponReceiving("get product by ID with no auth token")
        .path("/product/10")
      .willRespondWith()
        .status(401)
      .toPact();
  }

  @Test
  @PactTestFor(pactMethod = "noAuthToken2")
  void testNoAuthToken2(MockServer mockServer) {
    productServiceClient.setBaseUrl(mockServer.getUrl());
    try {
      productServiceClient.getProductById(10L);
      fail("Expected service call to throw an exception");
    } catch (HttpClientErrorException ex) {
      assertThat(ex.getMessage(), containsString("401 Unauthorized"));
    }
  }
```

然后运行 Pact 测试生成一个新的 Pact 文件。

下面我们来测试 Provider 端。将更新的 pact 文件拷贝到 provider 的 pact 目录中，然后运行测试：

```console
provider ❯ mvn verify

<<< Omitted >>>

[ERROR] Tests run: 6, Failures: 2, Errors: 0, Skipped: 0, Time elapsed: 4.631 s <<< FAILURE! - in io.pact.workshop.product_service.PactVerificationTest
[ERROR] pactVerificationTestTemplate{PactVerificationContext}[1]  Time elapsed: 0.487 s  <<< FAILURE!
java.lang.AssertionError:
ProductCatalogue - Upon get all products with no auth token
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get all products with no auth token: has status code 401

    1.1) status: expected status of 401 but was 200


        at io.pact.workshop.product_service.PactVerificationTest.pactVerificationTestTemplate(PactVerificationTest.java:41)

[ERROR] pactVerificationTestTemplate{PactVerificationContext}[2]  Time elapsed: 0.027 s  <<< FAILURE!
java.lang.AssertionError:
ProductCatalogue - Upon get product by ID with no auth token
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get product by ID with no auth token: has status code 401

    1.1) status: expected status of 401 but was 200
```

因为在消费方我们添加了新的契约预期 ~~ 如果消费方未提供授权 token(最后两个测试)，那么 provider 端应该返回 401 响应，但是因为 provider 还没有处理授权，所以我们得到的是 200，所以最后两个测试失败了。

_转到 [step 9](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step9#step-9---implement-authorisation-on-the-provider)_

## Step 9 - 在 Provider 端实现授权

我们通过设置 Spring Security 来检查授权 HTTP header，如果请求中没有带上 token 或者 token 超过 1 小时的话就返回`401`。

令牌校验逻辑在代码`provider/src/main/java/io/pact/workshop/product_service/config/BearerAuthorizationFilter.java`中：

```java
public class BearerAuthorizationFilter extends OncePerRequestFilter {

  public static final long ONE_HOUR = 60 * 60 * 1000L;

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    String header = request.getHeader("Authorization");
    if (tokenValid(header)) {
      SecurityContextHolder.getContext().setAuthentication(new PreAuthenticatedAuthenticationToken("user", header));
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
    }
  }

  private boolean tokenValid(String header) {
    boolean hasBearerToken = StringUtils.isNotEmpty(header) && header.startsWith("Bearer ");
    if (hasBearerToken) {
      String token = header.substring("Bearer ".length());
      ByteBuffer buffer = ByteBuffer.allocate(Long.BYTES);
      buffer.put(Base64.getDecoder().decode(token));
      buffer.flip();
      long timestamp = buffer.getLong();
      return System.currentTimeMillis() - timestamp <= ONE_HOUR;
    } else {
      return false;
    }
  }
}
```

现在再次运行 Provider 端的契约测试：

```console
provider ❯ mvn verify

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[ERROR] Failures:
[ERROR]   PactVerificationTest.pactVerificationTestTemplate:41 ProductCatalogue - Upon get all products
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get all products: has status code 200

    1.1) status: expected status of 200 but was 401

    1.2) header: Expected a header 'Content-Type' but was missing

    1.3) body-content-type: Expected a response type of 'application/json' but the actual type was 'null'


[ERROR]   PactVerificationTest.pactVerificationTestTemplate:41 ProductCatalogue - Upon get product with ID 10
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get product with ID 10: has status code 404

    1.1) status: expected status of 404 but was 401


[ERROR]   PactVerificationTest.pactVerificationTestTemplate:41 ProductCatalogue - Upon get product with ID 10
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get product with ID 10: has status code 200

    1.1) status: expected status of 200 but was 401

    1.2) header: Expected a header 'Content-Type' but was missing

    1.3) body-content-type: Expected a response type of 'application/json' but the actual type was 'null'


[ERROR]   PactVerificationTest.pactVerificationTestTemplate:41 ProductCatalogue - Upon get all products
Failures:

1) Verifying a pact between ProductCatalogue and ProductService - get all products: has status code 200

    1.1) status: expected status of 200 but was 401

    1.2) header: Expected a header 'Content-Type' but was missing

    1.3) body-content-type: Expected a response type of 'application/json' but the actual type was 'null'


[INFO]
[ERROR] Tests run: 6, Failures: 4, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
```

结果显示更多测试失败了，你知道这是什么原因造成的吗？

_转到 [step 10](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step10#step-10---request-filters-on-the-provider)_

## Step 10 - 在Provider端添加请求过滤器

因为我们的pact文件中的数据是静态的，所以在Provider接收到请求时，其中的bearer token已经过期了，所以我们在Provider端的测试中获得很多的`401`错误。有几个办法可以解决这个问题，例如可以使用mock，或者将认证模块抽取到公共代码中。我们这边准备使用一种称为_Request Filtering_的技术 ~~ 截获并且修改请求。

_注意_：这是一种高级的技术，应该谨慎使用，因为它是一种workaround机制，潜在可能会是契约失效。更多细节请参考 https://github.com/pact-foundation/pact-jvm/blob/master/provider/junit5/README.md#modifying-the-requests-before-they-are-sent

下面我们来截获请求并注入合法的授权头：

1. 如果我们接收到一个授权HTTP头，那么我们就生成一个合法(在时间限制内)的授权头，添加到请求上，并继续向后调用。
1. 如果我们接收到的请求中不含授权头，那么我们就不做处理直接向后传递。

_注意_：在本例中我们不考虑`403`场景。

在代码 `provider/src/test/java/io/pact/workshop/product_service/PactVerificationTest.java`中：

```java
  @TestTemplate
  @ExtendWith(PactVerificationInvocationContextProvider.class)
  void pactVerificationTestTemplate(PactVerificationContext context, HttpRequest request) {
    // WARNING: Do not modify anything else on the request, because you could invalidate the contract
    if (request.containsHeader("Authorization")) {
      request.setHeader("Authorization", "Bearer " + generateToken());
    }
    context.verifyInteraction();
  }

  private static String generateToken() {
    ByteBuffer buffer = ByteBuffer.allocate(Long.BYTES);
    buffer.putLong(System.currentTimeMillis());
    return Base64.getEncoder().encodeToString(buffer.array());
  }
```

现在我们可以再次运行Provider端的测试：

```console
provider ❯ mvn verify

<<< Omitted >>>

[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-service ---
[INFO] Building jar: /home/ronald/Development/Projects/Pact/pact-workshop-Maven-Springboot-JUnit5/provider/target/product-service-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

_转到 [step 11](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step11#step-11---using-a-pact-broker)_
## Step 11 - 使用Pact中介(Broker)

![Broker collaboration Workflow](diagrams/workshop_step10_broker.svg)

之前我们通过文件拷贝的方式在Consumer和Provider之间分享契约文件。但是这种方式在多团队协作时管理不方便。实际上， 我们可以使用[Pact中介(Broker)](https://pactflow.io) 来分享契约。

使用broker可以简化对契约文件的管理，它还增加了其它一些有用的功能，包括针对持续交付的安全性增强，后面我们会介绍。

在本向导中我们会使用开源的Pact broker。

### 使用docker-compose运行Pact Broker

在项目根目录中，运行：

```console
 ❯ docker-compose up
```

### 从消费端发布契约

首先，在consumer项目中，我们需要告诉Pact关于broker的地址。我们在Pact Maven plugin中进行配置。

在文件 `consumer/pom.xml`中：

```xml
<build>
  <plugins>
      ...
      <plugin>
          <groupId>au.com.dius.pact.provider</groupId>
          <artifactId>maven</artifactId>
          <version>4.1.17</version>
          <configuration>
            <pactBrokerUrl>http://localhost:9292</pactBrokerUrl>
            <pactBrokerUsername>pact_workshop</pactBrokerUsername>
            <pactBrokerPassword>pact_workshop</pactBrokerPassword>
          </configuration>
      </plugin>
  </plugins>
</build>
```

现在我们运行：

```console
consumer ❯ mvn pact:publish
[INFO] Scanning for projects...
[INFO] 
[INFO] -----------------< io.pact.workshop:product-catalogue >-----------------
[INFO] Building product-catalogue 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven:4.1.17:publish (default-cli) @ product-catalogue ---
Publishing 'ProductCatalogue-ProductService.json' with tags 'prod, test' ... OK
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.745 s
[INFO] Finished at: 2021-03-01T09:51:29+11:00
[INFO] ------------------------------------------------------------------------
```

通过浏览器打开http://localhost:9292 (username/password：`pact_workshop/pact_workshop`)，你就可以看到刚发布的契约！

### 在Provider端校验契约

在provider端我们需要做的，就是更新测试中的pacts文件的地址，从本地文件改成broker的地址。

我们在测试中添加一个`@PactBroker`注解，并在测试中使用`PactVerificationSpringProvider`，然后创建一个用于测试的application.yml配置文件，其中设置关于Pact Broker的详细信息。

在文件`provider/src/test/java/io/pact/workshop/product_service/PactVerificationTest.java`中：

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("ProductService")
@PactBroker // <--
public class PactVerificationTest {
  @LocalServerPort
  private int port;

  @Autowired
  private ProductRepository productRepository;

  @BeforeEach
  void setup(PactVerificationContext context) {
    context.setTarget(new HttpTestTarget("localhost", port));
  }

  @TestTemplate
  @ExtendWith(PactVerificationSpringProvider.class) // <--
  void pactVerificationTestTemplate(PactVerificationContext context, HttpRequest request) {
    // WARNING: Do not modify anything else on the request, because you could invalidate the contract
    if (request.containsHeader("Authorization")) {
      request.setHeader("Authorization", "Bearer " + generateToken());
    }
    context.verifyInteraction();
  }  

```

然后创建文件 `provider/src/test/resources/application.yml`：

```yaml
pactbroker:
  host: localhost
  port: "9292"
  auth:
    username: pact_workshop
    password: pact_workshop
```

修改完毕，我们再次运行provider端校验：

```console
provider ❯ mvn verify -Dpact.verifier.publishResults=true -Dpact.provider.version=1.0-SNAPSHOT

<<< Omitted >>>

Verifying a pact between ProductCatalogue (0.0.1-SNAPSHOT) and ProductService

  Notices:
    1) The pact at http://localhost:9292/pacts/provider/ProductService/consumer/ProductCatalogue/pact-version/5565ba9dfe81399a66afbebb620a3d79df43d46a is being verified because it matches the following configured selection criterion: latest pact between a consumer and ProductService

  [from Pact Broker http://localhost:9292/pacts/provider/ProductService/consumer/ProductCatalogue/pact-version/5565ba9dfe81399a66afbebb620a3d79df43d46a/metadata/c1tdW2xdPXRydWUmc1tdW2N2bl09MC4wLjEtU05BUFNIT1Q=]
  get all products with no auth token
2021-03-01 10:32:06.625  INFO 32791 --- [o-auto-1-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-03-01 10:32:06.625  INFO 32791 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-03-01 10:32:06.625  INFO 32791 --- [o-auto-1-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
    returns a response which
      has status code 401 (OK)
      has a matching body (OK)
      
<<< Omitted >>>

[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ product-service ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

测试完毕后，校验的结果(测试是否成功、交互失败的详细信息)也会被发布到Broker。

这是Broker的强大功能之一。它被称为校验[Verifications](https://docs.pact.io/pact_broker/advanced_topics/provider_verification_results)，它允许providers将校验结果报告发送回broker。然后，你可以通过一个不错的dashboard快速查看consumer和provider的状态。但是实际上Broker还有更重要的用途。

### 我可以发布吗(Can I deploy)?

通过简单使用`pact-broker`的一个工具[can-i-deploy](https://docs.pact.io/pact_broker/advanced_topics/provider_verification_results) - Broker会决定一个consumer或者provider是否可以安全地发布到某个特定的环境。

你可以按如下方式运行`can-i-deploy`检查：

```console
consumer ❯ mvn pact:can-i-deploy -Dpacticipant='ProductCatalogue' -Dlatest=true
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< io.pact.workshop:product-service >------------------
[INFO] Building product-service 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven:4.1.17:can-i-deploy (default-cli) @ product-service ---
Computer says yes \o/ 

All required verification results are published and successful
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------


provider ❯ mvn pact:can-i-deploy -Dpacticipant='ProductService' -Dlatest=true
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< io.pact.workshop:product-service >------------------
[INFO] Building product-service 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven:4.1.17:can-i-deploy (default-cli) @ product-service ---
Computer says yes \o/ 

All required verification results are published and successful
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```



本向导学习完毕 - 你现在对Pact契约测试已经入门，可以开始在正式项目中使用Pact 🔨
