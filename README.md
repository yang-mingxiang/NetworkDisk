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



# 项目结构
# 1. 仓库结构

仓库整体分为前端、后端服务、公共组件、配置与 DDL 四层。

### 前端

- `disk-by-cursor`

当前前端技术栈是 Vue 3 + Vite + Element Plus。开发联调时，Vite 会把不同接口前缀分别代理到不同业务服务端口，而不是统一走网关。

### 核心后端服务

- `networkdisk-auth`
- `networkdisk-gateway`
- `networkdisk-business/networkdisk-files`
- `networkdisk-business/networkdisk-share`
- `networkdisk-business/networkdisk-recycle`
- `networkdisk-business/networkdisk-user`
- `networkdisk-business/networkdisk-ai`

这几个模块构成了当前项目的主干服务。

### 预留或轻量模块

- `networkdisk-business/networkdisk-app`
- `networkdisk-business/networkdisk-notice`
- `networkdisk-business/networkdisk-engine`

### 公共组件层

`networkdisk-common` 下沉了大量跨服务复用能力，主要包括：

- `networkdisk-api`：Dubbo Facade 接口、DTO、请求响应对象
- `networkdisk-web`：统一返回体、全局异常、登录切面、序列化等 Web 公共能力
- `networkdisk-rpc`：Dubbo 相关基础设施
- `networkdisk-file`：文件存储引擎抽象与实现
- `networkdisk-es`：ES 配置封装
- `networkdisk-mq`：基于 Spring Cloud Stream 的消息发送封装
- `networkdisk-lock`：分布式锁
- `networkdisk-delay-message`：基于 Redisson 的延迟消息能力
- `networkdisk-cache`：Redis、Redisson、JetCache 配置
- `networkdisk-config`：Nacos 配置导入

### 配置与文档资源

- 根目录 `networkdisk-config/`：DDL、图示资源
- `networkdisk-common/networkdisk-config`：被服务端模块导入的公共配置资源

# 2. 总体架构

从当前代码实现看，系统可以概括成下面这条链路：

```text
前端层
  -> disk-by-cursor

接入层
  -> networkdisk-gateway（统一路由、Sa-Token 过滤）
  -> 开发联调时主要还是前端直连各服务端口

业务服务层
  -> auth：认证编排
  -> user：用户信息
  -> files：文件核心域
  -> share：分享域
  -> recycle：回收站域
  -> ai：AI 文档增强

公共基础层
  -> API / RPC / Web / Lock / MQ / DelayQueue / File / Cache / ES

存储与中间件层
  -> MySQL
  -> Redis + Redisson
  -> Elasticsearch
  -> PostgreSQL + pgvector
  -> Nacos
  -> Canal
  -> RocketMQ
  -> Local / FastDFS / OSS
```

# 3. 各服务职责

## 3.1 auth

`networkdisk-auth` 负责注册、登录、退出等认证入口。

当前注册链路是：

```text
auth.register
  -> userFacade.register
  -> user 表落库
  -> filesFacade.createUserRootFile
  -> 创建用户根目录
```

当前登录链路是：

```text
auth.login
  -> userFacade.query
  -> StpUtil.login(userId)
  -> 生成 JWT
  -> Redis 保存 USER_LOGIN_{userId} = token
  -> 返回 token 给前端
```

也就是说，当前认证体系登录时调用了 Sa-Token，同时业务服务侧依赖 JWT + Redis 做登录态校验。

## 3.2 gateway

`networkdisk-gateway` 基于 Spring Cloud Gateway，主要承担：统一入口、跨域配置、Sa-Token 登录校验以及路由转发。

## 3.3 files

`networkdisk-files` 是项目里最核心的业务服务，负责：

- 文件列表、面包屑、目录树
- 新建文件夹、重命名、复制、移动
- 文件逻辑删除
- 秒传、分片上传、合并
- 文件预览、下载
- 文件搜索
- 上传后触发 AI 预热

这个模块还承载了项目最关键的数据模型拆分：

