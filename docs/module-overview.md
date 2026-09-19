# mall-swarm 模块说明

> `mall-swarm` 是一套**微服务架构的电商商城系统**，基于 Spring Cloud 2025 + Spring Cloud Alibaba + Spring Boot 3.5 + Sa-Token + MyBatis 构建。
> 本文档用目录树（Markdown tree）的方式，说明各个模块分别负责什么。
>
> 生成时间：基于当前工作区代码（Java 17 / Spring Boot 3.5.14 / Spring Cloud 2025.0.2 / Spring Cloud Alibaba 2025.0.0.0）。

---

## 一、模块总览（Markdown Tree）

```text
mall-swarm/                                 微服务商城系统（Maven 聚合父工程，packaging=pom）
│
├── mall-common/                            【基础库·无端口】工具类与通用代码：统一响应、分页、异常、Redis 封装、日志切面
├── mall-mbg/                               【基础库·无端口】MyBatis Generator 生成的数据库操作代码：76 个 Mapper + 实体/Example
│
├── mall-gateway/                           【微服务·8201】API 网关：路由转发 + 统一登录/权限校验 + 跨域 + Knife4j 文档聚合
├── mall-auth/                              【微服务·8401】统一认证中心：后台管理员 / 前台会员登录发 Token
├── mall-monitor/                           【微服务·8101】监控中心：Spring Boot Admin，可视化查看各服务健康/指标/日志
│
├── mall-admin/                             【业务服务·8080】后台管理系统服务：商品、订单、营销、用户权限、文件上传
├── mall-portal/                            【业务服务·8085】移动端商城服务：首页、购物车、下单、支付、会员中心
├── mall-search/                            【业务服务·8081】商品搜索服务：基于 Elasticsearch 的搜索与推荐
└── mall-demo/                              【示例服务·8082】远程调用（OpenFeign）测试与各种技术整合示例
│
├── config/                                 配置中心（Nacos）存储的配置文件，按服务/环境分目录
│   ├── admin/     mall-admin-dev.yaml      mall-admin-prod.yaml
│   ├── gateway/   mall-gateway-dev.yaml    mall-gateway-prod.yaml
│   ├── portal/    mall-portal-dev.yaml     mall-portal-prod.yaml
│   ├── search/    mall-search-dev.yaml     mall-search-prod.yaml
│   └── demo/      mall-demo-dev.yaml       mall-demo-prod.yaml
│
├── document/                               项目配套文档与资源（非代码）
│   ├── docker/    docker-compose-env.yml（环境）、docker-compose-app.yml（应用）、nginx.conf
│   ├── k8s/       各服务的 Deployment / Service YAML
│   ├── elk/       logstash.conf、logback-spring.xml（日志收集）
│   ├── sh/        各服务的启动/停止脚本
│   ├── sql/       mall.sql（建库建表 + 初始化数据，共 76 张表）
│   ├── mind/      各业务模块思维导图（pms/oms/sms/ums/cms/app/home）
│   ├── pdm/       数据库设计模型（PowerDesigner）
│   ├── pos/       架构图源文件（业务架构/系统架构/微服务架构/开发进度）
│   ├── reference/ 部署与开发文档（deploy_windows.md、dev_flow.md、function.md）
│   └── resource/  文档用截图
│
├── docs/                                   本项目说明文档（即本文件所在目录）
├── pom.xml                                 父 POM：统一版本管理、依赖管理、Docker 镜像构建插件
├── README.md                               项目主说明
└── LICENSE                                 Apache License 2.0
```

---

## 二、各模块职责详解

### 1. `mall-common` — 通用基础模块

所有服务的公共依赖，不单独启动，被其它模块以 Maven 依赖方式引入。

```text
mall-common/src/main/
├── java/com/macro/mall/common/
│   ├── annotation/CacheException.java      缓存异常注解
│   ├── api/                                统一 API 返回约定
│   │   ├── CommonResult.java               统一响应结果封装
│   │   ├── CommonPage.java                 统一分页结果封装
│   │   ├── IErrorCode.java                 错误码接口
│   │   └── ResultCode.java                 通用错误码枚举
│   ├── config/BaseRedisConfig.java         Redis 通用配置（RedisTemplate 序列化等）
│   ├── constant/AuthConstant.java          认证授权常量（clientId、请求头、Redis key 等）
│   ├── domain/WebLog.java                  操作日志实体
│   ├── dto/UserDto.java                    登录用户信息 DTO
│   ├── exception/                          全局异常处理
│   │   ├── ApiException.java               业务异常
│   │   ├── Asserts.java                    断言工具
│   │   └── GlobalExceptionHandler.java     全局异常处理器
│   ├── log/WebLogAspect.java               AOP 操作日志切面
│   └── service/                            Redis 操作封装
│       ├── RedisService.java
│       └── impl/RedisServiceImpl.java
└── resources/logback-spring.xml            统一日志配置（含 Logstash 输出）
```

