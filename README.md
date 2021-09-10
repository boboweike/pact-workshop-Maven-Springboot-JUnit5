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
- [step 7: **提供方陈述(states)**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step7#step-7---adding-the-missing-states): 更新提供方的 API 来处理 404 情况
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

_现在可以进入[step 2](https://github.com/pact-foundation/pact-workshop-Maven-Springboot-JUnit5/tree/step2#step-2---client-tested-but-integration-fails)_

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

现在请[访问网站](http://localhost:8080)，页面上应该会显示 3 个不同的产品。上面有一个`Details ...`链接应该可以展示产品的详情。

但是当你点击详情链接时候，看看发生了什么！

![Failed page](diagrams/workshop_step2_failed_page.png)

结果显示每次点击详情链接都会得到 500 错误。经过排查，发现 Consumer 端的日志中有访问 Producer Service 时产生的 404 错误。

```
HttpClientErrorException$NotFound: 404 : [{"timestamp":"2021-02-25T05:27:51.264+00:00","status":404,"error":"Not Found","message":"","path":"/products/9"}]]
```

问题出在 provider 提供的端点是`/product/{id}`和`/products`，而 consumer 请求的是`products/{id}`。

所以，消费方需要先和提供方确认端点！

_现在转到[step 3](https://github.com/pact-foundation/pact-workshop-Maven-Springboot-JUnit5/tree/step3#step-3---pact-to-the-rescue)_
