# 生成更新参数函数

## Step 1 获取更新resource参数的数据

- 调用用户提供的更新资源的API，从结果中获取请求参数
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容
- 最后在资源函数前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

## Step 2 生成更新参数函数体

- 函数名为`update{XXX}{YYY}`，其中 `{XXX}` 为该资源的功能信息，`{YYY}` 为参数名称，函数名最终采用驼峰格式
- 函数参数首先需要包含`d`和`client`
- 如果API响应消息中包含`job_id`等任务信息，那么函数参数需要包含`ctx`
- 如果API请求消息中包含包周期相关参数，例如`自动支付（is_auto_pay）`，或者API响应消息中包含`order_id`，那么函数参数需要包含`bssClient`
- 将函数放到更新resource的函数的后边

```go
func updateRdsInstanceFlavor(ctx context.Context, d *schema.ResourceData, client, bssClient *golangsdk.ServiceClient) diag.Diagnostics {
    // 构造请求参数和请求体

	// 发送请求
	
	// 解析结果
	
	// 等待任务或订单完成
}
```

## Step 3 构造请求参数和请求体

- 将path参数替换为具体的值，禁止使用`utils.ReplacePathVariables`，使用`strings.ReplaceAll`
- 首先需要构造请求体，函数名为 `buildUpdate{XXX}{YYY}BodyParams`， 其中 `{XXX}` 为该资源的功能信息，`{YYY}` 为参数名称，函数名最终采用驼峰格式

```go
httpUrl := "v3/{project_id}/instances/{instance_id}/name" 
updatePath := client.Endpoint + httpUrl
updatePath = strings.ReplaceAll(updatePath, "{project_id}", client.ProjectID)
updatePath = strings.ReplaceAll(updatePath, "{instance_id}", fmt.Sprintf("%v", d.Id()))

updateOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	MoreHeaders: map[string]string{"Content-Type": "application/json"},
}
updateOpt.JSONBody = buildUpdateInstanceNameBodyParams(d)
```
## Step 4 发送请求、解析结果

### Step 4.1 如果API响应消息为空

- 那么就不需要处理返回结果

```go
_, err = client.Request("POST", getPath, &getOpt)
if err != nil {
	return diag.Errorf("error updating RDS instance name: %s", err)
}
```

### Step 4.2 如果API响应消息不为空

- 获取更新结果，然后使用`utils.FlattenResponse`去解析

```go
updateResp, err := client.Request("POST", getPath, &getOpt)
if err != nil {
	return diag.Errorf("error updating RDS instance name: %s", err)
}

updateRespBody, err := utils.FlattenResponse(updateResp)
if err != nil {
	return diag.FromErr(err)
}
```

## Step 5 等待任务或订单完成

### Step 5.2 添加等待订单完成逻辑

- 从API响应消息中获取订单ID信息，如果存在，那么就等待订单的完成

```go
orderId := utils.PathSearch("order_id", createRespBody, "").(string)
if orderId != "" {
	err = common.WaitOrderComplete(ctx, bssClient, orderId, d.Timeout(schema.TimeoutUpdate))
	if err != nil {
		return diag.FromErr(err)
	}
}
```

### Step 5.3 添加等待任务完成逻辑

- 从API响应消息中获取任务ID信息，如果存在，那么就等待任务的完成
- 函数名为 `check{XXX}JobFinish`， 其中 `{XXX}` 为该资源的功能信息


```go
jobId := utils.PathSearch("job_id", createRespBody, "").(string)
if jobId != "" {
	if err = checkInstanceJobFinish(client, jobId, d.Timeout(schema.TimeoutUpdate)); err != nil {
		return diag.Errorf("error updating publication: %s", err)
	}
}
```

## Step 6 生成函数请求体

- 首先需要构造请求体，函数名为 `buildUpdate{XXX}{YYY}BodyParams`， 其中 `{XXX}` 为该资源的功能信息，`{YYY}` 为参数名称，函数名最终采用驼峰格式
- 将函数放到最后边

```go
func updateRdsInstanceNameBodyParams(d *schema.ResourceData) map[string]interface{} {
	bodyParams := map[string]interface{}{
		"name": d.Get("name"),
	}
}
```

## Step 7 添加注释

- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

## Step 8 添加等待时间

- 在资源函数中添加更新资源等待时间（示例中的`超时时间`位置）
- 如果已经存在，则不需要再次添加

```go
Timeouts: &schema.ResourceTimeout{
	Update:  schema.DefaultTimeout(30 * time.Minute),
	// ....
},
```
