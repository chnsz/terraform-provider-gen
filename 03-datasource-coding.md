# Data Source 编码开发技能

本文档以 `services/vpc/data_source_huaweicloud_vpcs.go` 为例，详细介绍 Data Source 的开发技能。

## Data Source 基本结构

### 1. Data Source 定义函数

Data Source 定义函数返回一个 `*schema.Resource` 对象：

```go
// @API VPC GET /v2.0/{project_id}/vpcs/{id}/tags
// @API VPC GET /v1/{project_id}/vpcs
func DataSourceVpcs() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceVpcsRead,

        Schema: map[string]*schema.Schema{
            // 查询参数
            "region": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            "id": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            "name": {
                Type:     schema.TypeString,
                Optional: true,
            },
            // ... 其他查询参数

            // 结果字段
            "vpcs": {
                Type:     schema.TypeList,
                Computed: true,
                Elem: &schema.Resource{
                    Schema: map[string]*schema.Schema{
                        "id": {
                            Type:     schema.TypeString,
                            Computed: true,
                        },
                        // ... 其他结果字段
                    },
                },
            },
        },
    }
}
```

### 2. Data Source 与 Resource 的区别

| 特性 | Data Source | Resource |
|------|-------------|----------|
| CRUD 操作 | 只有 Read | Create, Read, Update, Delete |
| Schema 用途 | 查询参数 + 结果字段 | 资源属性 |
| ID 设置 | 基于查询结果生成 | API 返回的资源 ID |
| 状态管理 | 无状态 | 有状态 |

## Schema 设计

### 查询参数设计

查询参数用于过滤数据源结果：

```go
Schema: map[string]*schema.Schema{
    "region": {
        Type:     schema.TypeString,
        Optional: true,
        Computed: true,
    },
    "id": {
        Type:     schema.TypeString,
        Optional: true,
        Computed: true,
    },
    "name": {
        Type:     schema.TypeString,
        Optional: true,
    },
    "cidr": {
        Type:     schema.TypeString,
        Optional: true,
    },
    "enterprise_project_id": {
        Type:     schema.TypeString,
        Optional: true,
    },
    "status": {
        Type:     schema.TypeString,
        Optional: true,
    },
    "tags": {
        Type:     schema.TypeMap,
        Optional: true,
        Elem:     &schema.Schema{Type: schema.TypeString},
    },
},
```

### 结果字段设计

结果字段通常是列表类型，包含嵌套对象：

```go
"vpcs": {
    Type:     schema.TypeList,
    Computed: true,
    Elem: &schema.Resource{
        Schema: map[string]*schema.Schema{
            "id": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "name": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "cidr": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "enterprise_project_id": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "status": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "description": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "tags": {
                Type:     schema.TypeMap,
                Computed: true,
                Elem:     &schema.Schema{Type: schema.TypeString},
            },
            "secondary_cidrs": {
                Type:     schema.TypeList,
                Computed: true,
                Elem:     &schema.Schema{Type: schema.TypeString},
            },
        },
    },
},
```

## Read 函数实现

### 基本结构

```go
func dataSourceVpcsRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    region := cfg.GetRegion(d)
    
    // 1. 创建客户端
    v1client, err := cfg.NetworkingV1Client(region)
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }

    v2Client, err := cfg.NetworkingV2Client(region)
    if err != nil {
        return diag.Errorf("error creating VPC V2 client: %s", err)
    }

    // 2. 构建查询参数
    listOpts := vpcs.ListOpts{
        ID:                  d.Get("id").(string),
        Name:                d.Get("name").(string),
        Status:              d.Get("status").(string),
        CIDR:                d.Get("cidr").(string),
        EnterpriseProjectID: cfg.GetEnterpriseProjectID(d, "all_granted_eps"),
    }

    // 3. 调用 API 查询
    vpcList, err := vpcs.List(v1client, listOpts)
    if err != nil {
        return diag.Errorf("unable to retrieve vpcs: %s", err)
    }

    log.Printf("[DEBUG] retrieved VPC using given filter: %+v", vpcList)

    // 4. 处理查询结果
    var vpcs []map[string]interface{}
    tagFilter := d.Get("tags").(map[string]interface{})
    var ids []string
    
    for _, vpcResource := range vpcList {
        vpc := map[string]interface{}{
            "id":                    vpcResource.ID,
            "name":                  vpcResource.Name,
            "cidr":                  vpcResource.CIDR,
            "enterprise_project_id": vpcResource.EnterpriseProjectID,
            "status":                vpcResource.Status,
            "description":           vpcResource.Description,
        }

        // 处理标签过滤
        if resourceTags, err := tags.Get(v2Client, "vpcs", vpcResource.ID).Extract(); err == nil {
            tagmap := utils.TagsToMap(resourceTags.Tags)

            if !utils.HasMapContains(tagmap, tagFilter) {
                continue
            }
            vpc["tags"] = tagmap
        }

        vpcs = append(vpcs, vpc)
        ids = append(ids, vpcResource.ID)
    }

    // 5. 设置结果
    d.Set("vpcs", vpcs)

    // 6. 设置 Data Source ID
    d.SetId(hashcode.Strings(ids))
    return nil
}
```

