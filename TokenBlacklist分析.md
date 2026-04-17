# TokenBlacklist 存储机制与 AuthInterceptor 调用链分析

## 一、TokenBlacklist 存储机制

### 1.1 存储结构
TokenBlacklist 使用 **内存存储** 方案，具体实现位于 `TokenBlacklistService.java:19`：

```java
private final Map<String, Long> blacklist = new ConcurrentHashMap<>();
```

- **存储容器**：`ConcurrentHashMap`（线程安全的哈希表）
- **Key**：JWT token 字符串
- **Value**：token 的过期时间戳（毫秒）

### 1.2 存储操作

#### 添加到黑名单
调用位置：`AuthController.java:59-60`（登出接口）
```java
// AuthController.logout()
long expirationTime = jwtUtil.getExpirationTime(token);
tokenBlacklistService.addToBlacklist(token, expirationTime);
```

存储方法：`TokenBlacklistService.java:26-29`
```java
public void addToBlacklist(String token, long expirationTime) {
    blacklist.put(token, expirationTime);
    log.info("Token已加入黑名单");
}
```

### 1.3 清理机制
定时清理过期 token，位于 `TokenBlacklistService.java:43-56`：
- 触发方式：`@Scheduled(fixedRate = 3600000)`，每小时执行一次
- 清理逻辑：遍历 Map，移除 `expirationTime < now` 的条目

### 1.4 服务重启后的状态
**服务重启后，黑名单中的数据会全部丢失**，原因：
- 黑名单存储在 `ConcurrentHashMap` 中，属于 JVM 堆内存
- 服务重启意味着 JVM 进程重启，内存中的所有数据都会被清空
- 没有持久化到数据库或磁盘文件

---

## 二、AuthInterceptor 验证调用链

### 2.1 完整验证流程
验证顺序在 `AuthInterceptor.java:40-94` 的 `preHandle()` 方法中，执行顺序如下：

```
请求到达
   ↓
1. OPTIONS请求直接放行（CORS预检）
   ↓
2. 提取请求头中的 Authorization token
   ↓
3. 检查是否为公开GET接口
   ├─ 是公开接口 + 无token → 放行
   └─ 非公开接口 + 无token → 抛401"未登录"
   ↓
4. 去掉token的"Bearer "前缀
   ↓
5. **检查token是否在黑名单中**（TokenBlacklistService.isBlacklisted()）
   ├─ 在黑名单中 + 是公开接口 → 放行
   └─ 在黑名单中 + 非公开接口 → 抛401"登录已失效"
   ↓
6. **验证token签名有效性**（JwtUtil.validateToken()）
   ├─ token无效 + 是公开接口 → 放行
   └─ token无效 + 非公开接口 → 抛401"登录已过期"
   ↓
7. 解析token，设置用户上下文（UserContext）
   ↓
放行
```

### 2.2 关键顺序说明
**先检查黑名单，后验证签名**，具体代码位置：

- 第68行：先检查黑名单
  ```java
  if (tokenBlacklistService.isBlacklisted(token)) {
      // 已登出处理
  }
  ```

- 第75行：后验证签名
  ```java
  if (!jwtUtil.validateToken(token)) {
      // token过期处理
  }
  ```

### 2.3 公开接口的特殊处理
对于 `PUBLIC_GET_PATTERNS` 中定义的 GET 接口：
- 无 token：放行
- 有 token 但在黑名单：放行
- 有 token 但签名无效：放行
- 有有效 token：正常解析用户上下文

---

## 三、核心调用链总结

### 登出流程
```
POST /api/auth/logout
   ↓
AuthController.logout()
   ↓
提取token → 解析过期时间 → TokenBlacklistService.addToBlacklist()
   ↓
存入 ConcurrentHashMap
```

### 请求验证流程
```
请求 → AuthInterceptor.preHandle()
           ↓
           ├─ 提取token
           ├─ 检查黑名单（内存查询）
           ├─ 验证JWT签名
           └─ 设置用户上下文
```

### 关键结论
1. **黑名单存储**：纯内存存储（ConcurrentHashMap），服务重启后清空
2. **验证顺序**：先查黑名单，后验签名
3. **持久化**：无持久化机制，仅依赖定时清理过期token
