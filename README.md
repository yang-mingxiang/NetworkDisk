# NetworkDisk 项目介绍

## 一、项目介绍

该项目是一个基于 **Spring Boot 3 + Dubbo 3 + Vue 3** 构建的前后端分离、多模块网盘系统，核心业务包括认证、文件管理、搜索、分享、回收站，以及新增的 AI 文档分析能力。

前端采用 **Vue 3 + Vite + Element Plus** 实现，通过网关和业务接口完成交互。

后端采用多模块 Maven 工程，主要模块包括 `auth`、`gateway`、`files`、`share`、`recycle`、`user`、`ai` 等。

## 二、技术选型与架构设计

- **前端技术：** Vue 3 + Vite + Element Plus
- **核心框架：** Spring Boot 3.2.2 + Spring MVC
- **网关与服务治理：** Spring Cloud Gateway + Nacos
- **服务通信：** Dubbo 3
- **数据库访问：** MySQL + MyBatis-Plus
- **缓存与分布式能力：** Redis + JetCache + Redisson
- **消息驱动：** Spring Cloud Stream + RocketMQ
- **搜索能力：** Elasticsearch + Canal
- **文件存储：** StorageEngine 抽象 + Local / OSS / FastDFS 多实现
- **AI 能力：** Apache Tika + OpenAI-compatible Provider（当前默认配置为 DashScope/Qwen 兼容接口）+ PostgreSQL + pgvector

## 三、核心功能

### 3.1 用户模块

- **注册：** 创建用户并初始化用户根目录。
- **登录：** 服务端调用 Sa-Token 登录，同时签发 JWT，并将 JWT 写入 Redis。
- **鉴权：** 业务接口主要通过 AOP 统一校验 `Authorization` 中的 JWT，并比对 Redis 中的登录态；部分接口通过 `@LoginIgnore` 放行。
- **退出：** 清理 Redis 登录信息，并注销登录态。

### 3.2 文件模块

- 文件列表、面包屑、目录树。
- 新建文件夹、重命名、移动、复制、逻辑删除。
- 单文件上传、秒传、分片上传、断点续传、分片合并。
- 文件下载、在线预览。

### 3.3 搜索模块

- 支持关键字检索、拆词/子串模糊命中、高亮返回。
- 优先使用 Elasticsearch 查询，ES 未命中或不可用时回退 MySQL。
- 通过 Canal 实现 `user_file` → `user_file_index` 的增量同步。

### 3.4 分享模块

- 创建分享、取消分享、分享详情查询。
- 匿名访问分享页（`@LoginIgnore`）。
- 提取码校验、分享文件列表、分享文件下载。
- **分享过期控制：** 访问时校验 + Redisson 延迟队列自动清理。

### 3.5 回收站模块

- 查看已删除文件。
- 批量还原（恢复 `deleted=0`）。
- 批量彻底删除。
- 支持选中目录时递归处理子节点。

### 3.6 AI 模块

- 提供 AI 能力探测、文档建索引、生成摘要、生成标签、单文件问答接口。
- 文件服务会在上传、秒传、分片合并成功后，对支持的文档类型异步发送“AI 预热”消息。
- AI 服务消费消息后，执行**建索引 + 摘要 + 标签**预处理，减少用户首次打开文档时的等待时间。
- 单文件问答会优先通过 pgvector 检索相关片段，再将片段上下文交给大模型回答。

## 四、项目亮点

### 4.1 文件存储抽象

通过 `StorageEngine` 解耦业务与底层存储实现，业务层不直接依赖本地磁盘、OSS 或 FastDFS 的具体 SDK。

### 4.2 大文件传输能力

- 支持分片上传、断点续传、秒传、分片合并。
- 下载和预览采用流式处理，降低大文件 IO 的内存压力。

### 4.3 搜索双通路设计

- 平时优先使用 Elasticsearch 提供更好的检索性能，同时通过 Canal 实现 `user_file` → `user_file_index` 的增量同步。
- ES 不可用或未同步完成时，仍可回退到 MySQL，保证搜索功能的可用性。

### 4.4 统一鉴权与 ID 安全

- 登录态校验通过 **JWT + Redis + AOP** 组合完成。
- 多数文件 ID 对外使用加密字符串，避免直接暴露数据库主键。

### 4.5 分享延迟清理

使用 Redisson 延迟队列调度过期分享的清理任务，避免依赖单机定时轮询。

### 4.6 AI 异步预热与检索增强问答

- 文件上传后不阻塞用户请求，而是异步完成索引、摘要、标签生成。
- 单文件问答先进行向量检索再提问，属于典型的轻量级 RAG 落地方案。
