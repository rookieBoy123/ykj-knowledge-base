# Webhook回调注册与事件驱动推送流程（服务端）

> 文档版本 2026-07-03 | 面向开发人员和技术支持

## 概述

Webhook（回调钩子）用于平台在特定业务事件发生时，自动向外部系统推送通知。管理员可以注册多个回调地址，绑定到不同的事件类型（如"问诊创建"、"处方签章完成"、"订单支付成功"等）。系统使用Redis缓存已注册的Webhook列表，提高事件触发时的查询效率。

## 触发条件

- 外部系统（如HIS、第三方药品平台）需要实时接收平台业务事件
- 需要为不同事件类型配置不同的回调地址
- 回调地址变更或失效时需要更新或停用

## 步骤详解

### 第一步：系统启动加载缓存
系统启动时（`@PostConstruct`），自动执行初始化：
1. 从数据库查询所有状态为"启用"的Webhook记录
2. 按事件类型（event）分组，形成 `event -> [webhook列表]` 的映射
3. 将每个事件的Webhook列表以JSON格式写入Redis缓存
4. 缓存过期时间设为1小时（`Duration.ofHours(1)`），1小时后自动失效并从数据库重新加载

### 第二步：注册新的Webhook
管理员提交新增请求，系统执行：
1. 将Webhook状态默认为"启用"
2. 写入数据库
3. **清除Redis缓存**：删除该Webhook关联的所有事件类型的缓存key（`webhook:{event}`），确保下次查询时从数据库加载最新数据

### 第三步：业务事件触发回调
当业务模块（如问诊模块）产生事件时，调用 `findByEvent(event)` 查询该事件对应的Webhook列表：

**缓存查询流程**：
1. 先从Redis根据 `webhook:{event}` 键查询缓存
2. 如果缓存命中，直接返回缓存的Webhook列表
3. 如果缓存未命中（过期或被清除），从数据库查询该事件类型下所有启用的Webhook
4. 将查询结果写入Redis缓存（1小时过期），然后返回

业务模块拿到Webhook列表后，遍历每个回调地址，发送HTTP请求携带事件数据。

### 第四步：管理Webhook
- **编辑**：修改Webhook的URL、事件类型等配置，修改后清除关联事件的缓存
- **删除**：支持批量删除（按ID列表），删除后清除被删Webhook关联的所有事件缓存
- **启用/停用**：修改状态字段（ENABLE/DISABLE），修改后清除缓存。停用的Webhook不会出现在事件查询结果中

### 第五步：缓存自动维护
每次执行以下操作时，系统自动清除相关事件的Redis缓存：
- 新增Webhook
- 编辑Webhook
- 删除Webhook
- 启用/停用Webhook

清除缓存的key格式为 `webhook:{event}`（如 `webhook:order.created`、`webhook:prescription.signed`）。

## 异常处理

| 情况 | 系统处理方式 | 来源 |
|---|---|---|
| 启用/停用时指定不存在的Webhook | 返回"参数有误，请检查" | Service |
| 批量删除时指定不存在的ID | 不影响，只删除存在的 | MyBatis-Plus |
| Redis缓存不可用 | 自动降级为数据库查询 | Service |
| 新增操作数据库失败 | 返回"新增失败" | Controller |

## 缓存策略

| 场景 | 处理 |
|---|---|
| 系统启动 | 加载所有启用Webhook到Redis，过期1小时 |
| 配置变更 | 立即清除关联事件缓存，触发时重新加载 |
| 缓存过期 | 下次查询时自动从数据库重新加载 |
| 缓存命中 | 直接返回，避免数据库查询 |

## 关联功能

- **管理后台-Webhook管理页面**：前端通过该Controller实现CRUD
- **业务模块事件发布**：问诊、订单、处方、会诊等模块在关键节点发布事件，触发Webhook查询与推送
- **Redis缓存**：用于缓存Webhook配置，key格式 `webhook:{event}`，过期1小时

## 代码来源

- `.../controller/mgmt/MgmtSysWebhookController.java` (74行)
- `.../service/impl/SysWebhookServiceImpl.java` (126行)
