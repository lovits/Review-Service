# Review Service 启动指南

## 文档概述

本文档详细讲解 Review 服务的启动流程，包括各阶段的功能、关键组件及执行顺序。

---

## 快速启动

```bash
cd Kratos8/review/cmd/review
go run main.go -conf ../../configs
```

或指定自定义配置路径：
```bash
go run main.go -conf /path/to/config.yaml
```

---

## 完整启动流程

### 第一步：命令行参数解析

**代码位置**：`init()` 函数

```go
func init() {
	flag.StringVar(&flagconf, "conf", "../../configs", "config path, eg: -conf config.yaml")
}
```

**功能**：
- 注册 `-conf` 命令行参数
- 默认值为 `../../configs`（相对于 main.go 的配置目录）
- 允许用户通过参数覆盖默认路径

**执行时机**：Go 程序加载前自动执行

---

### 第二步：解析命令行参数 & 初始化日志

**代码位置**：`main()` 函数开始

```go
flag.Parse()
logger := log.With(log.NewStdLogger(os.Stdout),
"ts", log.DefaultTimestamp,
"caller", log.DefaultCaller,
"service.id", id,
"service.name", Name,
"service.version", Version,
"trace.id", tracing.TraceID(),
	"span.id", tracing.SpanID(),
)
```

**功能**：

| 步骤 | 说明 |
|------|------|
| `flag.Parse()` | 解析所有注册的命令行参数，填充到变量 `flagconf` 中 |
| `log.NewStdLogger(os.Stdout)` | 创建标准日志记录器，输出到控制台 |
| `log.With(...)` | 向日志添加结构化字段（Key-Value 对） |

**日志字段说明**：

| 字段 | 来源 | 说明 |
|------|------|------|
| `ts` | DefaultTimestamp | 当前时间戳 |
| `caller` | DefaultCaller | 调用函数的文件和行号 |
| `service.id` | `id` | 服务实例 ID（机器主机名） |
| `service.name` | `Name` | 服务名称 `"review.service"` |
| `service.version` | `Version` | 服务版本 `"0.0.1"` |
| `trace.id` | `tracing.TraceID()` | 分布式链路追踪 ID |
| `span.id` | `tracing.SpanID()` | 链路追踪的 Span ID |

**输出示例**：
```
ts=2026-01-27T10:30:45Z caller=main.go:70 service.id=MacBook-Pro service.name=review.service service.version=0.0.1 trace.id=xyz123 span.id=abc456
```

---

### 第三步：加载配置文件

**代码位置**：

```go
c := config.New(
config.WithSource(
file.NewSource(flagconf),
),
)
defer c.Close()

if err := c.Load(); err != nil {
	panic(err)
}
```

**功能流程**：

1. **创建配置对象**：`config.New()` - 使用 Kratos 的配置管理器
2. **指定配置源**：`file.NewSource(flagconf)` - 从文件系统读取配置
3. **延迟关闭**：`defer c.Close()` - 程序退出时释放配置连接
4. **加载配置**：`c.Load()` - 读取配置文件内容

**预期的配置文件结构**：

```yaml
# configs/config.yaml 示例

# 服务器配置
server:
  http:
    addr: ":8080"
    timeout: 10s
  grpc:
    addr: ":9090"
    timeout: 10s

# 数据库配置
data:
  database:
    driver: "mysql"
    source: "user:password@tcp(localhost:3306)/review_db"

# 注册中心配置
registry:
  consul:
    address: "localhost:8500"

# 雪花算法配置
snowflake:
  startTime: 1609459200
  machineId: 1

# Elasticsearch 配置（可选）
elasticsearch:
  address: "http://localhost:9200"
```

---

### 第四步：扫描配置到结构体

**代码位置**：

```go
var bc conf.Bootstrap
if err := c.Scan(&bc); err != nil {
	panic(err)
}

var rc conf.Registry
if err := c.Scan(&rc); err != nil {
	panic(err)
}
```

**功能**：

- **`bc` (Bootstrap)**：启动配置结构体，包含服务器、数据库、雪花、Elasticsearch 等配置
- **`rc` (Registry)**：注册中心配置结构体，包含服务发现相关配置

**数据流向**：

```
YAML配置文件
    ↓
c.Load() 读取
    ↓
c.Scan(&bc) 映射 → conf.Bootstrap
c.Scan(&rc) 映射 → conf.Registry
```

**预期的结构定义**（在 `internal/conf/conf.go` 中）：

```go
type Bootstrap struct {
	Server        *Server
	Data          *Data
	Snowflake     *Snowflake
	Elasticsearch *Elasticsearch
}

type Registry struct {
	Consul *Consul  // 或其他注册中心
}

type Server struct {
	HTTP *HTTP
	GRPC *GRPC
}

type Data struct {
	Database *Database
	Redis    *Redis  // 可选
}

type Snowflake struct {
	StartTime int64
	MachineId int64
}

type Elasticsearch struct {
	Address string
	// 其他配置
}
```

---

