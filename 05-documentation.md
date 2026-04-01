# 什么时候触发

当我提到以下内容之一时触发
- `文档`
- `数据源文档`
- `资源文档`
- `一次性资源文档`
- `docs`
- `document`
- `markdown文档`
- `md文档`
- `data source文档`
- `resource文档`
- `action 文档`

# 文档目录结构
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

# SKILL 概览

- **文档基本写法**（data source、resource、action resource 三者通用）
- **文档独有写法**
  - resource 文档独有写法
  - action resource 文档独有写法

# 文档模板
参考项目根目录下的: 
- Read: data source 文档模板：`references/docs/data-source-template.md`
- Read: resource 文档模板：`references/docs//resource-template.md`
- Read: action resource 文档模板：`references/docs//action-resource-template.md`

# 文档基本写法
- data source文档、resource文档 和 action resource文档 三者通用

## 基本规范

### 文档存放位置
- data source ：`docs/data-sources/`
- resource 文档存放位置：`docs/resources/`
- 目录已存在，无需新建目录

### 文档命名
- 根据 schema 文件名转换
- 举例1：
  - schema 文件：`data_source_huaweicloud_vpc_subnets.go`
  - 文档文件：`vpc_subnets.md`
- 举例2：
  - schema 文件：`data_source_huaweicloud_workspace_users.go`
  - 文档文件：`workspace_users.md`

### 字段描述规范
- **Argument**：描述前加 `Specifies`（已包含则无需添加）
- **Attribute**：描述以 `The` 开头（已包含则无需添加）
- **Bool类型**：描述以 `Whether` 开头（已包含则无需添加）

### 结构体超链接命名

有 `2` 种情况：
- **情况1：** Argument 与 Attribute 字段名不同
- **情况2：** Argument 与 Attribute 字段名相同

#### 情况1：Argument 与 Attribute 字段名不同
- 去除服务名，保留 `资源名+字段名`
- **举例1：**
  - 服务名 `workspace`，资源名 `huaweicloud_workspace_desktop_pools`，字段名 `volume`
  - 则超链接命名为：`desktop_pools_volume`
- **举例2：**
  - 服务名 `vpc`，资源名 `huaweicloud_vpc_subnets_xxx_yyy`，字段名 `zzz`
  - 则超链接命名为：`subnets_xxx_yyy_zzz`

#### 情况2：Argument 与 Attribute 字段名相同
- 去除服务名，保留 `资源名+字段名+arg` 或 `资源名+字段名+attr`
- **举例1：**
  - 服务名 `workspace`，资源名 `huaweicloud_workspace_desktop_pools`，`Argument` 和 `Attribute` 有相同字段名 `volume`
  - `Argument` 超链接命名为：`desktop_pools_volume_arg`
  - `Attribute` 超链接命名为：`desktop_pools_volume_attr`
- **举例2：**
  - 服务名 `vpc`，资源名 `huaweicloud_vpc_subnets_xxx_yyy`，`Argument` 和 `Attribute` 有相同字段名 `zzz`
  - `Argument` 超链接命名为：`subnets_xxx_yyy_zzz_arg`
  - `Attribute` 超链接命名为：`subnets_xxx_yyy_zzz_attr`

### ForceNew 属性
- data source 所有字段均无此属性
- resource 需要根据schema中的字段定义是否有`ForceNew`属性
- action resource 所有字段均无此属性

## 文档示例

### 文档开头

```md
---
subcategory: "Workspace"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_workspace_users"
description: |-
  Use this data source to query the Workspace users under a specified region within HuaweiCloud.
---
```

#### subcategory
- 云服务名称

#### layout
- 云服务布局，来源于project名称`terraform-provider-huaweicloud`，固定值`huaweicloud`

#### page_title
- 云服务页面标题，根据schema文件转换，例如`data_source_huaweicloud_workspace_users.go`转换为`huaweicloud_workspace_users`

#### description
- 该数据源/资源的作用描述
  - data source：`Use this data source to query the Workspace users under a specified region within HuaweiCloud.`
  - resource：`Manages a Workspace desktop resource within HuaweiCloud.`
  - action resource为：`Use this resource to create a Workspace desktop within HuaweiCloud.`

### 文档主题和使用示例

````md
# huaweicloud_workspace_users

Use this data source to query the Workspace users under a specified region within HuaweiCloud.

## Example Usage

### Basic Usage

```hcl
data "huaweicloud_workspace_users" "test" {}
```

### Filter by user name

```hcl
variable "user_name" {}

data "huaweicloud_workspace_users" "test" {
  user_name = var.user_name
}
```
````

#### 一级标题
- data source 名称（如 `huaweicloud_workspace_users`）

