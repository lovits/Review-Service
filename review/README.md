添加了consul服务注册功能
实现追加评价;待审核的申诉记录可修改
重构: 将 submodule 路径从 api/review/v1 修改为 ./api
添加ES消费kafka消息功能之根据商家ID获取评论列表
添加根据用户ID获取评论列表（分页）
添加了redis缓存查询数据，并通过singleflight合并并发请求

# Review微服务技术文档

## 概述

Review服务是一个基于Go语言和Kratos框架开发的微服务系统，主要负责电商平台的评论管理功能。该服务提供了评论的创建、审核、回复、申诉等完整生命周期管理，支持多端（C端用户、B端商家、O端运营）操作。

## 架构设计

### 整体架构
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   HTTP/gRPC     │    │   Service层     │    │   Business层    │
│   接口层        │───▶│   (业务逻辑)    │───▶│   (领域逻辑)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Data层        │    │   Repository    │    │   外部服务      │
│   (数据访问)    │    │   (数据模型)    │    │   (ES/Redis)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 技术栈
- **框架**: Kratos v2.8.4 (Go微服务框架)
- **通信**: gRPC + HTTP REST API
- **数据库**: MySQL 8.0 (主存储)
- **缓存**: Redis 7 (缓存层)
- **搜索**: Elasticsearch 8 (全文搜索)
- **消息队列**: Kafka (异步处理)
- **服务发现**: Consul (服务注册发现)
- **依赖注入**: Google Wire
- **ORM**: GORM v1.30.0
- **ID生成**: Snowflake算法
- **容器化**: Docker + Docker Compose

## 核心功能

### 评论流程
```
用户发表评论 → 待审核状态 → 运营审核 → 已发布/驳回
     ↓
商家回复评论
     ↓
如有异议 → 商家申诉 → 运营审核申诉 → 通过/驳回
```

### API接口

#### 主要RPC方法
- `CreateReview`: C端创建评论
- `GetReview`: 获取评论详情
- `AuditReview`: O端审核评论
- `ReplyReview`: B端回复评论
- `AppealReview`: B端申诉评论
- `AuditAppeal`: O端审核申诉
- `ListReviewByStoreID`: 根据商家ID分页获取评论列表
- `ListReviewByUserID`: 根据用户ID分页获取评论列表

#### HTTP映射
- `POST /v1/review` - 创建评论
- `GET /v1/review/{reviewID}` - 获取评论
- `POST /v1/review/audit` - 审核评论
- `POST /v1/review/reply` - 回复评论
- `POST /v1/review/appeal` - 申诉评论
- `POST /v1/appeal/audit` - 审核申诉
- `GET /v1/review/store/{storeID}` - 获取商家评论列表
- `GET /v1/review/user/{userID}` - 获取用户评论列表

## 数据模型

### 核心表结构

#### review_info (评论信息表)
```sql
- id: 主键ID
- review_id: 评论ID (雪花算法生成)
- user_id: 用户ID
- order_id: 订单ID
- store_id: 店铺ID
- content: 评论内容
- score/service_score/express_score: 评分 (1-5分)
- status: 状态 (10:待审核, 20:已发布, 30:驳回, 40:申诉通过)
- anonymous: 是否匿名
- pic_info/video_info: 媒体信息
- op_user/op_reason/op_remarks: 运营操作信息
```

#### review_reply_info (评论回复表)
```sql
- reply_id: 回复ID
- review_id: 评论ID
- store_id: 店铺ID
- content: 回复内容
- pic_info/video_info: 媒体信息
```

#### review_appeal_info (评论申诉表)
```sql
- appeal_id: 申诉ID
- review_id: 评论ID
- store_id: 店铺ID
- status: 申诉状态 (10:待审核, 20:通过, 30:驳回)
- reason/content: 申诉原因和内容
- op_user/op_remarks: 运营审核信息
```

## 配置管理

