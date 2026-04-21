# 生成仅支持删除资源的删除resource函数

## Step 1 获取删除资源的数据

- 调用用户提供的更新资源的API，从结果中获取请求参数
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容

## Step 2 生成删除资源函数

- 生成一个删除资源的函数，格式为`resource{XXX}Delete`，其中 `{XXX}` 为该资源的功能信息，例如：`resourceInstanceDelete`

```go
func resourcePublicationDelete(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
	// 参数定义

	// 创建client

    // 构造请求参数和请求体
	
	// 发送请求
	
	// 解析结果
	
	// 等待任务
}
```

## Step 3 添加注释

- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

## Step 4 参数定义，创建client

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

## Step 5 构造请求参数和请求体

- 将path参数替换为具体的值，禁止使用`utils.ReplacePathVariables`，使用`strings.ReplaceAll`
- 首先需要构造请求体，函数名为 `buildDelete{XXX}BodyParams`， 其中 `{XXX}` 为该资源的功能信息

```go
deletePath := client.Endpoint + httpUrl
deletePath = strings.ReplaceAll(deletePath, "{project_id}", client.ProjectID)
deletePath = strings.ReplaceAll(deletePath, "{instance_id}", d.Id())

deleteOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	MoreHeaders: map[string]string{"Content-Type": "application/json"},
}
deleteOpt.JSONBody = utils.RemoveNil(buildDeleteInstanceBodyParams(d))
```

## Step 6 发送请求、解析结果

1. 如果API响应消息为空

    - 那么就不需要处理返回结果

   ```go
   _, err = client.Request("DELETE", deletePath, &deleteOpt)
   if err != nil {
      return diag.Errorf("error deleting RDS instance: %s", err)
   }
   ```

2. 如果API响应消息不为空

    - 获取删除结果，然后使用`utils.FlattenResponse`去解析

   ```go
   deleteResp, err := client.Request("DELETE", deletePath, &deleteOpt)
   if err != nil {
     return diag.Errorf("error deleting RDS instance: %s", err)
   }
   
   deleteRespBody, err := utils.FlattenResponse(deleteResp)
   if err != nil {
     return diag.FromErr(err)
   }
   ```

## Step 7 等待任务

- 从API响应消息中获取任务ID信息，如果存在，那么就等待任务的完成
- 函数名为 `check{XXX}JobFinish`， 其中 `{XXX}` 为该资源的功能信息

```go
jobId := utils.PathSearch("job_id", deleteRespBody, "").(string)
if jobId != "" {
	if err = checkInstanceJobFinish(client, jobId, d.Timeout(schema.TimeoutUpdate)); err != nil {
		return diag.Errorf("error deleting instance: %s", err)
	}
}
```