- `user_file`：用户视角的文件树
- `file`：物理文件元数据

这层拆分实现“复制、移动、回收站、秒传”都可以只改用户侧映射关系，不用反复操作物理文件。

## 3.4 share

`networkdisk-share` 负责分享域能力：

- 创建分享
- 查询分享列表和详情
- 分享码校验
- 匿名查看分享文件
- 分享文件下载
- 分享过期清理

分享过期目前是双保险：

- 访问时实时校验 `shareEndTime`
- 创建时投递 Redisson 延迟任务，到期做逻辑删除

## 3.5 recycle

`networkdisk-recycle` 负责回收站能力：

- 查询已删除文件列表
- 批量恢复
- 批量彻底删除

回收站没有单独建表，而是直接基于 `user_file.deleted != 0` 组织二阶段删除。

## 3.6 user

`networkdisk-user` 负责用户信息域：

- 用户注册落库
- 用户信息查询
- 用户根目录信息聚合

这个模块里已经用了 JetCache 做用户查询缓存，也用了 Redisson BloomFilter 和 SortedSet 存一些用户侧辅助结构，但整体上它仍是一个相对轻量的业务服务。

## 3.7 ai

`networkdisk-ai` 是围绕文档理解能力拆出来的独立服务，目前提供的功能有：

- 文档索引
- 文档摘要
- 文档标签
- 单文件问答

# 4. 数据模型设计

详细介绍可以看实践篇的通用设计。

## 4.1 主业务表

根据当前 DDL，主业务表包括：

- `user`
- `file`
- `user_file`
- `file_chunk`
- `share`
- `share_file`
- `user_search_history`
- `error_log`

其中最关键的是 `user_file` 和 `file` 分离。

## 4.2 AI 扩展表

如果开启 pgvector，AI 模块还会使用：

- `ai_document_index`
- `ai_document_chunk_vector`
- `ai_document_result`

这三张表分别承担：

- 文档级索引元信息
- chunk 向量存储
- 摘要与标签结果缓存

# 5. 存储与中间件

## 5.1 MySQL

主业务数据仍然落在 MySQL，连接池用的是 Druid，ORM 主要是 MyBatis-Plus + XML Mapper。

## 5.2 Redis / Redisson / JetCache

Redis 主要承担：

- 登录态 token 存储
- Redisson 分布式锁
- Redisson 延迟队列
- JetCache 远程缓存

JetCache 当前更偏局部使用，不是全局大规模缓存方案。

## 5.3 Elasticsearch

ES 用于文件搜索，当前配置启用了 Easy-ES，搜索同步不是业务双写，而是 Canal 订阅 `user_file` binlog 做异步同步。

## 5.4 PostgreSQL + pgvector

AI 模块的向量检索能力不走 MySQL，而是单独使用 PostgreSQL + pgvector。这样文件主业务库和 AI 向量库是分离的。

## 5.5 Nacos + Dubbo

Nacos 负责：Spring 配置中心和服务发现。

Dubbo 负责服务间 RPC，当前项目里跨服务调用基本都走 Facade 接口，例如：auth 调 user、auth 调 files，或者 ai 调 files。

## 5.6 RocketMQ + Spring Cloud Stream

消息能力目前主要用于 AI 文档预热。项目不是自己直接操作底层 RocketMQ 客户端，而是通过 Spring Cloud Stream + StreamBridge 发消息。

## 5.7 文件存储引擎

`networkdisk-file` 已经抽象出了统一的 `StorageEngine`，代码层支持：

- Local
- FastDFS
- OSS

也就是说，业务层不直接绑定某一种物理存储实现，而是依赖存储引擎抽象。

# 6. 当前运行端口

可以从 `application.yml` 看到，当前服务端口如下：

- `gateway`：8081
- `files`：8082
- `recycle`：8084
- `share`：8085
- `user`：8086
- `ai`：8087
- `auth`：8090

前端开发时，`disk-by-cursor/vite.config.js` 也是按这些端口做代理的。
