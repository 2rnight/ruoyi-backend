# 阶段 3：核心模块源码深潜

> **分析范围：** 选取 `ruoyi-framework` 中的 `SysLoginService.login()`，沿着身份验证、密码校验、权限装载、Redis 会话缓存和 JWT 返回继续追踪。重点是源码当前的执行顺序和边界，不代表已经运行集成测试；本阶段只新增本分析文档，不修改业务代码。

## 1. 模块定位、输入与输出

登录的 HTTP 入口是 `ruoyi-admin/src/main/java/com/ruoyi/web/controller/system/SysLoginController.java` 中的 `login()`（第 56–65 行），它接收 `LoginBody` 并调用 `SysLoginService.login(username, password, code, uuid)`。`LoginBody` 定义了用户名、密码、验证码和验证码 UUID（`ruoyi-common/src/main/java/com/ruoyi/common/core/domain/model/LoginBody.java:8-28`）。

`SysLoginService.login()` 的主要输出是签名后的 JWT 字符串；Controller 将其放进成功响应的 `token` 字段。链路还会产生几类副作用：

- 从 Redis 读取并删除验证码；密码错误时读写失败计数。
- 成功登录后更新 MySQL 中用户的最后登录 IP 和时间。
- 异步安排登录日志写入。
- 在 Redis 中保存 `LoginUser` 及其有效期。

核心入口见 `ruoyi-framework/src/main/java/com/ruoyi/framework/web/service/SysLoginService.java:63-100`。

## 2. 按执行顺序拆解

### 2.1 验证码：先读、再删、再比较

`login()` 首先调用 `validateCaptcha()`（`SysLoginService.java:65-66`）。该方法先由 `configService.selectCaptchaEnabled()` 判断验证码是否开启；开启时，以 `CacheConstants.CAPTCHA_CODE_KEY + uuid` 作为 Redis key 读取验证码。key 不存在时安排失败日志并抛出 `CaptchaExpireException`。key 存在时，代码先删除 key，再以忽略大小写的方式比较验证码；不匹配则安排失败日志并抛 `CaptchaException`（`SysLoginService.java:110-129`）。

因此，此实现把验证码作为单次凭据使用：**无论输入是否正确，只要读到了验证码，就会先删除它**。`uuid` 为空时会用空字符串参与拼 key，但方法没有在这里验证 `code` 非空。

### 2.2 登录前置检查：输入范围与 IP 黑名单

随后 `loginPreCheck()` 检查用户名、密码非空，密码长度与用户名长度是否在 `UserConstants` 规定范围内，并读取配置项 `sys.login.blackIPList` 检查当前 IP。违反任一条件时会安排失败日志并抛相应业务异常（`SysLoginService.java:136-165`；长度常量见 `ruoyi-common/src/main/java/com/ruoyi/common/constant/UserConstants.java:71-80`）。

当前默认配置为密码最多错误 5 次、锁定 10 分钟（`ruoyi-admin/src/main/resources/application.yml:40-46`）。

### 2.3 认证上下文与 Spring Security 用户加载

进入认证阶段后，`SysLoginService` 创建 `UsernamePasswordAuthenticationToken`，先放入 `AuthenticationContextHolder`，再调用 `AuthenticationManager.authenticate()`（`SysLoginService.java:69-76`）。源码注释指出，认证过程会调用 `UserDetailsServiceImpl.loadUserByUsername()`。

`UserDetailsServiceImpl` 依次执行以下处理（`ruoyi-framework/src/main/java/com/ruoyi/framework/web/service/UserDetailsServiceImpl.java:38-65`）：

1. 调用 `ISysUserService.selectUserByUserName()` 查询用户。实现委托给 `SysUserMapper`（`ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysUserServiceImpl.java:114-118`）。
2. 用户不存在时抛业务异常；检查删除标志和停用状态。
3. 调用 `SysPasswordService.validate(user)` 校验密码和错误次数。
4. 调用 `createLoginUser()`，将用户 ID、部门 ID、`SysUser` 和菜单权限集合装入 `LoginUser` 并返回。

用户名查询的 MyBatis SQL 使用 `selectUserVo`，关联 `sys_dept`、`sys_user_role` 和 `sys_role`，并且只查 `del_flag = '0'` 的用户（`ruoyi-system/src/main/resources/mapper/system/SysUserMapper.xml:50-58, 124-127`）。因此在该查询路径中，已软删除用户通常会先被过滤为“查无此人”；`UserDetailsServiceImpl` 中的删除状态判断仍然存在。

### 2.4 密码错误计数与权限集合