### 第五步：依赖注入（Wire）

**代码位置**：

```go
app, cleanup, err := wireApp(bc.Server, bc.Data, logger, &rc, bc.Elasticsearch)
if err != nil {
	panic(err)
}
defer cleanup()
```

**功能**：

`wireApp` 是由 Google Wire 自动生成的依赖注入函数。它负责：

1. **初始化数据层**：
   - 创建数据库连接池
   - 连接 Redis（如果有）
   - 创建 DAO/Repository 对象

2. **初始化业务层**：
   - 创建各个 Service 对象
   - Service 引用 DAO

3. **初始化传输层**：
   - 创建 gRPC Server
   - 创建 HTTP Server
   - 注册路由和 RPC 处理器

4. **初始化注册中心**：
   - 连接 Consul、Nacos 等服务发现中心
   - 创建 Registrar 对象

5. **返回应用对象**：
   - `app` - 完全初始化的 Kratos App
   - `cleanup` - 清理函数，释放资源
   - `err` - 初始化过程中的错误

**Wire 工作流程图**：

```
配置 (bc.Server, bc.Data, ...)
    ↓
[wireApp 自动生成的代码]
    ↓
Database Connection ──→ DAO/Repository
                         ↓
                      Service Layer
                         ↓
                    gRPC Handler
                    HTTP Handler
                         ↓
    Registry Center ←─→ Registrar
                         ↓
                    Kratos App
    ↓
返回 (app, cleanup, err)
```

---

### 第六步：初始化雪花算法

**代码位置**：

```go
if err := snowflake.Init(bc.Snowflake.StartTime, bc.Snowflake.MachineId); err != nil {
	panic(err)
}
```

**功能**：

- **Snowflake 算法**：分布式环境下的唯一 ID 生成算法
- **参数说明**：
  - `StartTime` - 起始时间戳（用于计算时间差）
  - `MachineId` - 机器 ID（1-31，用于区分不同服务器）

**生成的 ID 格式**：

```
64 bit ID:
┌─────────────────────┬───────────┬──────────────┐
│   时间戳 (41bit)    │ 机器ID    │  序列号      │
│   (毫秒)            │ (10bit)   │  (12bit)     │
└─────────────────────┴───────────┴──────────────┘
```

**使用场景**：
- 生成业务记录的唯一 ID
- 生成链路追踪 ID
- 生成分布式事务 ID

---

### 第七步：启动服务

**代码位置**：

```go
if err := app.Run(); err != nil {
	panic(err)
}
```

**功能**：

1. **启动 gRPC Server**
   - 监听配置中指定的端口（如 `:9090`）
   - 等待 RPC 调用

2. **启动 HTTP Server**
   - 监听配置中指定的端口（如 `:8080`）
   - 等待 HTTP 请求

3. **向注册中心注册**
   - 将服务信息（名称、地址、端口等）发送给 Consul/Nacos
   - 启用服务发现

4. **阻塞等待**
   - 程序进入主循环，监听系统信号
   - 收到 SIGINT/SIGTERM 时执行优雅关闭

**启动后输出示例**：

```
ts=2026-01-27T10:30:46Z caller=main.go:95 msg="start grpc server" addr=:9090
ts=2026-01-27T10:30:46Z caller=main.go:95 msg="start http server" addr=:8080
ts=2026-01-27T10:30:47Z caller=main.go:95 msg="register to consul" service=review.service
```

---

## 完整启动流程时序图

```
应用启动
  │
  ├─ [1] flag.Parse()
  │       ↓
  ├─ [2] 初始化日志
  │       ↓
  ├─ [3] 创建配置对象
  │       ├─ 指定配置源（文件）
  │       └─ 加载配置文件
  │       ↓
  ├─ [4] 扫描配置到结构体
  │       ├─ bc (Bootstrap)
  │       └─ rc (Registry)
  │       ↓
  ├─ [5] 依赖注入 (Wire)
  │       ├─ 初始化数据层 (DB)
  │       ├─ 初始化业务层 (Service)
  │       ├─ 初始化传输层 (gRPC/HTTP)
  │       └─ 返回 app & cleanup
  │       ↓
  ├─ [6] 初始化雪花算法
  │       ↓
  ├─ [7] 启动服务 (app.Run())
  │       ├─ 启动 gRPC Server
  │       ├─ 启动 HTTP Server
  │       ├─ 向注册中心注册
  │       └─ 阻塞等待信号
  │       ↓
  └─ [8] 收到关闭信号
          ├─ 执行 cleanup()
          └─ 程序退出
```

---

## 关键组件说明

### 1. Logger（日志记录器）

**用途**：记录程序运行日志，便于排查问题

**特性**：
- 输出到标准输出（控制台）
- 包含时间戳、调用位置、服务信息
- 支持分布式链路追踪 ID 注入

---

### 2. Config（配置管理）

**用途**：从 YAML 文件读取配置，映射到 Go 结构体

**支持的配置源**：
- 本地文件
- 环境变量
- 远程配置中心（Consul、Apollo 等）

