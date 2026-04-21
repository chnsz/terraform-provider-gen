# 生成 ReadContext 函数流程

- 该函数总共包含：函数名称、参数定义、创建client、构造请求url、构造请求体、构造查询参数、发送请求、解析结果、生成资源id、设置返回参数

```go
func dataSourceRdsBackupDatabasesRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 参数定义

	// 创建client

    // 构造请求参数

	// 构造请求体

	// 发送请求

	// 解析结果

	// 生成资源id

	// 设置返回参数

}
```

## Step 1 参数定义，创建client

- 创建client时，其中第一个参数为服务类型，错误信息中的服务类型需要大写

```go
cfg := meta.(*config.Config)
region := cfg.GetRegion(d)

var mErr *multierror.Error

client, err := cfg.NewServiceClient("rds", region)
if err != nil {
	return diag.Errorf("error creating RDS client: %s", err)
}
```

## Step 2 构造请求url

- 将path参数替换为具体的值
- 如果查询参数是path参数，那么就需要添加分页参数之外的所有查询参数，通过`buildGet{XXX}QueryParams`方法实现，其中 `{XXX}` 为该API要获取的功能信息

1. 如果该API不支持分页，那么参数名应为：`getPath`

   ```go
   httpUrl := "v3/{project_id}/instances/{instance_id}/database/db-table-name"
   getPath := client.Endpoint + httpUrl
   getPath = strings.ReplaceAll(getPath, "{project_id}", client.ProjectID)
   getPath = strings.ReplaceAll(getPath, "{instance_id}", d.Get("instance_id").(string))
   getPath += buildGetDatabasesBackupQueryParams(d)
   ```

2. 如果该API支持分页，那么参数名应为：`listPath`

   ```go
   httpUrl := "v3/{project_id}/instances/{instance_id}/database/db-table-name"
   listPath := client.Endpoint + httpUrl
   listPath = strings.ReplaceAll(listPath, "{project_id}", client.ProjectID)
   listPath = strings.ReplaceAll(listPath, "{instance_id}", d.Get("instance_id").(string))
   listPath += buildGetDatabasesBackupQueryParams(d)
   ```

## Step 3 构造请求体

1. 如果API支持分页，并且请求URI中的请求方法不为GET，那么需要生成请求体，参数名为`listOpt`：

   - 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

   ```go
   listOpt := golangsdk.RequestOpts{
	   KeepResponseBody: true,
	   MoreHeaders: map[string]string{
		   "Content-Type": "application/json",
       },
   }
   ```

2. 如果API不支持分页，那么就需要生成请求体，参数名为`getOpt`：

   - 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

   ```go
   getOpt := golangsdk.RequestOpts{
	   KeepResponseBody: true,
	   MoreHeaders: map[string]string{
		   "Content-Type": "application/json",
          },
   }
   ```

## Step 4 发送请求、解析结果

- 参考 ./send_request_and_get_result.md
- 要严格遵守流程执行，不要跳过或者是添加其他流程

### Step 5 生成资源id

- 需要引入包 `"github.com/hashicorp/go-uuid"`
- 使用`uuid.GenerateUUID()`生成UUID

```go
dataSourceId, err := uuid.GenerateUUID()
if err != nil {
	return diag.Errorf("unable to generate ID: %s", err)
}
d.SetId(dataSourceId)
```

### Step 6 设置返回参数

1. API支持分页，并且URI中的请求方法不为 GET

   - 如果该服务为全局函数，那么就不需要返回`region`，如果不是全局函数，那么就需要返回`region`
   - 结果已经在循环中解析完成，直接赋值即可
   
   ```go
   mErr = multierror.Append(
	   d.Set("region", region),
	   d.Set("databases", res), 
   )
   return diag.FromErr(mErr.ErrorOrNil())
   ```

2. API不支持分页，或者API支持分页，但是请求方法为 GET

   - 如果该服务为全局函数，那么就不需要返回`region`，如果不是全局函数，那么就需要返回`region`
   - 如果参数类型不为对象或者列表，那么就直接从返回结果中获取后设置即可
   - 如果参数类型为对象或者列表， 需要使用函数 `flattenGet{XXX}Body` 解析当前查询结果，其中 `{XXX}` 为该API要获取的功能信息

   ```go
   mErr = multierror.Append(
	   d.Set("region", region),
	   d.Set("attribute", utils.PathSearch("attribute", v, nil)),
	   d.Set("databases", flattenGetBackupDatabasesBody(getRespBody)), 
   )
   return diag.FromErr(mErr.ErrorOrNil())
   ```
