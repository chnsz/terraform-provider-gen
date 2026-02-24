# Resource 编码开发技能

本文档以 `services/vpc/resource_huaweicloud_vpc.go` 为例，详细介绍 Resource 的开发技能。

## Resource 基本结构

### 1. Resource 定义函数

Resource 定义函数返回一个 `*schema.Resource` 对象，包含资源的完整定义：

```go
// @API VPC POST /v1/{project_id}/vpcs
// @API VPC GET /v1/{project_id}/vpcs/{id}
// @API VPC PUT /v1/{project_id}/vpcs/{id}
// @API VPC DELETE /v1/{project_id}/vpcs/{id}
func ResourceVirtualPrivateCloudV1() *schema.Resource {
    return &schema.Resource{
        CreateContext: resourceVirtualPrivateCloudCreate,
        ReadContext:   resourceVirtualPrivateCloudRead,
        UpdateContext: resourceVirtualPrivateCloudUpdate,
        DeleteContext: resourceVirtualPrivateCloudDelete,
        
        Importer: &schema.ResourceImporter{
            StateContext: schema.ImportStatePassthroughContext,
        },

        Timeouts: &schema.ResourceTimeout{
            Create: schema.DefaultTimeout(10 * time.Minute),
            Delete: schema.DefaultTimeout(3 * time.Minute),
        },

        CustomizeDiff: config.MergeDefaultTags(),

        Schema: map[string]*schema.Schema{
            // Schema 定义
        },
    }
}
```

### 2. 核心组件说明

| 组件 | 说明 |
|------|------|
| `CreateContext` | 创建资源函数 |
| `ReadContext` | 读取资源状态函数 |
| `UpdateContext` | 更新资源函数 |
| `DeleteContext` | 删除资源函数 |
| `Importer` | 资源导入配置 |
| `Timeouts` | 超时配置 |
| `CustomizeDiff` | 自定义差异检测 |
| `Schema` | 资源 Schema 定义 |

## Schema 定义

### 基本字段类型

```go
Schema: map[string]*schema.Schema{
    "region": {
        Type:     schema.TypeString,
        Optional: true,
        ForceNew: true,
        Computed: true,
    },
    "name": {
        Type:     schema.TypeString,
        Required: true,
    },
    "cidr": {
        Type:         schema.TypeString,
        Required:     true,
        ValidateFunc: utils.ValidateCIDR,
    },
    "description": {
        Type:     schema.TypeString,
        Optional: true,
    },
    "status": {
        Type:     schema.TypeString,
        Computed: true,
    },
    "tags": common.TagsSchema(),
},
```

### Schema 字段属性

| 属性 | 说明 |
|------|------|
| `Type` | 字段类型（String, Int, Bool, List, Set, Map） |
| `Required` | 必填字段 |
| `Optional` | 可选字段 |
| `Computed` | 计算字段（由 API 返回） |
| `ForceNew` | 修改时重建资源 |
| `Default` | 默认值 |
| `ValidateFunc` | 验证函数 |
| `ConflictsWith` | 冲突字段列表 |
| `AtLeastOneOf` | 至少填写其中一个 |
| `ExactlyOneOf` | 必须且只能填写其中一个 |
| `Deprecated` | 废弃提示信息 |

### 常用 Schema 类型

#### 1. 字符串类型

```go
"name": {
    Type:     schema.TypeString,
    Required: true,
    ValidateFunc: validation.StringLenBetween(1, 64),
},
```

#### 2. 列表类型

```go
"secondary_cidrs": {
    Type:     schema.TypeSet,
    Optional: true,
    Computed: true,
    Elem: &schema.Schema{
        Type:         schema.TypeString,
        ValidateFunc: utils.ValidateCIDR,
    },
},
```

#### 3. 嵌套对象类型

```go
"routes": {
    Type:     schema.TypeList,
    Computed: true,
    Elem: &schema.Resource{
        Schema: map[string]*schema.Schema{
            "destination": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "nexthop": {
                Type:     schema.TypeString,
                Computed: true,
            },
        },
    },
},
```

