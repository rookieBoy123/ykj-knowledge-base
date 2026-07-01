# CA签章用户管理流程

> 文档版本：2026-07-01 | 面向读者：技术管理员

## 概述

CA（电子认证）签章系统负责管理医生和药师的电子签名账号。用户账号在CA系统中创建、同步到第三方CA平台（医网信/源康健PaaS/放心签），完成实名认证后获得电子签章能力，用于在处方、病历、会诊报告等医疗文书上完成具有法律效力的电子签名。

## 触发条件

- 医生注册或资料更新时自动创建CA账号
- 管理员手动触发CA账号同步
- CA认证回调通知

## 步骤详解

### 第一步：创建CA账号（createAccount）

系统收到同步请求（`UserSynDto` 包含姓名、身份证、手机号等）后：

1. 按姓名+加密身份证查询是否已有账号
2. 如果有 → 复用已有 `userNo`（一个自然人在系统中只有一个CA账号）
3. 如果没有 → 生成新 `userNo`（19字节随机数Base64编码为25位字符串，含递归防重复）
4. 调用活跃的CA平台策略 `getActiveStrategy().sync()` 同步用户信息到第三方CA平台，获取 `openId`
5. 如果同步失败（`openId` 为空），记录错误日志，返回 null
6. 创建 `CaUserEntity` 用户记录 + `CaUserPlatform` 平台关联记录（状态设为 `Authing`）

### 第二步：CA认证（authByUserNo）

当用户需要进行电子签章时，系统发起认证流程：

1. 查询用户是否存在
2. 查询该用户在当前CA平台上的记录
3. 组装 `AuthUser`（openId + 姓名 + 证件类型 + 证件号 + 手机号）
4. 调用CA平台策略的 `auth()` 方法获取认证响应
5. 认证结果在回调中异步处理

### 第三步：认证结果处理（handlerAuthed）

CA平台完成用户认证后异步回调：

- 认证成功：`authStatus = AuthStatus.Succeed`，记录签章图片（stamp）
- 认证失败：`authStatus = AuthStatus.Failure`，记录失败原因
- 同时更新 `caSubStatus` 子状态
- 查询主用户记录 → 发布 `UserAuthEvent` 认证事件通知其他模块

### 第四步：密码与签章设置

**密码设置（passwordSetting）**：
1. 检查当前CA平台是否允许用户修改设置（`allowUpdateSetting()`）
2. 如果用户还没有签章图片，自动生成一个默认签章（将用户名字转为Base64图片）
3. 密码用MD5加密后存储

**签章图片设置（stampImageSetting）**：
1. 同样检查平台是否允许修改
2. 必须先验证密码："您的密码验证不通过，请重新输入"
3. 更新签章图片

### 第五步：手机号变更

用户变更手机号时：
1. 查询 `CaUserEntity` + `CaUserPlatform` 两个表
2. 更新本地用户手机号（加密存储）
3. 重新同步到CA平台（`getActiveStrategy().sync()` 带新手机号）
4. 如果CA平台同步失败，当前请求失败并回滚
5. 成功后平台状态重新设为 `Authing`（需要CA平台重新确认）

### 第六步：账号停用

调用 `deactivateAccount` 将平台的认证状态重置为 `NotAuth`（未认证），相当于撤销该用户在CA平台上的认证状态。

### 第七步：密码验证

`verifyPassword` 方法用于签章图片设置前的安全检查，将用户输入密码MD5加密后与存储值比对。

## 异常处理

| 情况 | 系统处理方式 |
|------|-------------|
| CA服务配置异常 | 后端拦截："CA服务配置异常" |
| 账号不存在 | 后端拦截："账号不存在" |
| CA同步失败 | 记录错误日志，createAccount返回null |
| 认证回调openId不匹配 | 记录warn日志，静默跳过 |
| 平台不支持用户修改设置 | 后端拦截："暂不支持用户修改配置" |
| 密码验证不通过 | 后端拦截："您的密码验证不通过，请重新输入" |
| 变更手机号CA同步失败 | 后端拦截："变更手机号码失败，请稍后重试" |
| 切换平台后旧数据不兼容 | @PostConstruct自动修复appId为null的旧数据 |

## 关联功能

- **处方签章**：PrescriptionSignServiceImpl 调用CA服务完成处方电子签名
- **会诊签章**：ConsultationSignServiceImpl 调用CA服务完成会诊报告电子签名
- **病历签章**：MedicalRecordServiceImpl 调用CA服务完成电子病历签名
- **医生管理**：医生注册时自动触发CA账号创建
- **会员系统**：手机号变更时同步更新CA系统

## 代码来源

CaUserServiceImpl.java（411行） + CaUserPlatform + CaUserEntity
