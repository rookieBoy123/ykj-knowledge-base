# 科室层级管理与HIS同步流程（服务端）

> 文档版本 1.0 | 面向读者：产品/技术

## 概述

服务端的科室管理支持**树形层级结构**（父科室→子科室），并针对互联网诊疗机构提供**从HIS系统同步科室数据**的能力。科室数据作为医生注册、排班管理、问诊分诊的基础数据。

## 涉及服务模块

- `ykj-service-hospital`：科室增删改查、树形查询
- `ykj-service-his`：HIS科室同步（互联网诊疗模式）

## 数据结构

科室表核心字段：

| 字段 | 说明 |
|------|------|
| id | 主键 |
| parentId | 上级科室ID（null表示一级科室） |
| code | 院内科室编码（唯一，最多10位） |
| name | 科室名称（最多32字） |
| typeCode | 科室类型编码（关联字典"kslx"） |
| stdCode | 标准科室编码（关联字典"ksbm"） |
| insuranceCode | 医保科室编码（关联字典"ybks"） |
| insuranceName | 医保科室名称（冗余字段，提交时自动填入） |
| description | 科室简介（最多500字） |

## 管理端接口

### 1. 查询科室树 `GET /mgmt/departments/tree`
- 返回完整树形结构（parentId关联）
- 支持按科室名称、标准科室、科室类型筛选
- 查询全部数据（不分页），前端进行树形过滤

### 2. 新增科室 `POST /mgmt/departments`
- 必填：code、name、typeCode、stdCode
- 可选：parentId（选择后即不可修改）、insuranceCode、description
- 提交时自动填充insuranceName（通过ybksObj映射）

### 3. 编辑科室 `PUT /mgmt/departments`
- code和parentId不可修改
- 互联网诊疗模式下name不可修改（只读标记）
- 其它字段正常可修改

### 4. 删除科室 `DELETE /mgmt/departments`
- 按ID删除
- 需注意：删除父科室时，其子科室的处理策略取决于业务规则（级联删除或保留为一级科室）

### 5. 按条件查询单个科室 `GET /mgmt/departments/{id}`
- 用于编辑时回填数据

## HIS同步模式

互联网诊疗机构（internetDiagnosis=true）时：

**管理后台行为变化**：
- 「新增」按钮隐藏
- 「同步HIS科室」按钮显示
- 编辑时科室名称只读

**服务端同步流程**：
1. 管理后台触发同步请求
2. 服务端调用HIS接口获取科室列表
3. 对比现有科室数据，执行增量同步
4. 新科室自动创建，已有科室更新信息

## 双模式对比

| 对比维度 | 普通模式 | 互联网诊疗模式 |
|---------|---------|---------------|
| 科室来源 | 手动新增 | HIS同步 |
| 新增接口 | POST /mgmt/departments | 通过HIS同步接口 |
| 科室名称修改 | 可修改 | 只读 |
| 权限控制 | `hosp:department:write` | `hosp:department:write` |
| 适用条件 | internetDiagnosis=false | internetDiagnosis=true |

## 关联功能

- 医生信息管理：医生关联科室
- 排班管理：按科室生成排班计划
- 问诊分诊：按科室分配问诊
- 诊疗科目配置（specialtyPractice）：标准科室编码的映射来源