#### 二级标题
- `Example Usage` - 示例使用章节

#### 三级标题
- 具体使用场景，一般2个使用示例
  - `Basic Usage` - 基本使用示例，只包含最小参数（查询所有Workspace用户，必选）
  - `Filter by user name` - 过滤查询示例，包含可选参数（根据用户名查询，可选）

### Argument Reference

```md
## Argument Reference

The following arguments are supported:

* `region` - (Optional, String) Specifies the region in which to query the users.  
  If omitted, the provider-level region will be used.

* `name` - (Optional, String) Specifies the user name to be queried.

* `description` - (Optional, String) Specifies the user description for fuzzy matching.

* `type` - (Optional, String) Specifies the activation type of the user.  
  The valid values are as follows:
  + **USER_ACTIVATE**
  + **ADMIN_ACTIVATE**

* `path` - (Optional, Int) Specifies the path of the probe.  
  This parameter is only available when the `type` is set to **USER_ACTIVATE**.

* `source` - (Required, String) Specifies the source configuration of the component, in JSON format.  
  For the keys, please refer to the [documentation](https://support.huaweicloud.com/intl/en-us/api-servicestage/servicestage_06_0076.html#servicestage_06_0076__en-us_topic_0220056058_ref28944532).

* `users` - (Optional, String) Specifies the user information
  The [users](#workspace_users_arg) structure is documented below.

<a name="workspace_users_arg"></a>
The `users` block supports:

* `age` - The age of the user.

```

#### 二级标题
- `Argument Reference` - 固定标题，参数参考章节

#### 不在文档中体现的参数
- 分页参数：`limit`、`offset`、`marker`

#### Argument 参数描述
- 根据获取的 `API` 参数，使用英文描述参数的作用和取值范围

#### 请求参数（Argument）
参数如 `region`、`name`、`description`、`type`、`path`、`source`、`users` 等

- `region`：固定写法，替换描述中的 `do something`：`Specifies the region in which to do something. If omitted, the provider-level region will be used.`
  - 全局服务没有 `region` 参数
  - `region` 参数只在 `Argument` 中出现

- `type`：固定取值参数（如 `USER_ACTIVATE`、`ADMIN_ACTIVATE`）
  - 参数描述后添加2个空格，然后另起一行添加：`The valid values are as follows:`
  - 最后列出所有可能的取值

- `path`：描述中引用其他字段（如 `type`），需要在描述中对字段名添加反引号，对应的value加粗
  - This parameter is only available when the `type` is set to **USER_ACTIVATE**.

#### 参数描述格式：`(Optional, String)`
- 第1个参数：必选/可选（`Required` 或 `Optional`）
- 第2个参数：参数类型（`String`、`Int`、`Bool`、`Float`、`Map`、`List`、`Set`）
- data source 无第3个参数

#### URL 超链接：如 `source` 字段中的链接
- 格式：`[documentation](链接地址)`


### Attribute Reference
```md
## Attribute Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The data source ID.

* `gender` - The gender of the user.

  ->**Note:** This parameter only returns in **630** and later version.

* `chinese` - The chinese score of the user.

* `math` - The math score of the user.

~>**WARNING:** The `chinese` and `math` only returns in **630** and later version.

* `users` - The list of users that matched filter parameters.  
  The [users](#workspace_users_attr) structure is documented below.

<a name="workspace_users_attr"></a>
The `users` block supports:

* `id` - The ID of the user.

* `sid` - The SID of the user.
```

#### 二级标题
- `Attribute Reference` - 固定标题，属性值参考章节

#### 不在文档中体现的参数
- 数组长度参数：`count`、`total`、`total_count`

#### Attribute 参数描述
- 根据获取的 `API` 参数，使用英文描述参数的作用和取值范围

#### 属性值（Attribute）
如 `id`、`users`、`users.id`、`users.sid` 等

- `id`：data source 的唯一标识，即使没有其他 Attribute 参数，也会有此字段

#### 参数描述格式
- 属性值无需标注可选/必选、参数类型等描述

#### 说明信息
- **单一字段说明**：如果需要额外说明，则字段描述结束后空一行，添加 2 个空格，再添加 Note
```md
* `gender` - The gender of the user.

  ->**Note:** This parameter only returns in **630** and later version.
```

- **多字段说明**：如果需要额外说明，则字段描述结束后空一行，添加 2 个空格，再添加 Warning
```md
* `chinese` - The chinese score of the user.

* `math` - The math score of the user.

~>**WARNING:** The `chinese` and `math` only returns in **630** and later version.
```

- **全局说明**：如果需要额外说明，则置于文档最前方
```md
# huaweicloud_workspace_users

->**Note:** This resource can only be used in **530** and later version.

Use this data source to query the Workspace users under a specified region within HuaweiCloud.

## Example Usage

### Basic Usage
```

