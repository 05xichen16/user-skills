# [项目/功能名称] 软件设计文档

> 版本：v1.0  
> 作者：  
> 日期：  
> 状态：草稿 / 评审中 / 已定稿

---

## 第一章 需求背景

<!-- 用简明要点列出业务痛点和项目目标，每条具体可验证 -->

1. [痛点1：现状描述 → 导致的问题]
2. [痛点2：现状描述 → 导致的问题]
3. [目标：期望达成的效果]

---

## 第二章 需求分析

### 2.1 [功能模块1名称]

功能描述：[简要说明该模块做什么]

用户故事：作为[角色]，我希望[操作]，以便[价值]。

验收标准：
• [标准1]

• [标准2]


### 2.2 [功能模块2名称]

功能描述：[简要说明]

用户故事：作为[角色]，我希望[操作]，以便[价值]。

验收标准：
• [标准1]

• [标准2]


---

## 第三章 4+1 视图

### 3.1 系统架构总览

<!-- 使用 PlantUML 组件图展示完整系统架构 -->

```plantuml
@startuml
title 系统架构图

package "前端" {
  [Web应用]
}

package "网关层" {
  [API Gateway]
}

package "服务层" {
  [服务A]
  [服务B]
}

package "数据层" {
  database "主数据库" as DB
  database "缓存" as Cache
  queue "消息队列" as MQ
}

[Web应用] --> [API Gateway]
[API Gateway] --> [服务A]
[API Gateway] --> [服务B]
[服务A] --> DB
[服务A] --> Cache
[服务B] --> MQ
[服务B] --> DB
@enduml
```

### 3.2 用例视图

<!-- 展示系统参与者和核心用例 -->

```plantuml
@startuml
left to right direction
skinparam packageStyle rectangle

actor "用户" as User
actor "管理员" as Admin

rectangle "系统名称" {
  usecase "用例1" as UC1
  usecase "用例2" as UC2
  usecase "用例3" as UC3
  usecase "用例4（管理）" as UC4
}

User --> UC1
User --> UC2
User --> UC3
Admin --> UC4
Admin --> UC1
@enduml
```

### 3.3 逻辑视图

<!-- 类图展示核心领域模型，体现 SOLID 原则 -->

```plantuml
@startuml
skinparam classAttributeIconSize 0

interface IRepository<T> {
  +findById(id: Long): T
  +save(entity: T): T
  +delete(id: Long): void
}

interface IService {
  +execute(request: Request): Response
}

class ServiceImpl implements IService {
  -repository: IRepository
  -validator: IValidator
  +execute(request: Request): Response
}

class Entity {
  -id: Long
  -name: String
  -createdAt: DateTime
  +validate(): boolean
}

class RepositoryImpl implements IRepository {
  -dataSource: DataSource
  +findById(id: Long): Entity
  +save(entity: Entity): Entity
  +delete(id: Long): void
}

ServiceImpl --> IRepository : 依赖倒置
RepositoryImpl ..|> IRepository
ServiceImpl ..> Entity
@enduml
```

### 3.4 过程视图

<!-- 时序图展示核心交互流程 -->

#### 3.4.1 [流程1名称] 时序图

```plantuml
@startuml
actor 用户
participant "前端" as FE
participant "后端服务" as BE
participant "数据库" as DB

用户 -> FE: 发起操作
activate FE
FE -> BE: POST /api/resource
activate BE
BE -> BE: 参数校验
BE -> DB: INSERT/SELECT
activate DB
DB --> BE: 返回结果
deactivate DB
BE --> FE: 200 OK + 响应体
deactivate BE
FE --> 用户: 展示结果
deactivate FE
@enduml
```

#### 3.4.2 [流程2名称] 时序图

```plantuml
@startuml
' 根据实际流程填写
@enduml
```

### 3.5 开发视图

<!-- 包图展示模块结构和依赖关系 -->

