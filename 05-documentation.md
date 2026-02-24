# 文档开发技能

本文档介绍 terraform-provider-huaweicloud 项目的文档开发规范和技能。

## 文档目录结构

```
docs/
├── resources/                    # Resource 文档
│   ├── vpc.md
│   ├── vpc_subnet.md
│   └── ...
├── data-sources/                 # Data Source 文档
│   ├── vpcs.md
│   ├── vpc_subnets.md
│   └── ...
├── json/                         # JSON 格式文档（可选）
│   ├── resources/
│   │   └── vpc.json
│   └── data-sources/
│       └── vpcs.json
└── index.md                      # 文档首页
```

## Resource 文档模板

### 文档结构

```markdown
---
subcategory: "Virtual Private Cloud (VPC)"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_vpc"
description: ""
---

# huaweicloud_vpc

Manages a VPC resource within HuaweiCloud.

## Example Usage

### 基本用法

```hcl
variable "vpc_name" {
  default = "huaweicloud_vpc"
}

variable "vpc_cidr" {
  default = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "vpc" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

### 带标签的用法

```hcl
resource "huaweicloud_vpc" "vpc_with_tags" {
  name = var.vpc_name
  cidr = var.vpc_cidr

  tags = {
    foo = "bar"
    key = "value"
  }
}
```

## Argument Reference

The following arguments are supported:

* `region` - (Optional, String, ForceNew) Specifies the region in which to create the VPC.
  If omitted, the provider-level region will be used. Changing this creates a new VPC resource.

* `name` - (Required, String) Specifies the name of the VPC. The name must be unique for a tenant.
  The value is a string of no more than 64 characters and can contain digits, letters,
  underscores (_), and hyphens (-).

* `cidr` - (Required, String) Specifies the range of available subnets in the VPC.
  The value ranges from 10.0.0.0/8 to 10.255.255.0/24, 172.16.0.0/12 to 172.31.255.0/24,
  or 192.168.0.0/16 to 192.168.255.0/24.

* `description` - (Optional, String) Specifies supplementary information about the VPC.
  The value is a string of no more than 255 characters and cannot contain angle brackets (< or >).

* `tags` - (Optional, Map) Specifies the key/value pairs to associate with the VPC.

* `enterprise_project_id` - (Optional, String) Specifies the enterprise project ID of the VPC.

## Attribute Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The VPC ID in UUID format.

* `status` - The current status of the VPC. Possible values are: CREATING, OK or ERROR.

## Timeouts

This resource provides the following timeouts configuration options:

* `create` - Default is 10 minutes.
* `delete` - Default is 3 minutes.

## Import

VPCs can be imported using the `id`, e.g.

```bash
$ terraform import huaweicloud_vpc.vpc_v1 7117d38e-4c8f-4624-a505-bd96b97d024c
```
```

### 文档头部说明

| 字段 | 说明 |
|------|------|
| `subcategory` | 服务分类名称 |
| `layout` | 固定为 `huaweicloud` |
| `page_title` | 页面标题 |
| `description` | 资源描述（可为空） |

## Data Source 文档模板

### 文档结构

```markdown
---
subcategory: "Virtual Private Cloud (VPC)"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_vpcs"
description: ""
---

# huaweicloud_vpcs

Use this data source to get a list of VPC.

## Example Usage

### 按名称和标签过滤

```hcl
variable "vpc_name" {}

data "huaweicloud_vpcs" "vpc" {
  name = var.vpc_name

  tags = {
    foo = "bar"
  }
}

output "vpc_ids" {
  value = data.huaweicloud_vpcs.vpc.vpcs[*].id
}
```

## Argument Reference

The arguments of this data source act as filters for querying the available VPCs in the
current region. All VPCs that meet the filter criteria will be exported as attributes.

* `region` - (Optional, String) Specifies the region in which to obtain the VPC.
  If omitted, the provider-level region will be used.

* `id` - (Optional, String) Specifies the id of the desired VPC.

* `name` - (Optional, String) Specifies the name of the desired VPC.
  The value is a string of no more than 64 characters and can contain digits,
  letters, underscores (_) and hyphens (-).

* `status` - (Optional, String) Specifies the current status of the desired VPC.
  The value can be CREATING, OK or ERROR.

* `cidr` - (Optional, String) Specifies the cidr block of the desired VPC.

* `enterprise_project_id` - (Optional, String) Specifies the enterprise project ID
  which the desired VPC belongs to.

* `tags` - (Optional, Map) Specifies the included key/value pairs which associated
  with the desired VPC.

## Attribute Reference

The following attributes are exported:

* `id` - The data source ID.

* `vpcs` - The list of all VPCs found. Structure is documented below.

The `vpcs` block supports:

* `id` - The ID of the VPC.

* `name` - The name of the VPC.

* `cidr` - The cidr block of the VPC.

* `status` - The current status of the VPC.

* `enterprise_project_id` - The enterprise project ID of the VPC.

* `description` - The description of the VPC.

* `tags` - The key/value pairs which associated with the VPC.

* `secondary_cidrs` - The secondary CIDR blocks of the VPC.
```

## 参数描述规范

### 参数属性标记

| 标记 | 格式 | 说明 |
|------|------|------|
| Required | `(Required, Type)` | 必需参数 |
| Optional | `(Optional, Type)` | 可选参数 |
| Computed | `(Computed, Type)` | 计算属性 |
| ForceNew | `(Optional, Type, ForceNew)` | 修改时重建 |

### 类型标记

| 类型 | 标记 |
|------|------|
| 字符串 | `String` |
| 整数 | `Int` |
| 布尔 | `Bool` |
| 浮点数 | `Float` |
| 列表 | `List` |
| 集合 | `Set` |
| 映射 | `Map` |

### 参数描述格式

```markdown
* `parameter_name` - (Required/Optional, Type[, ForceNew]) Description of the parameter.
  Additional details or constraints.
```

示例：

```markdown
* `name` - (Required, String) Specifies the name of the VPC. The name must be unique
  for a tenant. The value is a string of no more than 64 characters and can contain
  digits, letters, underscores (_), and hyphens (-).

* `region` - (Optional, String, ForceNew) Specifies the region in which to create
  the VPC. If omitted, the provider-level region will be used. Changing this creates
  a new VPC resource.
```

## 特殊文档元素

### 注意事项

使用引用块添加重要提示：

```markdown
 -> **Note:** A maximum of 10 tag keys are allowed for each query operation.
```

### 警告信息

```markdown
!> **Warning:** The following secondary CIDR blocks cannot be added to a VPC:
10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16.
```

### 废弃说明

```markdown
* `secondary_cidr` - (Optional, String) Specifies the secondary CIDR block of the VPC.
  Use `secondary_cidrs` instead. This parameter is deprecated.
```

### 链接引用

```markdown
[View the complete list of unsupported CIDR blocks](https://support.huaweicloud.com/intl/en-us/usermanual-vpc/vpc_vpc_0007.html).
```

## 导入文档

### 基本导入

```markdown
## Import

VPCs can be imported using the `id`, e.g.

```bash
$ terraform import huaweicloud_vpc.vpc_v1 7117d38e-4c8f-4624-a505-bd96b97d024c
```
```

### 带说明的导入

```markdown
## Import

VPCs can be imported using the `id`, e.g.

```bash
$ terraform import huaweicloud_vpc.vpc_v1 7117d38e-4c8f-4624-a505-bd96b97d024c
```

Note that the imported state may not be identical to your resource definition when
`secondary_cidr` was set. You can ignore changes as below.

```hcl
resource "huaweicloud_vpc" "vpc_v1" {
    ...

  lifecycle {
    ignore_changes = [ secondary_cidr ]
  }
}
```
```

### 复合 ID 导入

```markdown
## Import

Routes can be imported using the `route_table_id/destination`, e.g.

```bash
$ terraform import huaweicloud_vpc_route_table_route.test 7117d38e-4c8f-4624-a505-bd96b97d024c/192.168.0.0/24
```
```

## 超时配置文档

```markdown
## Timeouts

This resource provides the following timeouts configuration options:

* `create` - Default is 10 minutes.
* `update` - Default is 10 minutes.
* `delete` - Default is 3 minutes.
```

## 文档检查清单

### Resource 文档
- [ ] 文档头部元数据正确
- [ ] 示例代码可运行
- [ ] 所有参数都有描述
- [ ] 参数属性标记正确
- [ ] 计算属性已列出
- [ ] 导入说明完整
- [ ] 超时配置已说明（如有）

### Data Source 文档
- [ ] 文档头部元数据正确
- [ ] 示例代码可运行
- [ ] 查询参数描述完整
- [ ] 结果字段结构说明
- [ ] 嵌套字段已文档化

### 通用检查
- [ ] 拼写检查
- [ ] 格式一致
- [ ] 链接有效
- [ ] 代码块语法正确

## 文档生成

项目支持从代码注释自动生成文档。在资源文件顶部添加 API 注释：

```go
// @API VPC POST /v1/{project_id}/vpcs
// @API VPC GET /v1/{project_id}/vpcs/{id}
// @API VPC PUT /v1/{project_id}/vpcs/{id}
// @API VPC DELETE /v1/{project_id}/vpcs/{id}
```

运行文档生成命令：

```bash
make gen-doc
```

## 文档风格指南

### 语言规范

- 使用英文撰写文档
- 使用第三人称描述
- 保持简洁明了

### 格式规范

- 使用 `*` 作为列表标记
- 参数名使用反引号包裹
- 代码块指定语言类型

### 描述规范

- 参数描述以动词开头（Specifies, Indicates, Defines）
- 说明参数的用途和约束
- 提供有效值范围（如适用）
