+++
title = "实现优雅的 Golang 项目结构"
description = ""
date = "2021-02-19T18:10:22+08:00"
updated = "2021-02-19T18:10:22+08:00"
draft = false
[taxonomies]
tags = ["Golang"]
+++

随着工作经验的积累, 越来越感到仅仅写出 `Elegant Code` 是不够的, 特别是当多人合作开发的时候, 遇到种种结构混乱, 代码冗余, 模块耦合, 依赖耦合的问题. 这个时候, 就需要更高的要求, `Elegant Project`, 自顶向下的改善项目的结构.

仅仅实现业务逻辑是很容易的, 而大量的时间会用在测试, Debug 上. 一个良好定义的项目结构能够极大地提升开发效率, 降低维护成本. 下面介绍一套经过实践检验的 Go 项目结构规范.

## 项目结构规范

### 服务调用 (Service Invocation)
项目的核心调用链遵循一个清晰的层次结构: 
1. **接口定义**:  服务接口在 `./domain/<service_name>/domain.go` 文件中定义. 这是服务能力的抽象.
2. **服务实现**:  服务的具体实现在对应的 `./app/<service_name>` 目录中完成.
3. **Web Handler**:  Web Handler 实现在 `./internal/web` 目录中.
4. **调用流程**:  Web Handler 调用 `./domain/<service_name>` 中定义的服务接口, 而非直接依赖具体实现.
5. **复杂流程处理**:  对于需要调用多个服务方法的复杂业务流程, 应该在一个专用的, 以 `flow` 为后缀的服务中进行编排, 而不是直接在 Web Handler 中组合.

### 服务包命名 (Service Package Name)
- **避免服务包与它操作的实体同名**:  例如, 操作 `User` 实体的服务不应该命名为 `user`.
- **使用后缀来明确包的职责**:  推荐使用 `mgr`, `hdl`, `uc`, `svc`, `flow` 等后缀来命名服务包, 以清晰地表达其作为服务层的角色.

### 服务方法签名 (Service Method Signature)
为了保持一致性和可预测性, 所有服务方法都应遵循以下签名格式: 
```go
// 标准服务方法签名
Hello(ctx context.Context, req *HelloRequest) (resp *HelloResponse, err error)
```
- **请求/响应类型**:  请求和响应的结构体遵循 `<MethodName>Request` 和 `<MethodName>Response` 的命名模式.
- **上下文参数**:  第一个参数必须是 `context.Context`, 用于传递请求范围的数据, 如 trace ID, 用户身份等.
- **错误返回**:  最后一个返回值必须是 `error`, 用于错误处理.
- **命名返回值**:  推荐使用命名返回值, 以增强代码的可读性.
- **批量获取**:  对于通过 ID 批量获取数据的方法, 方法名应为 `Fetch<Objects>Dict`, 且响应中应包含一个名为 `<Objects>Dict` 的 `map[<IDType>]<Object>` 字段, 以方便调用方通过 ID 查找对象.

### 服务实现结构 (Service Implementation Structure)
一个独立服务的内部结构组织如下: 
- **服务接口**:  `Service` 接口位于 `./domain/<service_name>/domain.go`.
- **内部核心定义**:  如有需要, 内部使用的数据结构和 Repository 接口定义在 `./app/<service_name>/internal/core/core.go`, 以 `core` 为包名.
- **Repository 实现**:  Repository 的具体实现放在 `./app/<service_name>/internal/repo` 目录中.
- **服务初始化**:  服务的结构体和构造逻辑, 以及传入的依赖定义在 `./app/<service_name>/service.go`.
- **服务构造函数**:  服务构造函数应命名为 `NewService(inj *NewServiceInj, opts ...ServiceOption) (srv *Service, err error)`.
- **业务逻辑**:  核心业务逻辑实现在 `./app/<service_name>` 目录下的独立文件中, 与服务构造函数分离.
- **Repository 构造函数**:  Repository 的构造函数和通用的事务逻辑 (如 `(r *Repo) WithTx(...)`) 应放在 `./app/<service_name>/internal/repo/repo.go` 中.
- **Repository 操作**:  Repository 的具体数据库操作在 `./app/<service_name>/internal/repo` 目录下的独立文件中实现, 与其构造函数分离, 以 `repo` 为包名.
- **文件命名**:  业务逻辑文件应根据其主要处理的实体来命名, 清晰地反映其职责.
- **数据库事务边界**:  数据库事务边界应定义在服务层, 而不是在 Repository 层.

