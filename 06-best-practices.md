# 最佳实践开发技能

本文档介绍 terraform-provider-huaweicloud 项目开发的最佳实践，包括 API 调用、错误处理、状态等待、编码规范等。

## API 客户端创建

### 获取客户端

```go
cfg := meta.(*config.Config)
region := cfg.GetRegion(d)

// 创建 VPC v1 客户端
v1Client, err := cfg.NetworkingV1Client(region)
if err != nil {
    return diag.Errorf("error creating VPC v1 client: %s", err)
}

// 创建 VPC v2 客户端（用于标签操作）
v2Client, err := cfg.NetworkingV2Client(region)
if err != nil {
    return diag.Errorf("error creating VPC v2 client: %s", err)
}

// 创建 VPC v3 客户端（用于扩展 CIDR 操作）
v3Client, err := cfg.NewServiceClient("vpcv3", region)
if err != nil {
    return diag.Errorf("error creating VPC v3 client: %s", err)
}
```

### 常用客户端类型

| 服务 | 客户端方法 | 用途 |
|------|-----------|------|
| VPC v1 | `NetworkingV1Client()` | VPC、子网、路由表 |
| VPC v2 | `NetworkingV2Client()` | 标签、端口 |
| VPC v3 | `NewServiceClient("vpcv3")` | 扩展 CIDR |
| ECS | `ComputeV1Client()` | 云服务器 |
| EVS | `BlockStorageV3Client()` | 云硬盘 |
| RDS | `RdsV3Client()` | 关系型数据库 |
| IAM | `IdentityV3Client()` | 身份认证 |

## 错误处理

### CheckDeletedDiag - 处理资源不存在

```go
func resourceVirtualPrivateCloudRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    cfg := meta.(*config.Config)
    region := cfg.GetRegion(d)
    n, err := GetVpcById(cfg, region, d.Id())
    if err != nil {
        return common.CheckDeletedDiag(d, err, "error obtain VPC information")
    }
    // ...
}
```

`CheckDeletedDiag` 会自动处理 404 错误，将资源从 state 中移除。

### 错误类型判断

```go
import "github.com/chnsz/golangsdk"

// 判断是否为 404 错误
if _, ok := err.(golangsdk.ErrDefault404); ok {
    d.SetId("")
    return nil
}

// 判断是否为 403 错误
if _, ok := err.(golangsdk.ErrDefault403); ok {
    log.Printf("[WARN] permission denied: %s", err)
}

// 判断是否为 409 冲突错误
if _, ok := err.(golangsdk.ErrDefault409); ok {
    // 资源正在被操作，可以重试
}
```

### 错误信息格式

```go
// 好的错误信息
return diag.Errorf("error creating VPC: %s", err)
return diag.Errorf("error updating tags of VPC %s: %s", vpcID, err)

// 不好的错误信息
return diag.Errorf("create failed: %v", err)
return diag.FromErr(err)
```

## 状态等待 (WaitForState)

### StateChangeConf 配置

```go
stateConf := &resource.StateChangeConf{
    Pending:    []string{"CREATING", "PENDING"},
    Target:     []string{"ACTIVE", "OK"},
    Refresh:    waitForResourceActive(client, resourceID),
    Timeout:    d.Timeout(schema.TimeoutCreate),
    Delay:      5 * time.Second,
    MinTimeout: 3 * time.Second,
}

result, err := stateConf.WaitForStateContext(ctx)
if err != nil {
    return diag.Errorf("error waiting for resource to become active: %s", err)
}
```

### 状态刷新函数

```go
func waitForVpcActive(vpcClient *golangsdk.ServiceClient, vpcId string) resource.StateRefreshFunc {
    return func() (interface{}, string, error) {
        n, err := vpcs.Get(vpcClient, vpcId).Extract()
        if err != nil {
            return nil, "", err
        }

        // 成功状态
        if n.Status == "OK" {
            return n, "ACTIVE", nil
        }

        // 失败状态
        if n.Status == "DOWN" || n.Status == "ERROR" {
            return nil, "", fmt.Errorf("VPC status: '%s'", n.Status)
        }

        // 中间状态
        return n, n.Status, nil
    }
}
```

### 删除等待函数