**一句话**：把「所有服务都要用的东西」集中在这里，避免重复代码。

---

### 2. `mall-mbg` — 数据层代码生成模块

由 `MyBatis Generator` 依据数据库表自动生成，不写业务逻辑，供业务服务直接引用。

```text
mall-mbg/
├── src/main/java/com/macro/mall/
│   ├── mapper/       76 个 Mapper 接口（与数据库表一一对应）
│   ├── model/        实体类 + Example 条件查询类（共 152 个文件）
│   ├── CommentGenerator.java   自定义注释生成器
│   └── Generator.java          MBG 启动入口（main 方法）
└── src/main/resources/
    ├── generatorConfig.xml     MBG 配置（表名、包名、生成策略）
    └── generator.properties    数据库连接与目标包配置
```

**表名前缀即业务域划分**，这是 mall 项目的命名约定：

| 前缀  | 含义               | 典型表                                             |
| ----- | ------------------ | -------------------------------------------------- |
| `pms` | 商品管理 Product   | `pms_product`、`pms_brand`、`pms_sku_stock`        |
| `oms` | 订单管理 Order     | `oms_order`、`oms_cart_item`、`oms_order_item`     |
| `sms` | 营销管理 Sale      | `sms_coupon`、`sms_flash_promotion`、`sms_home_advertise` |
| `ums` | 用户管理 User      | `ums_admin`、`ums_member`、`ums_role`、`ums_resource` |
| `cms` | 内容管理 Content   | `cms_subject`、`cms_help`、`cms_topic`             |

**一句话**：数据库的「Java 镜像」，改表后重新跑一次生成器即可。

---

### 3. `mall-gateway` — API 网关

系统的**唯一入口**，所有外部请求先经过它。

```text
mall-gateway/src/main/java/com/macro/mall/
├── component/StpInterfaceImpl.java    Sa-Token 权限数据源（读取用户拥有的权限码）
├── config/
│   ├── SaTokenConfig.java             全局过滤器：登录认证 + 接口权限校验 + 异常统一返回 JSON
│   ├── IgnoreUrlsConfig.java          白名单路径配置（secure.ignore.urls）
│   ├── GlobalCorsConfig.java          跨域配置
│   └── RedisConfig.java               Redis 配置（读取权限规则缓存）
├── util/StpMemberUtil.java            前台会员的 Sa-Token 登录工具
└── MallGatewayApplication.java        启动类
```

核心职责：

1. **路由转发**（`lb://服务名`，配合 Nacos 服务发现）：

   | 路由前缀           | 目标服务       |
   | ------------------ | -------------- |
   | `/mall-auth/**`    | `mall-auth`    |
   | `/mall-admin/**`   | `mall-admin`   |
   | `/mall-portal/**`  | `mall-portal`  |
   | `/mall-search/**`  | `mall-search`  |
   | `/mall-demo/**`    | `mall-demo`    |

2. **认证与鉴权**（Sa-Token 全局过滤器 `SaReactorFilter`）：
   - `/mall-portal/**` → 校验前台**会员**登录（`StpMemberUtil.checkLogin()`）
   - `/mall-admin/**` → 校验后台**管理员**登录（`StpUtil.checkLogin()`）
   - 从 Redis 的 `auth:pathResourceMap`（路径 → 所需资源权限）中匹配当前请求所需权限，再做 `checkPermissionOr` 校验
   - 认证失败统一返回 `CommonResult` JSON（401 / 403）
3. **文档聚合**：Knife4j 网关模式，通过服务发现自动聚合各服务的 OpenAPI 文档，访问 `/doc.html`。
4. **白名单放行**：登录、注册、首页、商品、搜索、支付回调、静态资源、Actuator 等。

---

### 4. `mall-auth` — 统一认证中心

