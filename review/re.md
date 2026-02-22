领域	技术选型	说明
语言	Golang	1.18+
框架	Kratos v2	微服务核心框架
API	Protobuf, gRPC, HTTP	接口定义与通信
依赖注入	Google Wire	编译期依赖注入
数据库	MySQL 8.0	事务性数据存储
ORM	GORM Gen	类型安全的代码生成器，替代手写 SQL
搜索	Elasticsearch	支持多维度的评价检索
缓存	Redis	缓存 C 端热点数据
消息队列	Kafka	异步解耦，数据同步
并发控制	Singleflight	防止缓存击穿，合并并发请求
ID生成	Snowflake	分布式全局唯一 ID
服务发现	Consul	服务注册与发现
配置管理	YAML / Proto	强类型配置管理