`SysPasswordService.validate()` 从 `AuthenticationContextHolder` 读取当前认证请求的用户名和原始密码，再从 Redis 读取该用户名对应的错误计数（key 前缀为 `PWD_ERR_CNT_KEY`）。计数达到配置上限时抛出锁定异常；密码不匹配时计数加一，并按 `lockTime` 分钟写回 Redis；成功时删除错误计数（`ruoyi-framework/src/main/java/com/ruoyi/framework/web/service/SysPasswordService.java:39-85`）。密码比对调用 `SecurityUtils.matchesPassword()`，该方法使用 `BCryptPasswordEncoder`（`ruoyi-common/src/main/java/com/ruoyi/common/utils/SecurityUtils.java:93-114`）。

权限集合由 `SysPermissionService.getMenuPermission()` 生成：管理员直接获得全权限标识；非管理员遍历正常角色，查询其菜单权限并合并到集合中（`ruoyi-framework/src/main/java/com/ruoyi/framework/web/service/SysPermissionService.java:58-88`）。`LoginUser` 实现 Spring Security `UserDetails`，并在 `getAuthorities()` 中将权限字符串转换成 `SimpleGrantedAuthority`（`ruoyi-common/src/main/java/com/ruoyi/common/core/domain/model/LoginUser.java:19-94, 265-277`）。

### 2.5 认证异常、成功副作用与 token 生成

认证调用位于 `try/catch/finally` 内：

- `BadCredentialsException` 会安排密码错误日志，并转成 `UserPasswordNotMatchException`。
- 其他认证异常也会安排失败日志，再包装成 `ServiceException`。
- 无论成功或失败，`finally` 都会调用 `AuthenticationContextHolder.clearContext()`。

认证成功后，代码按顺序安排成功登录日志、取出 `LoginUser`、更新数据库中的登录 IP/时间，最后调用 `TokenService.createToken()`（`SysLoginService.java:78-100, 172-175`）。登录信息更新由 `SysUserServiceImpl.updateLoginInfo()` 委托 Mapper 执行，对应 SQL 更新 `sys_user.login_ip` 和 `login_date`（`ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysUserServiceImpl.java:358-369`；`ruoyi-system/src/main/resources/mapper/system/SysUserMapper.xml:208-210`）。

`TokenService.createToken()` 的步骤是：生成 UUID 并写入 `LoginUser.token`；补充 IP、登录地点、浏览器和操作系统；调用 `refreshToken()` 更新登录时间、过期时间，并按配置的分钟数把 `LoginUser` 写入 Redis；最后将 UUID 和用户名放入 JWT claims，并用配置密钥签名（`ruoyi-framework/src/main/java/com/ruoyi/framework/web/service/TokenService.java:115-125, 149-184`）。方法默认有效期由 `application.yml` 的 `token.expireTime` 配置，当前默认是 30 分钟（`ruoyi-admin/src/main/resources/application.yml:95-102`）。

### 2.6 后续 HTTP 请求如何恢复身份

后续请求由 `JwtAuthenticationTokenFilter.doFilterInternal()` 调用 `TokenService.getLoginUser()`：解析请求头中的 JWT，取出 UUID，再用 UUID 从 Redis 查找 `LoginUser`；若成功，则构造 Spring Security Authentication 并写入 `SecurityContext`（`ruoyi-framework/src/main/java/com/ruoyi/framework/security/filter/JwtAuthenticationTokenFilter.java:30-43`；`TokenService.java:63-83`）。`verifyToken()` 在剩余有效时间不超过 20 分钟时刷新 Redis 有效期（`TokenService.java:134-156`）。安全配置将 HTTP Session 设为无状态，并要求其他受保护请求必须认证（`ruoyi-framework/src/main/java/com/ruoyi/framework/config/SecurityConfig.java:97-117`）。

## 3. 异常处理与边界条件

| 情况 | 当前处理 | 代码位置 |
|---|---|---|
| 验证码 key 不存在 | 排入失败日志任务，抛 `CaptchaExpireException` | `SysLoginService.java:115-120` |
| 验证码不匹配 | 先删除验证码，再排入失败日志并抛 `CaptchaException` | `SysLoginService.java:122-126` |
| 用户名/密码缺失、长度不符、IP 被封 | 安排失败日志并抛业务异常 | `SysLoginService.java:136-164` |
| 用户不存在、已删除或停用 | `UserDetailsServiceImpl` 抛 `ServiceException` | `UserDetailsServiceImpl.java:40-55` |
| 密码错误次数达到上限 | `SysPasswordService` 抛锁定异常 | `SysPasswordService.java:50-60` |
| 认证阶段异常 | 密码不匹配单独转换；其他异常包装为 `ServiceException`；最终清理 ThreadLocal | `SysLoginService.java:71-94` |
| 后续请求的 JWT 解析或 Redis 查询异常 | `TokenService.getLoginUser()` 记录错误并返回 `null`；过滤器不建立认证上下文，受保护请求随后仍受安全规则约束 | `TokenService.java:63-83`；`SecurityConfig.java:100-108` |