### 服务 Mock (Service Mock)
- **Mock 初始化**:  总是使用辅助函数 `initMockService(t *testing.T) (srv *service.Service, mockInj *mockInjection)` 来初始化被测试的服务及其 mock 依赖.
- **Mock 生成**:  使用 `Mockery` 基于接口生成 mock 对象.
- **类型安全**:  测试辅助函数应使用**类型安全**的方法调用来设置 mock 行为, 而不是依赖于基于字符串的通用方法, 如 `mock.On("MethodName")`.

### 脚本 (Scripts)
- **通用脚本**:  常用的脚本应放置在 `./script` 目录中.
- **SQL 文件**:  SQL DML 和 DDL 文件应放置在 `./script/sql` 目录中.

### 可选实现: 使用 Ent ORM

本结构规范不强制绑定任何 ORM 框架, 但如果选择使用 [Ent](https://entgo.io/), 可以遵循以下约定来更好地集成.

- **Schema 定义**:  `Ent` 的 Schema 定义在 `./internal/schema` 目录中, 每个数据表对应一个文件, 如 `./internal/schema/user.go`.
- **生成代码**:  `Ent` 生成的代码位于 `./internal/ent` 目录.
- **实体命名**:  在 Repository 层, `Ent` ORM 生成的对象在命名时应使用 `Entity` (或 `Entities` 用于集合) 后缀, 以明确区分它们与领域中的 `core` 包中的实体, 并且 `Ent` 生成的对象不能泄露出 `repo` 包.
- **代码生成脚本**:  生成 Ent ORM 代码的脚本应位于 `./script/ent.sh`.

## 服务调用流程图
下图展示了从接收客户端请求到与数据库交互的完整分层调用链, 它强调了接口与实现分离 (依赖倒置原则), 以及各层在项目中的位置. 这样保证了每一个层级的职责清晰, 耦合度低, 方便维护和扩展, 也方便进行单元测试和排查问题.
```text
[ API Request ]
      |
      v
+--------------------------------+
|      Web Handler               |
|     (./internal/web)           |
+--------------------------------+
      |
      |  (依赖接口)
      v
+--------------------------------+ <----+
|      Domain Interface          |      |
|   (./domain/<service_name>)    |      | (服务之间通过接口调用)
+--------------------------------+      |
      |                                  
      |  (由 App 层实现)
      v
+--------------------------------+      |
|      App Implementation        | -----+
|    (./app/<service_name>)      |
+--------------------------------+
      |
      |  (依赖接口)
      v
+--------------------------------+
|      Repository Interface      |
|   (./app/.../internal/core)    |
+--------------------------------+
      |
      |  (由 Repo 层实现)
      v
+--------------------------------+
|      Repository Implementation |
|   (./app/.../internal/repo)    |
+--------------------------------+
      |
      v
[ Database / ORM ]
```

## 示例项目目录结构
这是一个遵循上述规范的示例项目结构. 它以一个用户注册场景为例, 其中 `registrationflow` 是一个复杂流程, 负责编排 `authuc` (认证用例) 和 `notiuc` (通知用例).
```text
/
├── app/
│   ├── registrationflow/           # “用户注册”流程的实现
│   │   ├── service.go              # 定义 Service, NewService
│   │   └── registration_logic.go   # 流程编排逻辑
│   ├── authuc/                     # “认证”用例的实现
│   │   ├── internal/
│   │   │   ├── core/
│   │   │   │   ├── core.go         # 定义 IAuthRepo 使用的数据结构
│   │   │   │   └── repo_mock.go    # IAuthRepo 的 mock 实现
│   │   │   └── repo/
│   │   │       ├── repo.go         # 定义 Repo, NewRepo, WithTx
│   │   │       └── auth_repo.go    # 实现 IAuthRepo
│   │   ├── service.go              # 定义 Service, NewService
│   │   ├── auth_logic.go           # 认证业务逻辑
│   │   └── auth_logic_test.go      # 认证业务逻辑的单元测试
│   └── notiuc/                     # “通知”用例的实现
│       ├── internal/               # (可选, 如需封装 client 或定义内部结构)
│       │   └── ...
│       ├── service.go              # 定义 Service, NewService
│       └── noti_logic.go           # 通知业务逻辑 (如发邮件, 短信)
├── cmd/
│   └── server/
│       └── main.go
├── domain/
│   ├── registrationflow/
│   │   ├── domain.go               # 定义 RegistrationFlow 接口
│   │   └── service_mock.go         # RegistrationFlow 的 mock 实现
│   ├── authuc/
│   │   ├── domain.go               # 定义 AuthUseCase 接口
│   │   └── service_mock.go         # AuthUseCase 的 mock 实现
│   └── notiuc/
│       ├── domain.go               # 定义 NotiUseCase 接口
│       └── service_mock.go         # NotiUseCase 的 mock 实现
├── internal/
│   └── web/
│       ├── registration_handler.go # 处理注册流程, 调用 domain/registrationflow
│       ├── auth_handler.go         # 处理认证请求, 调用 domain/authuc
│       └── auth_handler_test.go    # 认证请求处理的单元测试
...
```
上述目录结构展示了一个典型的分层应用: 
* **`domain`**:  定义了三个核心服务接口: `RegistrationFlow`、`AuthUseCase` 和 `NotiUseCase`. 这是业务能力的抽象层.
* **`app`**:  包含了上述接口的具体实现. `registrationflow` 是一个流程服务 (Flow), 它的实现会依赖 `authuc` 和 `notiuc` 的接口, 从而编排和调用这两个用例服务.
* **`internal/web`**:  Web Handler 层, 负责暴露服务给外部. 不同的 Handler 文件处理不同的业务, 例如 `registration_handler.go` 调用复杂的流程服务, 而 `auth_handler.go`可以直接调用简单的用例服务.
* **包命名**:  注意所有包名都使用简短, 全小写的形式, 并且不使用下划线, 这是 Go 语言推荐的风格.
* **Mock 与测试**:
    * **Mock 文件**: `*_mock.go` 是接口的 mock 实现, 用于单元测试.
    * **测试文件**: `*_test.go` 是对应的单元测试文件.
    * **依赖关系**: 基于依赖倒置原则, 测试文件会使用 mock 来隔离被测试的层级.
        * `app/authuc/auth_logic_test.go` (业务逻辑测试) 会使用 `app/authuc/internal/core/repo_mock.go` 来模拟数据访问行为.
        * `internal/web/auth_handler_test.go` (Handler 测试) 会使用 `domain/authuc/service_mock.go` 来模拟业务服务行为.

## 总结
本规范的核心价值在于提供一个清晰一致的项目结构, 从而提升项目的**可测试性**和**可维护性**. 通过明确的分层和接口定义, 实现了以下关键的架构原则: 
* **Monorepo 下的微服务化**: 在单一代码库(Monorepo)中, 通过模块化的服务(`app`)和统一的接口(`domain`), 实现了类似微服务的开发体验. 每个服务都可以独立开发, 测试和部署, 但又能在同一个项目中轻松共享代码和配置.
* **支持多人协作**: 清晰的边界和所有权使得多个开发团队可以并行工作, 而不会相互干扰. 每个团队可以专注于自己的服务, 并通过定义好的接口进行协作.
* **高可测试性**: 依赖倒置原则的贯彻, 以及接口 mock 的普遍使用, 使得每个模块都可以进行独立的单元测试, 保证了代码质量和迭代速度.

## 参考链接
* [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
* [Mattermost](https://github.com/mattermost/mattermost)