#### 4. 标签类型

```go
"tags": common.TagsSchema(),
```

## CRUD 函数实现

### 1. Create 函数

Create 函数负责创建资源并设置资源 ID：

```go
func resourceVirtualPrivateCloudCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    region := cfg.GetRegion(d)
    
    // 1. 创建客户端
    v1Client, err := cfg.NetworkingV1Client(region)
    if err != nil {
        return diag.Errorf("error creating VPC v1 client: %s", err)
    }

    // 2. 构建创建参数
    createOpts := vpcs.CreateOpts{
        Name:        d.Get("name").(string),
        CIDR:        d.Get("cidr").(string),
        Description: d.Get("description").(string),
    }

    // 3. 调用 API 创建资源
    n, err := vpcs.Create(v1Client, createOpts).Extract()
    if err != nil {
        return diag.Errorf("error creating VPC: %s", err)
    }

    // 4. 设置资源 ID
    d.SetId(n.ID)
    log.Printf("[DEBUG] VPC ID: %s", n.ID)

    // 5. 等待资源就绪
    stateConf := &resource.StateChangeConf{
        Pending:    []string{"CREATING"},
        Target:     []string{"ACTIVE"},
        Refresh:    waitForVpcActive(v1Client, n.ID),
        Timeout:    d.Timeout(schema.TimeoutCreate),
        Delay:      5 * time.Second,
        MinTimeout: 3 * time.Second,
    }

    _, stateErr := stateConf.WaitForStateContext(ctx)
    if stateErr != nil {
        return diag.Errorf("error waiting for Vpc (%s) to become ACTIVE: %s", n.ID, stateErr)
    }

    // 6. 设置标签等附加操作
    tagRaw := d.Get("tags").(map[string]interface{})
    if len(tagRaw) > 0 {
        v2Client, err := cfg.NetworkingV2Client(region)
        if err != nil {
            return diag.Errorf("error creating VPC v2 client: %s", err)
        }
        taglist := utils.ExpandResourceTags(tagRaw)
        if tagErr := tags.Create(v2Client, "vpcs", n.ID, taglist).ExtractErr(); tagErr != nil {
            return diag.Errorf("error setting tags of VPC %q: %s", n.ID, tagErr)
        }
    }

    // 7. 调用 Read 函数刷新状态
    return resourceVirtualPrivateCloudRead(ctx, d, meta)
}
```

### 2. Read 函数

Read 函数负责从 API 获取资源状态并更新 Terraform state：

```go
func resourceVirtualPrivateCloudRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    region := cfg.GetRegion(d)
    
    // 1. 获取资源详情
    n, err := GetVpcById(cfg, region, d.Id())
    if err != nil {
        return common.CheckDeletedDiag(d, err, "error obtain VPC information")
    }

    // 2. 设置基本属性
    d.Set("name", n.Name)
    d.Set("cidr", n.CIDR)
    d.Set("description", n.Description)
    d.Set("status", n.Status)
    d.Set("region", region)

    // 3. 设置复杂属性（列表、嵌套对象）
    routes := make([]map[string]interface{}, len(n.Routes))
    for i, rtb := range n.Routes {
        route := map[string]interface{}{
            "destination": rtb.DestinationCIDR,
            "nexthop":     rtb.NextHop,
        }
        routes[i] = route
    }
    d.Set("routes", routes)

    // 4. 获取并设置标签
    v2Client, err := cfg.NetworkingV2Client(region)
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }
    if resourceTags, err := tags.Get(v2Client, "vpcs", d.Id()).Extract(); err == nil {
        tagmap := utils.TagsToMap(resourceTags.Tags)
        if err := d.Set("tags", tagmap); err != nil {
            return diag.Errorf("error saving tags to state for VPC (%s): %s", d.Id(), err)
        }
    }

    return nil
}
```

### 3. Update 函数

Update 函数处理资源更新：

