# 跟波波学Pact(基于Maven + Springboot + JUnit5)

## 介绍

本向导旨在演示Pact的核心功能、以及契约驱动测试的好处。

本向导演示[消费者驱动契约consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)方法 - 服务的消费方和提供方并行开发，基于契约不断迭代演化服务，尤其在开发之初，双方对要开发功能的细节还不是非常清楚。

学习本课程耗时约1~2个小时，具体看你学习的深入程度。

**课程大纲**:

- [step 1: **创建消费方**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step1#step-1---simple-consumer-calling-provider): 在提供方API还不就绪前先创建消费方
- [step 2: **单元测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step2#step-2---client-tested-but-integration-fails): 为消费方编写单元测试
- [step 3: **Pact测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step3#step-3---pact-to-the-rescue): 为消费方编写Pact测试
- [step 4: **Pact校验**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step4#step-4---verify-the-provider): 利用提供方API校验消费方的Pact
- [step 5: **修复消费方**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step5#step-5---back-to-the-client-we-go): 修复消费方对提供方的错误假设
- [step 6: **Pact测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step6#step-6---consumer-updates-contract-for-missing-products): 在消费方编写一个404(用户不存在)的Pact测试用例
- [step 7: **提供方陈述(states)**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step7#step-7---adding-the-missing-states): 更新提供方的API来处理404情况
- [step 8: **Pact测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step8#step-8---authorization): 在消费方写一个401的Pact测试用例
- [step 9: **Pact测试**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step9#step-9---implement-authorisation-on-the-provider): 更新提供方的API来处理401情况
- [step 10: **请求过滤器**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step10#step-10---request-filters-on-the-provider) : 修复提供方以支持401情况
- [step 11: **Pact中介(broker)**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step11#step-11---using-a-pact-broker): 实现Pact中介工作流，集成CI/CD

_注意：每一步都和一个git分支相关联，这样你可以依次检出分支，逐步学习每一个步骤。例如，如果要学习step 2，那么请运行：`git checkout step2`_。

## 学习目标

[这里](./LEARNING.md)有更详细的关于每一步的学习目标。

## 需求

- JDK 8 or 更高版本
- Maven 3
- step 11需要Docker

## 场景

本课程的源码包含两个主要的服务：

1. Product Catalog网站。Product Service的消费方，它通过查询Product service获取产品信息。
1. Product Service(提供方)。提供关于产品的有用信息，包括列出所有产品，获取每个产品的详情。

## Step 1 - 消费方简单调用提供方

首先，我们需要Consumer端创建一个HTTP client，它可以调用我们的provider service：

![Simple Consumer](diagrams/workshop_step1.svg)

消费方已经实现了调用product service的client，它支持：

- `GET /products` - 获取所有产品
- `GET /products/{id}` - 通过ID获取一个产品

下图展示了获取ID为10的产品时，消费方和提供方之间的交互：

![Sequence Diagram](diagrams/workshop_step1_class-sequence-diagram.svg)

你可以在`consumer/src/main/java/io/pact/workshop/product_catalogue/clients/ProductServiceClient.java`这个文件中看到client实现:

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

现在在浏览器中[访问应用](http://localhost:8080)，访问页面上的*here*链接，你会得到500错误页面，因为下游的服务提供方还不存在。

在Springboot控制台输出中，你也会看到一个异常：

```
 I/O error on GET request for "http://localhost:9000/products": Connection refused
```

*现在可以进入[step 2](https://github.com/pact-foundation/pact-workshop-Maven-Springboot-JUnit5/tree/step2#step-2---client-tested-but-integration-fails)*