### 关键步骤说明

#### 1. 创建客户端

Data Source 可能需要多个客户端来获取完整数据：

```go
v1client, err := cfg.NetworkingV1Client(region)  // VPC v1 API
v2Client, err := cfg.NetworkingV2Client(region)  // 标签 API
v3Client, err := cfg.NewServiceClient("vpcv3", region)  // VPC v3 API
```

#### 2. 构建查询参数

从 Schema 获取查询参数：

```go
listOpts := vpcs.ListOpts{
    ID:                  d.Get("id").(string),
    Name:                d.Get("name").(string),
    Status:              d.Get("status").(string),
    CIDR:                d.Get("cidr").(string),
    EnterpriseProjectID: cfg.GetEnterpriseProjectID(d, "all_granted_eps"),
}
```

#### 3. 处理标签过滤

标签过滤通常需要在客户端进行：

```go
tagFilter := d.Get("tags").(map[string]interface{})

for _, vpcResource := range vpcList {
    if resourceTags, err := tags.Get(v2Client, "vpcs", vpcResource.ID).Extract(); err == nil {
        tagmap := utils.TagsToMap(resourceTags.Tags)

        // 使用 HasMapContains 检查标签匹配
        if !utils.HasMapContains(tagmap, tagFilter) {
            continue
        }
        vpc["tags"] = tagmap
    }
}
```

#### 4. 设置 Data Source ID

Data Source 的 ID 通常基于查询结果生成：

```go
var ids []string
for _, vpcResource := range vpcList {
    ids = append(ids, vpcResource.ID)
}

d.SetId(hashcode.Strings(ids))
```

## 常见 Data Source 模式

### 1. 单资源查询

查询单个资源：

```go
func DataSourceVpc() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceVpcRead,

        Schema: map[string]*schema.Schema{
            "region": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            "id": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            "name": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            // 结果字段直接定义在顶层
            "cidr": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "status": {
                Type:     schema.TypeString,
                Computed: true,
            },
            // ...
        },
    }
}
```

### 2. 列表查询

查询资源列表：

```go
func DataSourceVpcs() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceVpcsRead,

        Schema: map[string]*schema.Schema{
            // 查询参数
            "name": {
                Type:     schema.TypeString,
                Optional: true,
            },
            // 结果字段为列表
            "vpcs": {
                Type:     schema.TypeList,
                Computed: true,
                Elem: &schema.Resource{
                    Schema: vpcSchema(),
                },
            },
        },
    }
}
```

### 3. ID 列表查询

仅返回资源 ID 列表：

```go
func DataSourceVpcIds() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceVpcIdsRead,

        Schema: map[string]*schema.Schema{
            // 查询参数
            "name": {
                Type:     schema.TypeString,
                Optional: true,
            },
            // 结果为 ID 列表
            "ids": {
                Type:     schema.TypeList,
                Computed: true,
                Elem:     &schema.Schema{Type: schema.TypeString},
            },
        },
    }
}
```

### 4. 按标签查询

支持标签过滤：

```go
func DataSourceVpcsByTags() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceVpcsByTagsRead,

        Schema: map[string]*schema.Schema{
            "tags": {
                Type:     schema.TypeMap,
                Required: true,
                Elem:     &schema.Schema{Type: schema.TypeString},
            },
            "vpcs": {
                Type:     schema.TypeList,
                Computed: true,
                Elem: &schema.Resource{
                    Schema: vpcSchema(),
                },
            },
        },
    }
}
```

## 分页处理

### 使用 SDK 分页