```go
func resourceVirtualPrivateCloudUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    region := cfg.GetRegion(d)
    v1Client, err := cfg.NetworkingV1Client(region)
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }

    vpcID := d.Id()
    
    // 1. 检测变更并更新基本属性
    if d.HasChanges("name", "cidr", "description") {
        updateOpts := vpcs.UpdateOpts{
            Name: d.Get("name").(string),
            CIDR: d.Get("cidr").(string),
        }
        if d.HasChange("description") {
            desc := d.Get("description").(string)
            updateOpts.Description = &desc
        }

        _, err = vpcs.Update(v1Client, vpcID, updateOpts).Extract()
        if err != nil {
            return diag.Errorf("error updating VPC: %s", err)
        }
    }

    // 2. 更新标签
    if d.HasChange("tags") {
        v2Client, err := cfg.NetworkingV2Client(region)
        if err != nil {
            return diag.Errorf("error creating VPC v2 client: %s", err)
        }

        tagErr := utils.UpdateResourceTags(v2Client, d, "vpcs", vpcID)
        if tagErr != nil {
            return diag.Errorf("error updating tags of VPC %s: %s", vpcID, tagErr)
        }
    }

    // 3. 更新企业项目
    if d.HasChange("enterprise_project_id") {
        migrateOpts := config.MigrateResourceOpts{
            ResourceId:   vpcID,
            ResourceType: "vpcs",
            RegionId:     region,
            ProjectId:    v1Client.ProjectID,
        }
        if err := cfg.MigrateEnterpriseProject(ctx, d, migrateOpts); err != nil {
            return diag.FromErr(err)
        }
    }

    // 4. 调用 Read 函数刷新状态
    return resourceVirtualPrivateCloudRead(ctx, d, meta)
}
```

### 4. Delete 函数

Delete 函数处理资源删除：

```go
func resourceVirtualPrivateCloudDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    conf := meta.(*config.Config)
    v1Client, err := conf.NetworkingV1Client(conf.GetRegion(d))
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }

    // 使用 StateChangeConf 等待删除完成
    stateConf := &resource.StateChangeConf{
        Pending:    []string{"ACTIVE"},
        Target:     []string{"DELETED"},
        Refresh:    waitForVpcDelete(v1Client, d.Id()),
        Timeout:    d.Timeout(schema.TimeoutDelete),
        Delay:      5 * time.Second,
        MinTimeout: 3 * time.Second,
    }

    _, err = stateConf.WaitForStateContext(ctx)
    if err != nil {
        return diag.Errorf("error deleting VPC %s: %s", d.Id(), err)
    }

    return nil
}
```

## 状态等待函数

### 等待资源就绪

```go
func waitForVpcActive(vpcClient *golangsdk.ServiceClient, vpcId string) resource.StateRefreshFunc {
    return func() (interface{}, string, error) {
        n, err := vpcs.Get(vpcClient, vpcId).Extract()
        if err != nil {
            return nil, "", err
        }

        if n.Status == "OK" {
            return n, "ACTIVE", nil
        }

        if n.Status == "DOWN" {
            return nil, "", fmt.Errorf("VPC status: '%s'", n.Status)
        }

        return n, n.Status, nil
    }
}
```

### 等待资源删除

```go
func waitForVpcDelete(vpcClient *golangsdk.ServiceClient, vpcId string) resource.StateRefreshFunc {
    return func() (interface{}, string, error) {
        r, err := vpcs.Get(vpcClient, vpcId).Extract()
        if err != nil {
            if _, ok := err.(golangsdk.ErrDefault404); ok {
                log.Printf("[INFO] successfully delete VPC %s", vpcId)
                return r, "DELETED", nil
            }
            return r, "ACTIVE", err
        }

        err = vpcs.Delete(vpcClient, vpcId).ExtractErr()
        if err != nil {
            if _, ok := err.(golangsdk.ErrDefault404); ok {
                log.Printf("[INFO] successfully delete VPC %s", vpcId)
                return r, "DELETED", nil
            }
            if _, ok := err.(golangsdk.ErrDefault409); ok {
                return r, "ACTIVE", nil
            }
            return r, "ACTIVE", err
        }

        return r, "ACTIVE", nil
    }
}
```

