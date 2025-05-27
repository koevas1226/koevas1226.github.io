---
title: 个人简历
date: 2021-01-16 19:19:40
layout: resume
---

#### 基本信息
----
- 姓名 蒲锦辉
- 邮箱 3130104027@stmail.ujs.edu.cn
- 电话 15751005740
- 主页 https://koevas1226.github.io

#### 个人简介

资深后端开发工程师，具备基于云原生体系架构开发能力，专注于构建高性能、高可用服务系统。具备AI应用研发相关技术，了解大模型相关知识。丰富的后端开发经验，熟悉分布式系统、微服务架构、数据库设计与优化、缓存机制及消息队列等常见服务端基础技术。
#### 核心技能
- 编程语言：NodeJS / TypeScript / Python / Golang（基础）
- 框架技术：EggJS, NestJS, Express, Koa, FastAPI, Flask
- 数据库：MongoDB, Vector DBs, Postgres, MySQL, Redis, Elasticsearch
- 云原生工具：Docker, Kubernetes, Helm, Prometheus, Grafana
- AI相关: Langchain, RAG, Eembdding, Prompt Engineering

#### 工作经历
----
  ###### 2020.01.01 ~ 至今     &nbsp;&nbsp;&nbsp;&nbsp;[和鲸HeyWhale](https://www.kesci.com/) &ensp; 资深后端工程师
  - **主要工作**
    - 设计并开发机器学习全生命周期的平台的工具链及基础设施, 提供了在线运行notebook、模型部署、离线任务、计算资源及镜像管理、数据资产协同及管理等模块一体的Saas平台。服务60w+数据科学工作者，周均活跃度4K+。
    - 调研并从零到一实现了企业级RAG基础服务，基于Python+Langchain+Qdrant的异步业务架构，利用celery队列集成原有的知识库业务，支持了多种格式文件解析。封装并支持了类OpenAI格式的embedding+rerank模型。利用k8s HPA将批处理任务处理自动扩容，峰值任务处理效率提高80%+,利用多租户特点进行了分表机制，单机支持QPS 3000+查询。
    - 设计并实现了多LTS交付版本的onceJob升级系统，解决了多版本交付下的数据一致性问题，为运维部署团队减少了大量新环境部署的工作量，获得了2024年度优秀员工表彰。
    - 接入并优化了10+类lambda服务的CI镜像构建工作,利用istio网格实现了蓝绿部署，90%的项目减少了90%的build等待时间。
    - 基于promethus+grafana+opentelemetry搭建了观测系统,利用jaeger实现链路可视化。
    -  `权限管理` 历史包袱下的c端权限管理及设计随着业务场景越来越复杂，重复代码和接口越来越多的情况下，负责进行了重构。在RBAC的基础上，抽离出一套组件系统，作为中间件的形式，满足了接口的权限控制，业务代码和权限校验解耦，丰富了代码的可扩展性以及鲁棒性, 目前已支持30+模块及千万级数据p99<30ms返回。
    -  `任务调度` 对接第三方系统时，某些原因只提供了java版本的jar包，通过代码分析、抓包重写了一套node版本的基础服务。后续发现对方服务不稳定，会延迟自己服务的响应，基于nsq的消息队列实现了任务调度系统，将第三方服务与业务解耦，实现了自己的业务秒响应。
    - `埋点系统` 在需要采集数据操作历史的场景下，基于ts的decorator写了一套可拔插的埋点系统，将场景行为抽象统一储存，基于某些私有化的特殊需求，将高频需求和低频需求解耦，在不影响业务代码的情况下，将埋点的业务需求、场景行为、储存分离，增加了代码的可移植性。
    - `mongodb慢查询调研` 线上业务遇到了慢查询，通过分析Mongodb的源码以及在本地进行复现问题，写了篇mongodb查询计划分析的文档，并在公司技术会上进行分享。讨论和参与制定了业务层的一些规范，推动团队进行迭代，使业务慢查询减少了90%。
  


  ###### 2018.12 ~ 2019.11     &nbsp;&nbsp;&nbsp;&nbsp;[AfterShip](https://www.aftership.com/)&ensp; 后端开发
  - **主要工作**  
    面向海外客户提供了物流快递订单查询，技术架构以beanstalk消息队列为基础，nodejs作为调度系统，python进行查询的爬虫系统，每日新增查询在百万级别。
    - 接入了30+courier站点
    - 适配解决aws lambda内存限制的问题。
    - 完善单元测试覆盖率达90%以上
    - 自动化部署升级工作，部署一次的成本由2小时缩短至半小时

  ###### 2017.6 ~ 2018.9  &nbsp;&nbsp;&nbsp;&nbsp;全栈开发
  - **主要工作**  
    - 基于微信API开发, 使用svg图像touch热点，用纯jquery实现了不同地区的工位选座功能。
    - 基于java+springboot开发了商城系统
    - 基于typescript开发了法币交易系统



<p>&nbsp;</p>

#### 开源项目
- 将AWS S3、阿里云OSS、腾讯云COS、Minio统一接口的npm包, 通过环境变量注入解耦。
- China DOI期刊Typescript接口版本，通过wireshark抓包对接口进行解析，支持注册，更新期刊链接等功能。