```go
func dataSourceVpcsRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    client, err := cfg.NetworkingV1Client(cfg.GetRegion(d))
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }

    listOpts := vpcs.ListOpts{
        // 查询参数
    }

    // 使用 AllPages 获取所有页
    pages, err := vpcs.List(client, listOpts).AllPages()
    if err != nil {
        return diag.Errorf("unable to retrieve vpcs: %s", err)
    }

    vpcList, err := vpcs.ExtractVpcs(pages)
    if err != nil {
        return diag.Errorf("unable to extract vpcs: %s", err)
    }

    // 处理结果...
}
```

### 自定义分页

```go
func listAllResources(client *golangsdk.ServiceClient, opts vpcs.ListOpts) ([]vpcs.Vpc, error) {
    var allVpcs []vpcs.Vpc
    
    pager := vpcs.List(client, opts)
    err := pager.EachPage(func(page pagination.Page) (bool, error) {
        vpcList, err := vpcs.ExtractVpcs(page)
        if err != nil {
            return false, err
        }
        allVpcs = append(allVpcs, vpcList...)
        return true, nil
    })
    
    return allVpcs, err
}
```

## 错误处理

### 处理空结果

```go
vpcList, err := vpcs.List(v1client, listOpts)
if err != nil {
    return diag.Errorf("unable to retrieve vpcs: %s", err)
}

if len(vpcList) == 0 {
    log.Printf("[WARN] No VPCs found with the given filters")
    d.SetId("")
    d.Set("vpcs", []map[string]interface{}{})
    return nil
}
```

### 处理权限错误

```go
if resourceTags, err := tags.Get(v2Client, "vpcs", vpcResource.ID).Extract(); err == nil {
    // 处理标签
} else {
    // 标签 API 不支持企业项目授权，忽略 403 错误
    if _, ok := err.(golangsdk.ErrDefault403); ok {
        log.Printf("[WARN] error query tags of VPC (%s): %s", vpcResource.ID, err)
    } else {
        return diag.Errorf("error query tags of VPC (%s): %s", vpcResource.ID, err)
    }
}
```

## 注册 Data Source 到 Provider

在 `provider.go` 中注册：

```go
DataSourcesMap: map[string]*schema.Resource{
    "huaweicloud_vpc":      vpc.DataSourceVpc(),
    "huaweicloud_vpcs":     vpc.DataSourceVpcs(),
    "huaweicloud_vpc_ids":  vpc.DataSourceVpcIds(),
    // ... 其他 Data Source
},
```

## 开发检查清单

### Schema 设计
- [ ] 查询参数标记为 `Optional`
- [ ] 结果字段标记为 `Computed`
- [ ] 列表结果使用 `TypeList` 或 `TypeSet`
- [ ] 嵌套对象使用 `*schema.Resource`

### Read 函数实现
- [ ] 正确创建客户端
- [ ] 构建查询参数
- [ ] 处理 API 返回结果
- [ ] 实现客户端过滤（如标签）
- [ ] 设置 Data Source ID
- [ ] 设置结果字段

### 错误处理
- [ ] 处理空结果
- [ ] 处理权限错误
- [ ] 记录调试日志

### 性能优化
- [ ] 避免重复 API 调用
- [ ] 合理使用分页
- [ ] 缓存客户端连接

## 最佳实践

### 1. 复用 Schema 定义

```go
func vpcSchema() map[string]*schema.Schema {
    return map[string]*schema.Schema{
        "id": {
            Type:     schema.TypeString,
            Computed: true,
        },
        "name": {
            Type:     schema.TypeString,
            Computed: true,
        },
        // ... 其他字段
    }
}

// 在 Data Source 中复用
"vpcs": {
    Type:     schema.TypeList,
    Computed: true,
    Elem: &schema.Resource{
        Schema: vpcSchema(),
    },
},
```

### 2. 使用辅助函数

```go
func flattenVpc(vpc vpcs.Vpc) map[string]interface{} {
    return map[string]interface{}{
        "id":                    vpc.ID,
        "name":                  vpc.Name,
        "cidr":                  vpc.CIDR,
        "enterprise_project_id": vpc.EnterpriseProjectID,
        "status":                vpc.Status,
        "description":           vpc.Description,
    }
}
```

### 3. 日志记录

```go
log.Printf("[DEBUG] retrieved VPC using given filter: %+v", vpcList)
log.Printf("[DEBUG] VPC List after filter, count: %d vpcs: %+v", len(vpcs), vpcs)
```