```go
func waitForVpcDelete(vpcClient *golangsdk.ServiceClient, vpcId string) resource.StateRefreshFunc {
    return func() (interface{}, string, error) {
        r, err := vpcs.Get(vpcClient, vpcId).Extract()
        if err != nil {
            // 404 表示资源已删除
            if _, ok := err.(golangsdk.ErrDefault404); ok {
                log.Printf("[INFO] successfully delete VPC %s", vpcId)
                return r, "DELETED", nil
            }
            return r, "ACTIVE", err
        }

        // 执行删除
        err = vpcs.Delete(vpcClient, vpcId).ExtractErr()
        if err != nil {
            if _, ok := err.(golangsdk.ErrDefault404); ok {
                return r, "DELETED", nil
            }
            // 409 表示资源正在被使用，继续等待
            if _, ok := err.(golangsdk.ErrDefault409); ok {
                return r, "ACTIVE", nil
            }
            return r, "ACTIVE", err
        }

        return r, "ACTIVE", nil
    }
}
```

### 超时配置

```go
Timeouts: &schema.ResourceTimeout{
    Create: schema.DefaultTimeout(10 * time.Minute),
    Update: schema.DefaultTimeout(10 * time.Minute),
    Delete: schema.DefaultTimeout(3 * time.Minute),
},
```

## HTTP 请求封装

### 自定义 API 请求

当 SDK 未提供 API 时，使用通用请求方法：

```go
func addSecondaryCIDR(client *golangsdk.ServiceClient, vpcID string, cidrs []string) error {
    httpUrl := "v3/{project_id}/vpc/vpcs/{vpc_id}/add-extend-cidr"
    path := client.Endpoint + httpUrl
    path = strings.ReplaceAll(path, "{project_id}", client.ProjectID)
    path = strings.ReplaceAll(path, "{vpc_id}", vpcID)

    opts := golangsdk.RequestOpts{
        KeepResponseBody: true,
    }
    opts.JSONBody = map[string]interface{}{
        "vpc": map[string]interface{}{
            "extend_cidrs": cidrs,
        },
    }

    log.Printf("[DEBUG] add secondary CIDRs %s into VPC %s", cidrs, vpcID)
    _, err := client.Request("PUT", path, &opts)
    return err
}
```

### 解析响应

```go
func obtainV3VpcResp(client *golangsdk.ServiceClient, vpcID string) (interface{}, error) {
    httpUrl := "v3/{project_id}/vpc/vpcs/{vpc_id}"
    path := client.Endpoint + httpUrl
    path = strings.ReplaceAll(path, "{project_id}", client.ProjectID)
    path = strings.ReplaceAll(path, "{vpc_id}", vpcID)

    opts := golangsdk.RequestOpts{
        KeepResponseBody: true,
    }
    resp, err := client.Request("GET", path, &opts)
    if err != nil {
        return nil, err
    }

    return utils.FlattenResponse(resp)
}
```

### 使用 JMESPath 提取数据

```go
res, err := obtainV3VpcResp(client, vpcID)
if err != nil {
    return diag.Errorf("error retrieving VPC: %s", err)
}

// 使用 PathSearch 提取字段
extendCidrs := utils.PathSearch("vpc.extend_cidrs", res, []interface{}{}).([]interface{})
d.Set("secondary_cidrs", extendCidrs)
```

## 标签管理

### 创建标签

```go
tagRaw := d.Get("tags").(map[string]interface{})
if len(tagRaw) > 0 {
    taglist := utils.ExpandResourceTags(tagRaw)
    if err := tags.Create(client, "vpcs", resourceID, taglist).ExtractErr(); err != nil {
        return diag.Errorf("error setting tags: %s", err)
    }
}
```

### 读取标签

```go
if resourceTags, err := tags.Get(client, "vpcs", d.Id()).Extract(); err == nil {
    tagmap := utils.TagsToMap(resourceTags.Tags)
    if err := d.Set("tags", tagmap); err != nil {
        return diag.Errorf("error saving tags: %s", err)
    }
}
```

### 更新标签

```go
if d.HasChange("tags") {
    if err := utils.UpdateResourceTags(client, d, "vpcs", vpcID); err != nil {
        return diag.Errorf("error updating tags: %s", err)
    }
}
```

