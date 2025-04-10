---
title: 个人简历
date: 2021-01-16 19:19:40
layout: resume
---

#### 个人信息
----

- 姓名 蒲锦辉
- 邮箱 3130104027@stmail.ujs.edu.cn
- 毕业 江苏大学 信息与计算科学
- 出生 1995-03-16  
<p>&nbsp;</p>

#### 工作经历
----
  ###### 2020.01.01 ~ 至今     &nbsp;&nbsp;&nbsp;&nbsp;[和鲸HeyWhale](https://www.kesci.com/) &ensp; 后端工程师
  - **在做什么？**
    以Jupyter Notebook为基础搭建的数据科学协作平台，主要提供的服务为免费云计算资源的分配、数据协同功能。技术架构为以nodejs为核心的业务层搭建的微服务, 前后端分离，统一部署在k8s进行服务管理。
  - **做了什么？**
    1. `权限管理` 历史包袱下的c端权限管理及设计随着业务场景越来越复杂，重复代码和接口越来越多的情况下，负责进行了重构。在RDBC(基于角色的权限控制)的基础上，抽离出一套组件系统，作为中间件的形式，满足了接口的权限控制，业务代码和权限校验解耦，丰富了代码的可扩展性以及鲁棒性。
    2. `任务调度` 对接第三方系统时，某些原因只提供了java版本的jar包，通过代码分析、抓包重写了一套node版本的基础服务。后续发现对方服务不稳定，会延迟自己服务的响应，基于nsq的消息队列写了一套简易版的任务调度系统，将第三方服务与业务解耦，实现了自己的业务秒响应。
    3. `埋点系统` 在需要采集数据操作历史的场景下，基于ts的decorator写了一套可拔插的埋点系统，将场景行为抽象统一储存，基于某些私有化的特殊需求，将高频需求和低频需求解耦，在不影响业务代码的情况下，将埋点的业务需求、场景行为、储存分离，增加了代码的可移植性。
    4. `mongodb慢查询调研` 线上业务遇到了慢查询，通过分析Mongodb的源码以及在本地进行复现问题，写了篇mongodb查询计划分析的文档，并在公司技术会上进行分享。讨论和参与制定了业务层的一些规范，补充完善了一些基础设施。
  


  ###### 2018.12 ~ 2019.11     &nbsp;&nbsp;&nbsp;&nbsp;[AfterShip](https://www.aftership.com/)&ensp; 后端开发
  - **在做什么？**
    主要面向海外客户提供了物流快递订单查询。技术架构以beanstalk消息队列为基础，nodejs作为调度系统，每日新增查询在百万级别。
  - **做了什么？**  
    实事求是，因为当时的历史原因，整套系统进行迁移至新的技术栈，主要就是负责老系统的维护工作以及开发部分新功能，开发了一些部署时的自动化工作，把一些繁琐的人力工作进行了自动化，部署一次的成本由2小时缩短至半小时。
  - **工作总结**
    接触了很多第三方的技术，例如gcp的stackdriver日志监控、metrics性能分析，aws的serverless lambda,newrelic以及cloudflare等。完善的敏捷开发工作流，从support的需求下来，到PO的对接，单元测试集成测试，codereview以及上线测试，日志留存以及24小时On-call等，写文档、记录需求变更历史等工作习惯都是从这里学来的。


  ###### 2017.6 ~ 2018.10  &nbsp;&nbsp;&nbsp;&nbsp;全栈开发
  - **在做什么？**
    外包业务以及区块链业务开发，经历了两家公司，从全栈转到后端开发。
  - **做了什么？**  
    1. 对待大同小异的外包业务，在基础架构上抽离出了一套业务模板，在模板的基础上基本一周就能出一套解决方案。
    2. 基于svg图像touch热点，用纯jquery实现了不同地区的工位选座功能。
    3. 参与了区块业务上的法币-币种交易的业务开发。


  <p>&nbsp;</p>

#### 目前在做什么
 
##### 技术栈
 
----
<p>&nbsp;</p>

#### 工作期望
----