```plantuml
@startuml
package "Presentation Layer" {
  [Controller]
  [DTO]
}

package "Business Layer" {
  [Service]
  [Domain Model]
}

package "Data Access Layer" {
  [Repository]
  [Entity]
}

package "Infrastructure" {
  [Config]
  [Utils]
}

[Controller] --> [Service]
[Controller] ..> [DTO]
[Service] --> [Repository]
[Service] --> [Domain Model]
[Repository] --> [Entity]
[Service] --> [Config]
@enduml
```

### 3.6 部署视图

```plantuml
@startuml
node "负载均衡" {
  [Nginx / LB]
}

node "应用集群" {
  node "节点1" {
    [应用实例1]
  }
  node "节点2" {
    [应用实例2]
  }
}

database "数据库集群" {
  [主库]
  [从库]
}

cloud "外部服务" {
  [第三方API]
}

[Nginx / LB] --> [应用实例1]
[Nginx / LB] --> [应用实例2]
[应用实例1] --> [主库]
[应用实例2] --> [从库]
[应用实例1] --> [第三方API]
@enduml
```

---

## 第四章 流程设计

### 4.1 [核心流程1名称]

```plantuml
@startuml
start
:用户发起请求;

if (参数校验通过?) then (是)
  :查询相关数据;
  if (数据存在?) then (是)
    :执行业务处理;
    :保存结果;
    :返回成功响应;
  else (否)
    :返回数据不存在错误;
  endif
else (否)
  :返回参数错误;
endif

stop
@enduml
```

### 4.2 [核心流程2名称]

```plantuml
@startuml
' 根据实际流程填写
@enduml
```

---

## 第五章 数据库设计

### 5.1 ER 图

```plantuml
@startuml
entity "表A" as A {
  * id : bigint <<PK>>
  --
  * name : varchar(128)
  * status : tinyint
  description : text
  * created_at : datetime
  * updated_at : datetime
}

entity "表B" as B {
  * id : bigint <<PK>>
  --
  * a_id : bigint <<FK>>
  * type : varchar(64)
  value : varchar(256)
  * created_at : datetime
}

A ||--o{ B : "一对多"
@enduml
```

### 5.2 表清单

| 表名称 | 表功能描述 |
|--------|-----------|
| t_xxx_a | [表A功能描述] |
| t_xxx_b | [表B功能描述] |

---

## 第六章 DFX 分析

| DFX问题 | 问题类别 | 问题描述 | 解决方案 | 备注 |
|---------|---------|---------|---------|------|
| [问题1] | 性能 | [具体描述] | [具体方案] | |
| [问题2] | 可靠性 | [具体描述] | [具体方案] | |
| [问题3] | 安全 | [具体描述] | [具体方案] | |
| [问题4] | 可维护性 | [具体描述] | [具体方案] | |

---

## 接口设计（可选）

### API-1：[接口名称]

• Method: POST / GET / PUT / DELETE

• URL: `/v1/resource/action`

• 描述: [接口用途]


请求体:
```json
{
  "field1": "string",
  "field2": 0
}
```

响应体:
```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

错误码:

| 错误码 | 说明 |
|-------|------|
| 400 | 参数错误 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

---

## 表设计（可选）

### t_xxx_a

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|--------|------|---------|--------|------|
| id | bigint | 是 | 自增 | 主键 |
| name | varchar(128) | 是 | - | 名称 |
| status | tinyint | 是 | 0 | 状态：0-禁用 1-启用 |
| description | text | 否 | NULL | 描述 |
| created_at | datetime | 是 | CURRENT_TIMESTAMP | 创建时间 |
| updated_at | datetime | 是 | CURRENT_TIMESTAMP | 更新时间 |

索引:
| 索引名 | 字段 | 类型 | 说明 |
|--------|------|------|------|
| pk_id | id | PRIMARY | 主键 |
| idx_name | name | NORMAL | 名称查询 |
| idx_status | status | NORMAL | 状态筛选 |

---

## 测试分析（可选）

### [功能模块1] 测试用例

| 类型 | 测试场景 | 测试步骤 | 检查点 |
|------|---------|---------|--------|
| 正常场景 | [场景描述] | 1. [步骤1]<br>2. [步骤2]<br>3. [步骤3] | [预期结果] |
| 异常场景 | [场景描述] | 1. [步骤1]<br>2. [步骤2] | [预期结果] |