### Timeouts
- data source 没有 Timeouts 章节

### Import
- data source 没有 Import 章节

# 文档独有写法

## resource文档独有写法

### 参数描述格式：`(Optional, String, ForceNew)`
- 第1个参数：必选/可选（`Required` 或 `Optional`）
- 第2个参数：参数类型（`String`、`Int`、`Bool`、`Float`、`Map`、`List`、`Set`）
- 第3个参数：根据 schema 中的字段定义判断是否添加ForceNew
  - schema中定义了 `ForceNew` 为 `true`，则第3个参数为 `ForceNew`
  - schema中定义了 `ForceNew` 为 `false`，则第3个参数为空

### Timeouts

**什么时候添加 Timeouts 章节**
- 如果 Schema 中没有定义`Timeouts`，则不添加 Timeouts 章节
- 如果 Schema 中有定义`Timeouts`，在文档的 `Attribute Reference` 章节之后中添加

#### schema 中定义的 Timeouts
```go
Timeouts: &schema.ResourceTimeout{
  Create: schema.DefaultTimeout(30 * time.Minute),
  Update: schema.DefaultTimeout(20 * time.Minute),
  Delete: schema.DefaultTimeout(10 * time.Minute),
},
```

#### 文档中添加的 Timeouts 章节
```md
## Timeouts

This resource provides the following timeouts configuration options:

* `create` - Default is 30 minutes.
* `update` - Default is 20 minutes.
* `delete` - Default is 10 minutes.
```

### Import

**什么时候添加 Import 章节**
- 如果 Schema 中没有定义`Import`，则不添加 Import 章节
- 如果 Schema 中有定义`Import`，在文档的 `Timeouts` 章节之后中添加，如果没有 Timeouts 章节，则添加在 `Attribute Reference` 章节之后

#### schema 中定义的 Import
```go
Importer: &schema.ResourceImporter{
  StateContext: schema.ImportStatePassthroughContext,
},
```

#### 文档中添加的 Import 章节
````md
## Import

Users can be imported using the `id`, e.g.

```bash
$ terraform import huaweicloud_workspace_user.test <id>
```

Note that the imported state may not be identical to your resource definition, due to some attributes missing from the
API response, security or some other reason.
The missing attributes include: `password`.
It is generally recommended running `terraform plan` after importing the resource.
You can then decide if changes should be applied to the user, or the resource definition should be updated to
align with the user. Also you can ignore changes as below.

```hcl
resource "huaweicloud_workspace_user" "test" {
  ...

  lifecycle {
    ignore_changes = [
      password,
    ]
  }
}
```
````

**不返回参数说明** The missing attributes include: `password`. 这些参数的特征同时满足以下2个条件：
1.Schema 没有对该参数定义Computed: `Computed: true`
2.Schema Read方法中没有对该参数进行Set: `d.Set(xxx)`

## action resource 独有写法

### 文档开头说明
在 `Example Usage` 之前需要说明该 resource 是 action resource 及其作用。

**文档示例：**
```md
# huaweicloud_workspace_desktop_user_batch_attach

Use this resource to batch attach users to desktops within HuaweiCloud.

-> This resource is a one-time action resource for batch attaching users to a desktop. Deleting this resource will
   not clear the corresponding request record, but will only remove the resource information from the tfstate file.

## Example Usage
其他内容...
```

**描述来源：** 对应 Schema 中的 Delete 方法

```go
func resourceDesktopUserBatchAttachDelete(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
	errorMsg := `This resource is a one-time action resource for batch attaching users to a desktop. Deleting this
    resource will not clear the corresponding request record, but will only remove the resource information
    from the tfstate file.`
	return diag.Diagnostics{
		diag.Diagnostic{
			Severity: diag.Warning,
			Summary:  errorMsg,
		},
	}
}
```

### 参数描述格式：`(Optional, String, NonUpdatable)`
- 第1个参数：必选/可选（`Required` 或 `Optional`）
- 第2个参数：参数类型（`String`、`Int`、`Bool`、`Float`、`Map`、`List`、`Set`）
- 第3个参数：根据 schema 中的 `NonUpdatableParams []string` 数组判断该参数是否添加 `NonUpdatable`
  - 如果 schema 中定义了 `NonUpdatableParams` 数组，且该参数在数组中，则第3个参数为 `NonUpdatable`
  - 如果 schema 中没有定义 `NonUpdatableParams` 数组，或该参数不在数组中，则第3个参数为空

### Timeouts
- action resource 没有 Timeouts 章节

### Import
- action resource 没有 Import 章节
