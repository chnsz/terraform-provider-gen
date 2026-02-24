---
name: "terraform-provider-gen"
description: "Develops Huawei Cloud Terraform Provider resources and data sources. Invoke when user needs to create, modify, or debug Terraform provider code for Huawei Cloud services."
---

# Terraform Provider HuaweiCloud 开发技能

本技能用于指导开发华为云 Terraform Provider 的资源和数据源。

## 技能概述

本技能提供华为云 Terraform Provider 开发的完整指南，包括：

- 项目结构和架构设计
- Resource 开发技能
- Data Source 开发技能
- 验收测试开发
- 文档编写规范
- 最佳实践指南

## 使用场景

当用户需要以下操作时，应调用此技能：

- 创建新的 Resource 或 Data Source
- 修改现有的 Resource 或 Data Source
- 调试 Terraform Provider 代码
- 编写验收测试
- 编写或更新文档
- 了解项目结构和编码规范

## 技能文档

### 01-setup.md - 项目结构介绍

介绍 terraform-provider-huaweicloud 项目的整体结构，包括：
- 目录结构说明
- 核心组件介绍
- 资源命名规范
- 依赖说明
- 开发环境配置

### 02-resource-coding.md - Resource 编码开发技能

以 `services/vpc/resource_huaweicloud_vpc.go` 为例，详细介绍：
- Resource 基本结构
- Schema 定义
- CRUD 函数实现
- 状态等待函数
- 资源导入
- HTTP 请求封装

### 03-datasource-coding.md - Data Source 编码开发技能

以 `services/vpc/data_source_huaweicloud_vpcs.go` 为例，详细介绍：
- Data Source 基本结构
- Schema 设计
- Read 函数实现
- 常见 Data Source 模式
- 分页处理
- 错误处理

### 04-testing.md - 单元测试开发技能

介绍验收测试开发，包括：
- 测试框架概述
- Resource 测试
- Data Source 测试
- 测试工具函数
- 测试环境配置
- 运行测试

### 05-documentation.md - 文档开发技能

介绍文档编写规范，包括：
- 文档目录结构
- Resource 文档模板
- Data Source 文档模板
- 参数描述规范
- 导入文档
- 文档检查清单

### 06-best-practices.md - 最佳实践开发技能

介绍开发最佳实践，包括：
- API 客户端创建
- 错误处理
- 状态等待 (WaitForState)
- HTTP 请求封装
- 标签管理
- 企业项目管理
- 数据转换工具
- 并发控制
- 日志记录
- 编码规范

## 快速开始

### 创建新 Resource

1. 参考 [02-resource-coding.md](./02-resource-coding.md) 了解 Resource 结构
2. 在 `services/<service>/` 目录下创建 `resource_huaweicloud_<name>.go`
3. 实现 CRUD 函数
4. 在 `provider.go` 中注册资源
5. 参考 [04-testing.md](./04-testing.md) 编写测试
6. 参考 [05-documentation.md](./05-documentation.md) 编写文档

### 创建新 Data Source

1. 参考 [03-datasource-coding.md](./03-datasource-coding.md) 了解 Data Source 结构
2. 在 `services/<service>/` 目录下创建 `data_source_huaweicloud_<name>.go`
3. 实现 Read 函数
4. 在 `provider.go` 中注册数据源
5. 编写测试和文档

## 开发流程

```
1. 分析需求 → 2. 分析API → 3. 确定数据结构 → 4. 设计 Schema → 5. 实现 CRUD → 6. 编写测试 → 7. 编写文档 → 8. 代码审查
```

## 注意事项

- 遵循 Terraform Plugin SDK v2 的最佳实践
- 使用 `diag.Diagnostics` 返回错误
- 正确处理资源不存在的情况
- 配置合理的超时时间
- 添加适当的日志记录
- 编写完整的验收测试

## 参考资源

- [Terraform Plugin SDK v2 文档](https://www.terraform.io/plugin/sdkv2)
- [华为云 API 文档](https://support.huaweicloud.com/api/)
- [Terraform Provider 开发指南](https://www.terraform.io/plugin/developing-provider)