**可复现边界观察：** `validateCaptcha()` 在 `login()` 的认证 `try/catch` 之前执行（`SysLoginService.java:65-71`）。若验证码启用、Redis 中验证码存在，但请求的 `code` 为 `null`，`code.equalsIgnoreCase(captcha)` 有抛出 `NullPointerException` 的可能。`LoginBody` 的 `code` 字段没有字段级非空约束；异常会落入全局运行时异常处理，而不是 `login()` 中的认证异常分支（`GlobalExceptionHandler.java:96-102`）。这是源码边界分析，不代表当前客户端必然会发送空验证码。

## 4. 并发、缓存与性能设计

### 已实现的设计

- **Redis 承担三类认证数据：** 验证码、密码错误计数和在线 `LoginUser`。每次受保护请求从 Redis 恢复会话主体，避免每个请求都重新按用户名查数据库；登录时仍会查询用户、部门、角色和权限（`RedisCache.java:35-51, 106-119`；`TokenService.java:63-83, 149-156`）。
- **登录日志异步安排：** `AsyncManager.execute()` 使用 `ScheduledExecutorService` 延后 10 毫秒执行；`AsyncFactory.recordLogininfor()` 中的任务最终调用登录日志服务写入记录（`AsyncManager.java:17-45`；`AsyncFactory.java:37-79`）。`ThreadPoolConfig` 为该调度池配置了 50 个核心线程（`ThreadPoolConfig.java:48-61`）。
- **认证上下文按线程隔离：** `AuthenticationContextHolder` 使用 `ThreadLocal`；主认证流程在 `finally` 中移除上下文，避免请求线程复用时残留认证数据（`AuthenticationContextHolder.java:10-27`；`SysLoginService.java:73-94`）。异步日志任务在被安排时捕获 User-Agent 和 IP，执行时不依赖原请求线程的 ThreadLocal（`AsyncFactory.java:37-47`）。
- **权限计算对多角色用户逐角色查询：** 非管理员用户的 `getMenuPermission()` 遍历角色并逐个调用 `selectMenuPermsByRoleId()`。按源码可见，这一段没有批量查询；多角色用户登录时会有多次角色权限查询（`SysPermissionService.java:68-79`；`SysMenuMapper.xml:115-120`）。

### 并发边界（基于实现结构的推断）

1. **密码错误计数不是单条原子递增。** 当前逻辑是 Redis 读计数、Java 加一、再写回，并设置 TTL；`RedisCache` 暴露的是普通 `get` 和 `set`，这段代码没有使用原子 `INCR` 或锁。因此同一账号并发失败时，多个请求可能读到相同旧值并写回相同新值，导致计数少于实际尝试次数。需要通过并发测试确认具体影响。
2. **验证码消费也是读后删除两步操作。** `validateCaptcha()` 先 `getCacheObject()` 再 `deleteObject()`，不是原子读取并消费。在同一验证码被并发提交的极端情况下，存在多个请求都先读到验证码的竞态可能；源码没有使用原子 GET-and-delete。
3. **成功日志先于后续同步副作用排队。** 认证成功后，程序先安排成功日志，再更新用户登录信息和写入 Redis token。若后两步失败，HTTP 登录仍可能失败，但已排队的成功日志仍可能被写入。日志与 token/用户记录不在同一事务中。

### Token 有效期的实现边界

`TokenService` 的 JWT 构造方法只设置 claims 并签名，没有在该方法中显式设置 JWT 的 `exp` claim（`TokenService.java:179-184`）。因此从当前源码看，登录有效期主要依赖 Redis 中 `LoginUser` 的 TTL 和缓存对象上的 `expireTime`，而不是 JWT 自身的过期声明。`verifyToken()` 在缓存对象仍可读取时按剩余时间刷新 TTL。这个实现需要与部署环境中的 token 密钥、Redis 可用性和登出删除缓存行为一并理解。

## 5. 本模块结论

`SysLoginService.login()` 是一条多阶段、有副作用的认证编排：**验证码与登录前置校验 → 用户和状态查询 → 密码及错误次数检查 → 权限装载 → 记录登录信息 → Redis 会话缓存与 JWT 签名**。项目使用 JWT 作为客户端令牌，同时把完整的 `LoginUser` 放在 Redis 中，后续请求经 JWT Filter 恢复认证上下文。

实现中有清晰的认证异常分支、ThreadLocal 清理、Redis 错误计数 TTL 和异步登录日志；需要特别留意的源码边界是验证码参数的空值、Redis 计数/验证码读写的非原子性，以及成功日志和后续 token 写入之间没有事务保障。