## 企业项目管理

### 创建时设置企业项目

```go
epsID := cfg.GetEnterpriseProjectID(d)
if epsID != "" {
    createOpts.EnterpriseProjectID = epsID
}
```

### 迁移企业项目

```go
if d.HasChange("enterprise_project_id") {
    migrateOpts := config.MigrateResourceOpts{
        ResourceId:   vpcID,
        ResourceType: "vpcs",
        RegionId:     region,
        ProjectId:    client.ProjectID,
    }
    if err := cfg.MigrateEnterpriseProject(ctx, d, migrateOpts); err != nil {
        return diag.FromErr(err)
    }
}
```

## 数据转换工具

### 类型转换

```go
// 接口数组转字符串数组
names := utils.ExpandToStringList(d.Get("names").([]interface{}))

// 接口数组转整数数组
ports := utils.ExpandToIntList(d.Get("ports").([]interface{}))

// 接口映射转字符串映射
tags := utils.ExpandToStringMap(d.Get("tags").(map[string]interface{}))
```

### 移除 nil 值

```go
body := utils.RemoveNil(map[string]interface{}{
    "name":        d.Get("name").(string),
    "description": d.Get("description").(string),
    "tags":        d.Get("tags").(map[string]interface{}),
})
```

### 标签过滤

```go
tagFilter := d.Get("tags").(map[string]interface{})
tagmap := utils.TagsToMap(resourceTags.Tags)

if !utils.HasMapContains(tagmap, tagFilter) {
    continue
}
```

## 并发控制

### 使用互斥锁

```go
import "github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/helper/mutexkv"

// 全局互斥锁
var MutexKV = mutexkv.NewMutexKV()

// 锁定资源
MutexKV.Lock(resourceID)
defer MutexKV.Unlock(resourceID)
```

## 日志记录

### 日志级别

```go
log.Printf("[DEBUG] VPC ID: %s", n.ID)
log.Printf("[INFO] successfully delete VPC %s", vpcId)
log.Printf("[WARN] error query tags of VPC (%s): %s", vpcID, err)
log.Printf("[ERROR] failed to create VPC: %s", err)
```

### 调试日志位置

- 函数入口：记录参数
- API 调用前：记录请求参数
- API 调用后：记录响应结果
- 错误处理：记录错误详情

## 编码规范

### 函数命名

```go
// Resource 函数
func resource<ResourceName>Create()
func resource<ResourceName>Read()
func resource<ResourceName>Update()
func resource<ResourceName>Delete()

// Data Source 函数
func dataSource<DataSourcesName>Read()

// 辅助函数
func waitFor<ResourceName>Active()
func parse<ResourceName>ID()
```

### 错误返回

```go
// 使用 diag.Diagnostics
func resourceCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
    return diag.Errorf("error message: %s", err)
}

// 使用 error（辅助函数）
func getResourceByID(id string) (*Resource, error) {
    // ...
    return nil, fmt.Errorf("error message: %s", err)
}
```

### Context 使用

```go
// CRUD 函数必须支持 Context
func resourceCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 使用 context 的等待函数
    _, err := stateConf.WaitForStateContext(ctx)
    // ...
}
```

## 包导入顺序

```go
import (
    // 标准库
    "context"
    "fmt"
    "log"
    "time"

    // Terraform SDK
    "github.com/hashicorp/terraform-plugin-sdk/v2/diag"
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/resource"
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"

    // 华为云 SDK
    "github.com/chnsz/golangsdk"
    "github.com/chnsz/golangsdk/openstack/networking/v1/vpcs"

    // 项目内部包
    "github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/common"
    "github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/config"
    "github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/utils"
)
```

## 开发检查清单

### API 调用
- [ ] 正确创建客户端
- [ ] 处理客户端创建错误
- [ ] 使用正确的 API 版本

### 错误处理
- [ ] 使用 CheckDeletedDiag 处理 404
- [ ] 提供有意义的错误信息
- [ ] 记录适当的日志

### 状态管理
- [ ] 配置合理的超时时间
- [ ] 正确处理中间状态
- [ ] 处理失败状态

### 代码质量
- [ ] 遵循命名规范
- [ ] 添加适当的日志
- [ ] 处理所有错误
- [ ] 使用 Context
