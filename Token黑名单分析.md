# Token 黑名单机制分析

## 一、存储机制

### 1.1 存储结构

TokenBlacklistService 使用 **JVM 内存**存储黑名单：

```java
// TokenBlacklistService.java:19
private final Map<String, Long> blacklist = new ConcurrentHashMap<>();
```

- **存储容器**：`ConcurrentHashMap`（线程安全的 HashMap）
- **Key**：完整的 JWT token 字符串
- **Value**：token 的过期时间戳（毫秒）

### 1.2 数据生命周期

| 操作 | 触发时机 | 说明 |
|------|----------|------|
| **写入** | 用户调用 `/api/auth/logout` 登出时 | AuthController.logout() → TokenBlacklistService.addToBlacklist() |
| **查询** | 每次请求经过 AuthInterceptor 时 | preHandle() 中调用 isBlacklisted() |
| **删除** | 定时任务每小时执行一次 | cleanupExpiredTokens() 清理已过期的 token |

### 1.3 持久化问题

**结论：服务重启后黑名单数据会全部丢失**

原因：
- 黑名单存储在 `ConcurrentHashMap` 中，属于 JVM 堆内存
- 没有任何持久化机制（数据库、Redis、文件等）
- 服务重启后，JVM 内存被回收，`blacklist` Map 会重新初始化，变为空 Map

## 二、AuthInterceptor 调用链

### 2.1 执行顺序（关键）

**先检查黑名单，再验证 token 签名**

```
请求到达
    ↓
AuthInterceptor.preHandle()
    ├─ 1. OPTIONS 请求直接放行 (42-44行)
    ├─ 2. 提取 token (48行)
    ├─ 3. 无 token 时判断是否为公开接口 (54-60行)
    ├─ 4. 去掉 Bearer 前缀 (62-65行)
    ├─ 5. **检查黑名单** (67-73行) ← 先检查黑名单
    │    └─ tokenBlacklistService.isBlacklisted(token)
    │    └─ 如果在黑名单中：抛 401 异常
    ├─ 6. **验证 token 签名/有效性** (75-81行) ← 后验证签名
    │    └─ jwtUtil.validateToken(token)
    │    └─ 如果无效：抛 401 异常
    ├─ 7. 解析 token 设置用户上下文 (83-90行)
    └─ 8. 放行 (93行)
```

### 2.2 详细调用链

```
1. 用户登出流程：
   POST /api/auth/logout
      → AuthController.logout() (53-64行)
         → 提取 Authorization header 中的 token
         → jwtUtil.getExpirationTime(token) 获取过期时间
         → tokenBlacklistService.addToBlacklist(token, expirationTime)
            → blacklist.put(token, expirationTime) 写入 Map

2. 请求认证流程：
   Any Request
      → AuthInterceptor.preHandle() (40-94行)
         → 提取 token
         → tokenBlacklistService.isBlacklisted(token) (68行)
            → blacklist.containsKey(token) 检查 Map
         → 如不在黑名单，继续 jwtUtil.validateToken(token) (75行)
         → 验证通过，设置 UserContext
```

### 2.3 公开接口的特殊处理

对于公开的 GET 接口（`PUBLIC_GET_PATTERNS` 中定义）：
- token 在黑名单中 → 允许访问（不抛异常）
- token 无效或过期 → 允许访问（不抛异常）
- 本质是：匿名身份访问公开接口

## 三、定时清理机制

```java
// TokenBlacklistService.java:43-56
@Scheduled(fixedRate = 3600000)  // 每小时执行一次
public void cleanupExpiredTokens() {
    // 遍历 blacklist，移除 value（过期时间）< 当前时间的条目
}
```

- 执行频率：每小时（3600000ms）
- 清理条件：`entry.getValue() < now`（token 自然过期）
- 目的：防止 blacklist Map 无限增长导致内存溢出
