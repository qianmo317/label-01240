# Token黑名单机制分析报告

## 1. TokenBlacklist 存储机制

### 1.1 存储结构
Token黑名单使用 `ConcurrentHashMap<String, Long>` 作为存储容器：
- Key: JWT Token 字符串
- Value: Token的过期时间戳（毫秒级）

### 1.2 核心操作
1. **加入黑名单**：用户登出时调用 `TokenBlacklistService.addToBlacklist(token, expirationTime)`，将Token和其过期时间存入Map
2. **黑名单检查**：调用 `isBlacklisted(token)` 检查Token是否存在于Map中
3. **定时清理**：每小时执行一次 `cleanupExpiredTokens()`，自动删除已过期的Token条目

### 1.3 持久化特性
- 黑名单完全存储在JVM内存中，无数据库或磁盘持久化
- 所有数据随应用进程生命周期存在，进程终止则数据全部丢失

## 2. AuthInterceptor 验证流程

### 2.1 请求验证顺序
```
请求到达 → 提取Authorization头 → 去除Bearer前缀 → 检查黑名单 → 验证Token签名和有效性 → 设置用户上下文 → 放行
```

### 2.2 详细步骤
1. **第一步：黑名单检查**：先调用 `tokenBlacklistService.isBlacklisted(token)` 判断Token是否已被登出
   - 若已在黑名单中，直接返回401错误
2. **第二步：签名验证**：调用 `jwtUtil.validateToken(token)` 验证Token签名有效性和是否过期
   - 若签名无效或已过期，返回401错误

### 2.3 设计逻辑
先查黑名单的优势：避免对已登出的Token进行不必要的签名解析计算，提高拦截效率

## 3. 服务重启影响
- 服务重启后，JVM内存中的 `ConcurrentHashMap` 会被重新初始化，原有黑名单数据全部丢失
- 重启前已登出的Token，在重启后只要尚未过期，仍可正常使用
- 黑名单仅在服务运行期间有效，重启后失效