## 资源导入

### 基本导入

```go
Importer: &schema.ResourceImporter{
    StateContext: schema.ImportStatePassthroughContext,
},
```

### 自定义导入逻辑

```go
func resourceVpcRTBRouteImportState(_ context.Context, d *schema.ResourceData, _ interface{}) ([]*schema.ResourceData, error) {
    routeID, _ := parseResourceID(d.Id())
    if routeID == "" {
        return nil, fmt.Errorf("invalid format specified for import id, must be <route_table_id>/<destination>")
    }

    return []*schema.ResourceData{d}, nil
}
```

## HTTP 请求封装

对于 SDK 未提供的 API，可以使用通用 HTTP 请求方法：

```go
func addSecondaryCIDR(client *golangsdk.ServiceClient, vpcID string, cidrs []string) error {
    addSecondaryCIDRHttpUrl := "v3/{project_id}/vpc/vpcs/{vpc_id}/add-extend-cidr"
    addSecondaryCIDRPath := client.Endpoint + addSecondaryCIDRHttpUrl
    addSecondaryCIDRPath = strings.ReplaceAll(addSecondaryCIDRPath, "{project_id}", client.ProjectID)
    addSecondaryCIDRPath = strings.ReplaceAll(addSecondaryCIDRPath, "{vpc_id}", vpcID)

    addSecondaryCIDROpt := golangsdk.RequestOpts{
        KeepResponseBody: true,
    }
    addSecondaryCIDROpt.JSONBody = utils.RemoveNil(buildSecondaryCIDRBodyParams(cidrs))

    log.Printf("[DEBUG] add secondary CIDRs %s into VPC %s", cidrs, vpcID)
    _, err := client.Request("PUT", addSecondaryCIDRPath, &addSecondaryCIDROpt)
    return err
}

func buildSecondaryCIDRBodyParams(cidrs []string) map[string]interface{} {
    return map[string]interface{}{
        "vpc": map[string]interface{}{
            "extend_cidrs": cidrs,
        },
    }
}
```

## 辅助函数

### 获取资源详情

```go
func GetVpcById(conf *config.Config, region, vpcId string) (*vpcs.Vpc, error) {
    v1Client, err := conf.NetworkingV1Client(region)
    if err != nil {
        return nil, err
    }

    return vpcs.Get(v1Client, vpcId).Extract()
}
```

### 解析资源 ID

```go
func parseResourceID(id string) (tableID, destination string) {
    parts := strings.SplitN(id, "/", 2)
    if len(parts) != 2 {
        return
    }

    tableID, destination = parts[0], parts[1]
    return
}
```

## 注册资源到 Provider

在 `provider.go` 中注册资源：

```go
ResourcesMap: map[string]*schema.Resource{
    "huaweicloud_vpc": vpc.ResourceVirtualPrivateCloudV1(),
    // ... 其他资源
},
```

## 开发检查清单

### Schema 设计
- [ ] 所有必需字段标记为 `Required`
- [ ] 计算字段标记为 `Computed`
- [ ] 需要重建的字段标记为 `ForceNew`
- [ ] 添加适当的验证函数
- [ ] 处理字段冲突关系

### CRUD 实现
- [ ] Create 函数设置资源 ID
- [ ] Read 函数处理 404 错误
- [ ] Update 函数使用 `HasChange` 检测变更
- [ ] Delete 函数等待删除完成
- [ ] 所有函数返回 `diag.Diagnostics`

### 错误处理
- [ ] 使用 `CheckDeletedDiag` 处理资源不存在
- [ ] 提供有意义的错误信息
- [ ] 记录调试日志

### 其他
- [ ] 配置适当的超时时间
- [ ] 支持资源导入
- [ ] 处理企业项目迁移
- [ ] 支持标签管理
