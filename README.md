# 跟波波学 Pact(基于 Maven + Springboot + JUnit5)

## 介绍

本向导旨在演示 Pact 的核心功能、以及契约驱动测试的好处。

本向导演示[消费者驱动契约 consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)方法 - 服务的消费方和提供方并行开发，基于契约不断迭代演化服务，尤其在开发之初，双方对要开发功能的细节还不是非常清楚。

学习本课程耗时约 1~2 个小时，具体看你学习的深入程度。

**课程大纲**:

- [step 1: **创建消费方**](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step1#step-1---simple-consumer-calling-provider): 在提供方 API 还不就绪时先创建消费方
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

## 开始

*从[step 1](https://github.com/boboweike/pact-workshop-Maven-Springboot-JUnit5/tree/step1#step-1---simple-consumer-calling-provider)*开始