集中处理**两个端**的登录，签发 Token。

```text
mall-auth/src/main/java/com/macro/mall/auth/
├── controller/AuthController.java     POST /auth/login（参数 clientId + username + password）
├── domain/UmsAdminLoginParam.java     登录参数
├── service/
│   ├── UmsAdminService.java           @FeignClient("mall-admin") → 调用后台服务校验管理员
│   └── UmsMemberService.java          @FeignClient("mall-portal") → 调用前台服务校验会员
└── MallAuthApplication.java           @EnableFeignClients + @EnableDiscoveryClient
```

- 通过 `clientId` 区分端：`admin-app`（后台）、`portal-app`（前台），见 `AuthConstant`。
- 本身**不查数据库**，而是通过 OpenFeign 委托给 `mall-admin` / `mall-portal` 校验账密，再由 Sa-Token 生成 Token。
- 当前实现已从早期的 Spring Security OAuth2 迁移为 **Sa-Token**（token 名 `Authorization`，前缀 `Bearer`，支持 JWT）。

---

### 5. `mall-monitor` — 监控中心

基于 **Spring Boot Admin** 的服务监控台，访问 `http://localhost:8101`（账号 `macro` / `123456`）。

```text
mall-monitor/src/main/java/com/macro/mall/
├── config/SecuritySecureConfig.java   Spring Boot Admin 的安全配置（登录页、CSRF、内存用户）
├── filter/CustomCsrfFilter.java       自定义 CSRF 过滤器
└── MallMonitorApplication.java        @EnableDiscoveryClient（从 Nacos 发现被监控服务）
```

- 各业务服务通过 Actuator 暴露端点（`management.endpoints.web.exposure.include: '*'`）被它采集。
- 可查看：服务上下线、健康状态、JVM/内存/线程指标、环境变量、日志级别等。
- 不参与业务，属于运维观测类服务。

---

### 6. `mall-admin` — 后台管理系统服务

面向**运营/管理员**的业务服务，功能最全，端口 `8080`。

```text
mall-admin/src/main/
├── java/com/macro/mall/
│   ├── component/PathResourceRulesHolder.java   启动时把「接口路径→所需资源权限」写入 Redis
│   ├── config/                                  MyBatis / Redis / OSS / SpringDoc / Sa-Token 配置
│   ├── controller/  31 个控制器                 按 pms / oms / sms / ums / cms 分组
│   │   ├── 商品：PmsProductController、PmsBrandController、PmsProductCategoryController、
│   │   │        PmsProductAttributeController、PmsSkuStockController …
│   │   ├── 订单：OmsOrderController、OmsOrderReturnApplyController、OmsOrderReturnReasonController、
│   │   │        OmsOrderSettingController、OmsCompanyAddressController
│   │   ├── 营销：SmsCouponController、SmsFlashPromotionController、SmsHomeAdvertiseController、
│   │   │        SmsHomeBrandController、SmsHomeNewProductController、SmsHomeRecommend*Controller
│   │   ├── 权限：UmsAdminController、UmsRoleController、UmsMenuController、UmsResourceController、
│   │   │        UmsResourceCategoryController、UmsMemberLevelController
│   │   ├── 内容：CmsSubjectController、CmsPrefrenceAreaController
│   │   └── 文件：OssController（阿里云 OSS）、MinioController（MinIO 对象存储）
│   ├── service/ 31 个接口 + 31 个实现            业务逻辑层
│   │   └── impl/UmsAdminCacheServiceImpl.java   管理员信息缓存
│   ├── dao/     22 个自定义 DAO                 多表关联查询（配套 resources/dao/*.xml）
│   ├── dto/     29 个传输对象
│   └── validator/FlagValidator.java             自定义校验注解
└── resources/
    ├── dao/*.xml        自定义 SQL 映射（22 个）
    └── application*.yml 本地配置（数据库、Redis、OSS、MinIO、Sa-Token）
```

亮点：
- 权限模型 = 用户 → 角色 → 菜单/资源，登录后经网关做接口级鉴权。
- 支持两种对象存储：**阿里云 OSS** 与 **MinIO**。
- 该服务也是管理员数据的来源：`mall-auth` 通过 OpenFeign 调用它校验管理员账密。

---

### 7. `mall-portal` — 移动端商城服务

面向**前台会员/App/小程序**的业务服务，端口 `8085`。

