# 添加更新resource相关函数，生成创建资源所需方法

## Step 1 生成更新资源函数

- 生成一个创建资源的函数，格式为`resource{XXX}Update`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationUpdate`
- 该函数总共包含：函数名称、参数定义、创建client、更新参数方法、更新enterprise_project_id、更新tags、更新auto_renew

```go
func resourcePublicationUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 参数定义

	// 创建client

    // 更新参数
	
	// 更新enterprise_project_id
	
	// 更新tags
	
	// 更新auto_renew

	return resourcePublicationRead(ctx, d, meta)
}
```

## Step 2 参数定义，创建client

- 创建client时，其中第一个参数为服务类型，错误信息中的服务类型需要大写
- 如果参数中包含包周期参数，那么就需要创建更新包周期参数所需的client，变量名为`bssClient`

```go
cfg := meta.(*config.Config)
region := cfg.GetRegion(d)
var (
	product = "rds"
)

client, err := cfg.NewServiceClient("rds", region)
if err != nil {
	return diag.Errorf("error creating RDS client: %s", err)
}

bssClient, err := cfg.BssV2Client(region)
if err != nil {
	return diag.Errorf("error creating bss V2 client: %s", err)
}
```

## Step 2 更新参数

- 根据用户提供的更新resource参数API，依次添加参数更新方法

## Step 2.1 获取更新resource参数的数据

- 调用用户提供的查询资源的API，从结果中获取请求参数
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容

## Step 2.2 生成更新参数逻辑

- 如果API请求参数只有一个，那么就使用`d.HasChange`方法判断参数是否修改
- 如果API请求参数有多个，那么就使用`d.HasChanges`方法判断参数是否修改
- 更新参数通过`update{XXX}{YYY}`方法实现，其中 `{XXX}` 为该资源的功能信息，`{YYY}` 为参数名称，函数名最终采用驼峰格式
- 函数参数首先需要包含`d`和`client`
- 如果API响应消息中包含`job_id`等任务信息，那么函数参数需要包含`ctx`
- 如果API请求消息中包含包周期相关参数，例如`自动支付（is_auto_pay）`，或者API响应消息中包含`order_id`，那么函数参数需要包含`bssClient`
- 最后判断返回错误信息是否为空，如果为空，就要return错误信息，使用`diag.FromErr`处理错误信息

```go
if d.HasChange("name") {
	if err = updateInstanceName(d, client); err != nil {
		return diag.FromErr(err)
	}
}
```

```go
if d.HasChanges("kms_key_id", "description", "event_subscriptions") {
	if err := updateInstance(ctx, d, client); err != nil {
		return diag.FromErr(err)
	}
}
```

```go
if d.HasChange("flavor") {
	if err = updateInstanceFlavor(ctx, d, client, bssClient); err != nil {
		return diag.FromErr(err)
	}
}
```

## Step 3 更新enterprise_project_id

- 如果请求`创建资源API`参数中包含参数 `enterprise_project_id`，那么就添加`更新enterprise_project_id`
- 使用`cfg.MigrateEnterpriseProject`方法去更新`enterprise_project_id`，如果返回`err`不为空，则返回错误信息
- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置）：`// @API EPS POST /v1.0/enterprise-projects/{enterprise_project_id}/resources-migrat`

```go
if d.HasChange("enterprise_project_id") {
	migrateOpts := config.MigrateResourceOpts {
		ResourceId:   instanceID,
		ResourceType: "rds",
		RegionId:     region,
		ProjectId:    client.ProjectID,}
	if err = cfg.MigrateEnterpriseProject(ctx, d, migrateOpts); err != nil {
		return diag.FromErr(err)
	}
}
```

## Step 4 更新tags

- 如果`创建资源API`参数中包含 `tags`，那么就添加`更新tags`
- 使用`utils.UpdateResourceTags`方法去更新`tags`
- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置）：`// @API RDS POST /v3/{project_id}/instances/{id}/tags/action`

```go
if d.HasChange("tags") {
	if err = utils.UpdateResourceTags(client, d, "instances", instanceID); err != nil {
		return diag.Errorf("error updating tags of RDS instance (%s): %s", instanceID, err)
	}
}
```

## Step 5 更新auto_renew

- 如果`创建资源API`参数中包含 `auto_renew`，那么就添加`更新auto_renew`
- 使用`common.UpdateAutoRenew`方法去更新`auto_renew`
- 在`resource函数`前边加两个注释（示例中的`使用到的API`位置）：`// @API BSS POST /v2/orders/subscriptions/resources/autorenew/{instance_id}` 和 `// @API BSS DELETE /v2/orders/subscriptions/resources/autorenew/{instance_id}`

```go
if d.HasChange("auto_renew") {
	if err = common.UpdateAutoRenew(bssClient, d.Get("auto_renew").(string), instanceID); err != nil {
		return diag.Errorf("error updating the auto-renew of the instance (%s): %s", instanceID, err)
	}
}
```

## Step 6 生成更新参数函数

- 参考 ./update_param_function.md
- 严格按照文档中的步骤执行
- 不要随意添加其他参数，或者是删除参数

## Step 7 添加不支持更新的参数列表

- 如果创建API的参数（包含URI参数）不被更新API支持更新，那么该参数不支持更新，那么就在`resource函数`中添加不支持修改的参数列表（示例中的`不支持修改的参数列表`位置），类型为字符串数组
- 如果参数类型为list，那么就需要使用星号（`*`）来分开层级

```go
var rdsInstanceNonUpdatableParams = []string{
	"name",
	"images",
	"building_config.*.cluster", "building_config.*.image_pull_secrets",
}
```

## Step 8 添加 CustomizeDiff 信息

- 如果`Step 7`添加了不支持修改的参数列表：
  - 那么就在资源函数中添加CustomizeDiff 信息（示例中的`CustomizeDiff 信息`位置）
  - `resource函数`中添加`enable_force_new`参数

  ```go
  CustomizeDiff: config.FlexibleForceNew(rdsInstanceNonUpdatableParams),
  ```
  
  ```go
  "enable_force_new": {
  	Type:         schema.TypeString, 
  	Optional:     true,
  	ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false), 
  	Description:  utils.SchemaDesc("", utils.SchemaDescInput{Internal: true}),
  },
  ```