---

### 3. Snowflake（分布式 ID 生成）

**用途**：生成在分布式系统中全局唯一的 ID

**应用场景**：
- 业务表的主键 ID
- 订单号、请求 ID 等唯一标识

---

### 4. Wire（依赖注入）

**用途**：自动生成组件初始化代码，管理依赖关系

**优势**：
- 代码自动生成，避免手写初始化逻辑
- 依赖关系清晰明了
- 易于单元测试

**生成命令**：
```bash
cd Kratos8/review
wire ./cmd/review
```

---

### 5. Registry（服务注册）

**用途**：将服务注册到服务发现中心

**支持的注册中心**：
- Consul
- Nacos
- Etcd
- ZooKeeper

**工作流程**：
```
服务启动
  ↓
连接注册中心
  ↓
上报服务信息（IP、端口、版本等）
  ↓
定期心跳检测（Keep Alive）
  ↓
服务优雅关闭时下线
```

---

## 错误处理

当前代码使用 `panic` 处理所有严重错误：

```go
if err := c.Load(); err != nil {
	panic(err)  // ❌ 直接崩溃
}
```

**生产建议**：改为更优雅的错误处理

```go
if err := c.Load(); err != nil {
	logger.Errorf("failed to load config: %v", err)
	os.Exit(1)  // ✅ 记录日志后退出
}
```

---

## 环境变量配置

常用环境变量（可在启动前设置）：

```bash
# 日志级别
export KRATOS_LOG_LEVEL=debug

# 配置文件路径
export KRATOS_CONF=/etc/review/config.yaml

# 运行模式
export KRATOS_ENV=production

# 启动服务
./review -conf $KRATOS_CONF
```

---

## 常见问题 FAQ

### Q1: 怎样修改监听的端口？

**A**: 编辑 `configs/config.yaml`，修改 `server.grpc.addr` 和 `server.http.addr`

```yaml
server:
  grpc:
    addr: ":9090"  # 修改这里
  http:
    addr: ":8080"  # 修改这里
```

### Q2: 如何连接到不同的数据库？

**A**: 编辑 `configs/config.yaml`，修改 `data.database.source`

```yaml
data:
  database:
    source: "user:password@tcp(host:port)/dbname"
```

### Q3: 服务无法注册到 Consul？

**A**: 检查以下项：
1. Consul 服务是否运行：`consul agent -dev`
2. 配置文件中的 Consul 地址是否正确
3. 防火墙是否开放 8500 端口

### Q4: 如何让服务优雅关闭？

**A**: 程序会自动处理，按 `Ctrl+C` 即可：
- 停止接收新请求
- 等待现有请求完成
- 关闭数据库连接
- 从注册中心下线

---

## 构建与部署

### 本地开发运行

```bash
cd Kratos8/review/cmd/review
go run main.go -conf ../../configs
```

### 构建二进制文件（带版本号）

```bash
go build -ldflags "-X main.Version=1.0.0" -o review ./cmd/review
```

### Docker 运行

```bash
docker build -t review:latest .
docker run -p 8080:8080 -p 9090:9090 review:latest
```

### 使用 Makefile

```bash
cd Kratos8/review
make run      # 运行服务
make build    # 构建二进制
make test     # 运行测试
make docker   # 构建 Docker 镜像
```

---

## 相关文件索引

| 文件 | 说明 |
|------|------|
| `cmd/review/main.go` | 服务启动入口 |
| `cmd/review/wire.go` | Wire 依赖注入定义 |
| `cmd/review/wire_gen.go` | Wire 自动生成的代码 |
| `configs/config.yaml` | 配置文件 |
| `internal/conf/conf.go` | 配置结构体定义 |
| `pkg/snowflake/snowflake.go` | 雪花 ID 实现 |
| `internal/service/` | 业务层代码 |
| `internal/data/` | 数据层代码 |
| `api/proto/` | Protocol Buffer 定义 |

---

## 总结

Review Service 的启动流程遵循**企业级微服务标准**：

1. ✅ **配置集中管理** - 从 YAML 读取，支持多环境
2. ✅ **日志结构化** - 便于日志聚合与链路追踪
3. ✅ **依赖注入** - 代码自动生成，易于维护
4. ✅ **服务注册** - 自动发现，支持分布式部署
5. ✅ **优雅关闭** - 正确释放资源，避免数据丢失
6. ✅ **分布式 ID** - Snowflake 算法确保 ID 唯一

这样的架构适合于生产环境的微服务部署。

---

## 附录：Kratos 框架概述

[Kratos](https://go-kratos.dev/) 是一套轻量化的 Go 微服务框架，提供：

- **多协议支持**：gRPC + HTTP/REST
- **配置管理**：支持多种配置源
- **服务发现**：支持多种注册中心
- **中间件**：链路追踪、日志、认证等
- **代码生成**：基于 Protocol Buffer 和 Wire 自动生成

**官方文档**：https://go-kratos.dev/
**GitHub**：https://github.com/go-kratos/kratos