```text
mall-portal/src/main/
├── java/com/macro/mall/portal/
│   ├── controller/  13 个控制器
│   │   ├── 首页与商品：HomeController、PmsPortalProductController、PortalBrandController
│   │   ├── 购物车：OmsCartItemController
│   │   ├── 订单：OmsPortalOrderController、OmsPortalOrderReturnApplyController
│   │   ├── 支付：AlipayController（支付宝）
│   │   └── 会员中心：UmsMemberController、UmsMemberCouponController、
│   │                  UmsMemberReceiveAddressController、MemberAttentionController（品牌关注）、
│   │                  MemberCollectionController（商品收藏）、MemberReadHistoryController（浏览记录）
│   ├── service/ 15 个接口 + impl/ 15 个实现
│   │   ├── OmsCartItemService            购物车（含促销计算）
│   │   ├── OmsPortalOrderService         下单、支付回调、订单取消
│   │   ├── OmsPromotionService           促销/优惠券计算
│   │   ├── AlipayService                 支付宝对接
│   │   └── UmsMemberCacheService         会员信息缓存
│   ├── dao/          自定义 DAO（首页、订单、商品、优惠券）
│   ├── domain/       18 个领域对象（ConfirmOrderResult、CartPromotionItem、HomeContentResult …）
│   ├── repository/   MongoDB Repository（品牌关注、商品收藏、浏览记录）
│   ├── component/    订单超时处理
│   │   ├── CancelOrderSender.java        RabbitMQ 发送延迟/取消消息
│   │   ├── CancelOrderReceiver.java      消费消息执行取消
│   │   └── OrderTimeOutCancelTask.java   定时任务兜底（Spring Task）
│   ├── config/       RabbitMq / Redis / MongoDB / Alipay / Jackson / SpringTask 配置
│   └── util/         DateUtil、StpMemberUtil（会员登录态工具）
└── resources/
    ├── dao/*.xml
    └── application*.yml
```

技术要点：
- **MySQL** 存交易数据，**MongoDB** 存会员行为数据（浏览记录/收藏/关注），**Redis** 存验证码与会员缓存。
- **RabbitMQ + 定时任务**双保险处理「未支付订单超时自动取消」。
- **支付宝沙箱**支付，含回调处理。

---

### 8. `mall-search` — 商品搜索服务

基于 **Elasticsearch** 的商品检索服务，端口 `8081`。

```text
mall-search/src/main/
├── java/com/macro/mall/search/
│   ├── controller/EsProductController.java     搜索相关 REST 接口
│   ├── service/EsProductService.java           搜索业务接口
│   │   └── impl/EsProductServiceImpl.java      导入商品、综合搜索、简单搜索、相关推荐、聚合筛选
│   ├── repository/EsProductRepository.java     Spring Data Elasticsearch 仓库
│   ├── domain/                                 EsProduct（商品文档）、EsProductAttributeValue、
│   │                                           EsProductRelatedInfo（品牌/分类/属性聚合结果）
│   ├── dao/EsProductDao.java                   从 MySQL 读取商品数据用于同步索引（+ dao/EsProductDao.xml）
│   └── config/                                 MyBatis / SpringDoc 配置
└── resources/application*.yml
```

**一句话**：把 MySQL 的商品数据导入 ES 索引，对外提供带分词、筛选、聚合、相关推荐的高性能搜索。

---

### 9. `mall-demo` — 远程调用示例服务

学习/验证用的示例服务，端口 `8082`，**不承载真实业务**。

```text
mall-demo/src/main/java/com/macro/mall/
├── MallDemoApplication.java          @EnableFeignClients + @EnableDiscoveryClient
├── controller/
│   ├── DemoController.java           本地接口示例（如参数校验 FlagValidator）
│   ├── FeignAdminController.java     演示调用 mall-admin
│   ├── FeignPortalController.java    演示调用 mall-portal
│   └── FeignSearchController.java    演示调用 mall-search
├── service/                          FeignAdminService / FeignPortalService / FeignSearchService
├── component/FeignRequestInterceptor.java   Feign 请求拦截器（透传 Token 等请求头）
├── config/                           Feign / MyBatis / SpringDoc 配置
├── dto/                              示例 DTO
└── validator/                        自定义校验器示例
```

**一句话**：演示微服务之间如何用 OpenFeign 互相调用，以及如何透传认证信息。