### 配置文件结构
```yaml
server:
  http:
    addr: 0.0.0.0:8000
    timeout: 1s
  grpc:
    addr: 0.0.0.0:9000
    timeout: 1s

data:
  database:
    driver: mysql
    source: root:password@tcp(127.0.0.1:3306)/review_system
  redis:
    addr: 127.0.0.1:6379
    read_timeout: 0.2s
    write_timeout: 0.2s

snowflake:
  start_time: "2025-06-13"
  machine_id: 1

elasticsearch:
  addresses:
    - http://127.0.0.1:9200

consul:
  address: 127.0.0.1:8500
  scheme: http
```

## 部署架构

### 基础设施组件
- **MySQL**: 主数据库，存储评论相关数据
- **Redis**: 缓存层，提升查询性能，支持Singleflight合并并发请求
- **Elasticsearch**: 搜索服务，支持根据商家/用户ID搜索评论
- **Kafka**: 消息队列，异步处理评论数据同步到ES
- **Consul**: 服务注册中心，实现服务发现
- **Canal**: MySQL binlog监听，实时同步数据到Kafka

### 容器化部署
```dockerfile
# 构建阶段
FROM golang:1.19 AS builder
RUN make build

# 运行阶段
FROM debian:stable-slim
EXPOSE 8000 9000
CMD ["./server", "-conf", "/data/conf"]
```

### Docker Compose服务编排
包含完整的开发环境：MySQL、Redis、Consul、Kafka、Zookeeper等

## 代码组织

### 目录结构
```
review/
├── api/                    # API定义 (Protobuf)
│   └── review/v1/
├── cmd/review/            # 应用入口
│   ├── main.go
│   └── wire.go           # 依赖注入
├── internal/
│   ├── biz/              # 业务逻辑层
│   ├── data/             # 数据访问层
│   ├── server/           # 服务器配置
│   └── service/          # 服务层
├── pkg/                  # 公共包
│   └── snowflake/       # ID生成器
├── configs/              # 配置文件
├── third_party/          # 第三方依赖
├── Dockerfile
├── Makefile
├── docker-compose.yaml
└── review.sql           # 数据库初始化脚本
```

### 依赖注入
使用Google Wire进行依赖注入，ProviderSet模式组织各层依赖。

## 性能优化

### 缓存策略
- Redis缓存评论查询结果
- Singleflight模式合并并发请求，减少缓存穿透

### 异步处理
- Kafka消息队列异步同步评论数据到Elasticsearch
- Canal监听MySQL binlog，实时数据同步

### ID生成
- 雪花算法生成全局唯一ID
- 支持分布式部署的ID唯一性

## 监控和日志

### 日志系统
- 集成Kratos日志中间件
- 支持trace_id和span_id追踪
- 结构化日志输出

### 健康检查
- HTTP健康检查端点
- gRPC健康检查服务

## 开发和构建

### 环境准备
```bash
# 安装依赖工具
make init

# 生成API代码
make api

# 构建应用
make build
```

### 本地开发
```bash
# 启动基础设施
docker-compose up -d

# 运行应用
./bin/server -conf ./configs
```

## 扩展性考虑

### 水平扩展
- 无状态设计，支持多实例部署
- Consul服务发现自动负载均衡

### 数据扩展
- Elasticsearch支持全文搜索和复杂查询
- 支持评论数据的历史归档

### 功能扩展
- 插件化架构，易于添加新功能
- 事件驱动设计，支持业务流程扩展

## 安全考虑

### 数据验证
- Protobuf内置字段验证
- 业务层数据校验

### 访问控制
- 支持用户身份认证
- 不同端（C/B/O）权限隔离

### 敏感信息
- 数据库密码等敏感信息通过环境变量配置
- 日志脱敏处理

## 总结

Review服务是一个功能完整、架构清晰的微服务系统，采用了现代化的技术栈和最佳实践。通过Kratos框架的支撑，实现了高性能、可扩展的评论管理系统。整个系统设计考虑了生产环境的各种需求，包括缓存、异步处理、服务发现等，是一个优秀的微服务参考实现。