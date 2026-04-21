# 生成 CreateContext 函数流程

## Step 1 生成查询资源的函数

- 格式为`resource{XXX}Create`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationCreate`
- 该函数总共包含：函数名称、参数定义、创建client、构造请求参数、构造请求体、发送请求、解析结果、设置资源id、等待任务或订单完成

```go
func resourcePublicationCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 参数定义
    
    // 创建client
    
    // 构造请求参数和请求体
    
    // 发送请求
    
    // 解析结果
    
    // 设置资源id
    
    // 等待任务或订单完成
	
	// 更新其他参数

    return resourcePublicationRead(ctx, d, meta)
}
```

## Step 2 参数定义，创建client

- 创建client时，其中第一个参数为服务类型，错误信息中的服务类型需要大写

```go
cfg := meta.(*config.Config)
region := cfg.GetRegion(d)
var (
	httpUrl = "v3/{project_id}/instances/{instance_id}/replication/publications"
	product = "rds"
)

client, err := cfg.NewServiceClient(product, region)
if err != nil {
	return diag.Errorf("error creating RDS client: %s", err)
}
```

## Step 3 构造请求参数和请求体

- 将path参数替换为具体的值，禁止使用`utils.ReplacePathVariables`，使用`strings.ReplaceAll`
- 构造请求体，函数名为 `buildCreate{XXX}BodyParams`， 其中 `{XXX}` 为该资源的功能信息，最后使用`utils.RemoveNil`去除掉值为nil的参数
- 需要引入包`github.com/chnsz/golangsdk`

```go
createPath := client.Endpoint + httpUrl
createPath = strings.ReplaceAll(createPath, "{project_id}", client.ProjectID)
createPath = strings.ReplaceAll(createPath, "{instance_id}", d.Get("instance_id").(string))

createOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	MoreHeaders: map[string]string{"Content-Type": "application/json"},
}
createOpt.JSONBody = utils.RemoveNil(buildCreatePublicationBodyParams(d))
```

## Step 4 发送请求、解析结果

```go
createResp, err := client.Request("POST", getPath, &getOpt)
if err != nil {
	return diag.Errorf("error creating RDS publication: %s", err)
}

createRespBody, err := utils.FlattenResponse(createResp)
if err != nil {
	return diag.FromErr(err)
}
```

## Step 5 设置资源id

1. 如果用户说明了获取资源`id`的位置，那么就从用户指定的位置获取`id`
2. 如果用户没有说明获取`id`的位置，那么就直接从API响应消息中查找`id`

```go
id := utils.PathSearch("id", createRespBody, "").(string)
if id == "" {
	return diag.Errorf("error creating RDS publication: ID is not found in API response")
}
d.SetId(id)
```

## Step 6 等待任务或订单完成

### Step 6.1 添加等待订单完成逻辑

- 从API响应消息中获取订单ID信息，如`order_id`或`order_info`，如果存在，那么就等待订单的完成
- 如果API支持包周期，那么在resource函数前边加一个注释（示例中的`使用到的API`位置）：`@API BSS GET /v2/orders/customer-orders/details/{order_id}`

```go
orderId := utils.PathSearch("order_id", createRespBody, "").(string)
if orderId != "" {
	bssClient, err := cfg.BssV2Client(region)
	if err != nil {
		return diag.Errorf("error creating BSS v2 client: %s", err)
	}
	err = common.WaitOrderComplete(ctx, bssClient, orderId, d.Timeout(schema.TimeoutCreate))
	if err != nil {
		return diag.FromErr(err)
	}
}
```

### Step 6.2 添加等待任务完成逻辑

- 从API响应消息中获取任务ID信息，如果存在，那么就等待任务的完成
- 函数名为 `check{XXX}JobFinish`， 其中 `{XXX}` 为该资源的功能信息

```go
jobId := utils.PathSearch("job_id", createRespBody, "").(string)
if jobId != "" {
	if err = checkInstanceJobFinish(client, jobId, d.Timeout(schema.TimeoutCreate)); err != nil {
		return diag.Errorf("error creating publication: %s", err)
	}
}
```

## Step 7 更新其他参数

- 如果更新API需要在创建时触发，那么就添加更新逻辑，函数名为`update{XXX}{YYY}`，其中 `{XXX}` 为该资源的功能信息，`{YYY}` 为参数名称，函数名最终采用驼峰格式

```go
if _, ok = d.GetOk("auto_scaling"); ok {
	err = updateAutoScaling(ctx, d, client)
	if err != nil {
		return diag.FromErr(err)
	}
}
```

## Step 8 生成函数请求体

- 首先需要构造请求体，函数名为 `buildCreate{XXX}BodyParams`， 其中 `{XXX}` 为该资源的功能信息
- 如果API支持包周期，那么参数中的自动支付参数，如`auto_pay`，`is_auto_pay`， 直接设置为`true`
- 将函数放到创建resource的函数的后边

```go
func buildCreatePublicationBodyParams(d *schema.ResourceData) map[string]interface{} {
	bodyParams := map[string]interface{}{
		"publication_name":               d.Get("publication_name"),
		"publication_database":           d.Get("publication_database"),
		"is_create_snapshot_immediately": isCreateSnapshotImmediately,
		"subscription_options":           buildPublicationSubscriptionOptionsBodyParams(d.Get("subscription_options")),
		"job_schedule":                   buildPublicationJobScheduleBodyParams(d.Get("job_schedule")),
		"extend_tables":                  utils.ValueIgnoreEmpty(d.Get("extend_tables").(*schema.Set).List()),
		"tables":                         tables,
	}
}
```