---

## 三、模块间关系

#### 请求链路

```text
外部请求
   │
   ▼
┌──────────────────────────────┐
│      mall-gateway :8201      │   路由转发 + Sa-Token 登录认证 / 接口鉴权
└──────────────┬───────────────┘
               │ lb://服务名（Nacos 服务发现）
   ┌───────────┼───────────┬───────────────┬──────────────┐
   ▼           ▼           ▼               ▼              ▼
 mall-auth  mall-admin  mall-portal   mall-search    mall-demo
  :8401      :8080       :8085          :8081          :8082
   │
   │ OpenFeign 委托校验账密（mall-auth 自身不连数据库）
   ├──────────────────► mall-admin    （校验管理员）
   └──────────────────► mall-portal   （校验会员）

 mall-demo ──OpenFeign──► mall-admin / mall-portal / mall-search
             （仅用于演示远程调用，不属于业务链路）

 基础库：mall-common（公共代码）· mall-mbg（数据层代码）── 所有模块共同依赖
 支撑组件：Nacos 8848 · Redis · MySQL · MongoDB · RabbitMQ · Elasticsearch · MinIO/OSS
 运维观测：mall-monitor :8101（Spring Boot Admin，经 Nacos 发现各服务）
```

#### 服务间调用（OpenFeign）实际分布

| 调用方      | 被调用方                                            | 用途               |
| ----------- | --------------------------------------------------- | ------------------ |
| `mall-auth` | `mall-admin`、`mall-portal`                         | 登录时校验账密     |
| `mall-demo` | `mall-admin`、`mall-portal`、`mall-search`          | 演示远程调用       |

> 注：`mall-admin` 与 `mall-portal` 的启动类上声明了 `@EnableFeignClients`，但当前代码中未定义任何 `@FeignClient`，属于预留能力。

**依赖层次**：

1. `pom.xml`（父工程）统一管理版本。
2. `mall-common`、`mall-mbg` 是被依赖的**基础库**，无启动类。
3. `mall-gateway`、`mall-auth`、`mall-monitor`、`mall-demo` 是**基础设施/示例**服务。
4. `mall-admin`、`mall-portal`、`mall-search` 是**三大业务服务**。

---

## 四、配置与运行方式

- **配置中心**：Nacos（`localhost:8848`）。`config/` 目录下的 `mall-<服务>-<环境>.yaml` 需要导入 Nacos，各服务通过 `spring.config.import: nacos:mall-xxx-dev.yaml?refreshEnabled=true` 拉取。
- **环境切换**：`spring.profiles.active: dev | prod`（每个服务都有 `application-dev.yml` / `application-prod.yml`）。
- **端口约定**：

  | 服务         | 端口 | 说明               |
  | ------------ | ---- | ------------------ |
  | mall-gateway | 8201 | 统一入口、文档聚合 |
  | mall-auth    | 8401 | 认证中心           |
  | mall-monitor | 8101 | 监控中心           |
  | mall-admin   | 8080 | 后台管理           |
  | mall-search  | 8081 | 商品搜索           |
  | mall-demo    | 8082 | 示例               |
  | mall-portal  | 8085 | 前台商城           |
  | Nacos        | 8848 | 注册与配置中心     |

- **部署**：`document/docker/`（Docker Compose）、`document/k8s/`（Kubernetes）、`document/sh/`（启动脚本）；父 POM 内置 `docker-maven-plugin`，`mvn package` 时可自动构建镜像。

---

## 五、附：用 eza 快速遍历项目

```powershell
# 只看到模块级（推荐先跑这个）
eza --tree --level=1 --group-directories-first --ignore-glob='.git|.idea|target'

# 看某模块的源码结构
eza --tree --level=4 --group-directories-first mall-admin/src/main/java

# 带文件大小/日期，按修改时间排序
eza --tree --level=2 --long --time-style=long-iso --sort=modified

# 只看目录，忽略构建产物
eza --tree --level=3 --only-dirs --ignore-glob='target|node_modules'

# 区分 git 状态（未跟踪/已修改）
eza --tree --level=2 --git --git-ignore
```

---

## 相关文档

- 项目主说明：[`README.md`](../README.md)
- 部署文档：`document/reference/deploy_windows.md`
- 功能清单：`document/reference/function.md`
- 数据库脚本：`document/sql/mall.sql`
