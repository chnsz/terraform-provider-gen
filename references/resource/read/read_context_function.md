# 生成 ReadContext 函数流程

## Step 1 生成查询资源的函数

- 格式为`resource{XXX}Read`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationRead`
- 该函数总共包含：函数名称、参数定义、创建client、构造请求参数、构造请求体、发送请求、解析结果、设置返回参数

```go
func resourcePublicationRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 参数定义

	// 创建client

    // 构造请求参数
	
	// 构造请求体
	
	// 发送请求
	
	// 解析结果
	
	// 设置返回参数
	
	// 设置其他API查询参数
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

## Step 3 构造请求url

- 将path参数替换为具体的值，禁止使用`utils.ReplacePathVariables`，使用`strings.ReplaceAll`
- 如果查询参数是path参数，那么就需要添加分页参数之外的所有查询参数，通过`buildGet{XXX}QueryParams`方法实现，其中 `{XXX}` 为该资源的功能信息

```go
getPath := client.Endpoint + httpUrl
getPath = strings.ReplaceAll(getPath, "{project_id}", client.ProjectID)
getPath = strings.ReplaceAll(getPath, "{instance_id}", d.Get("instance_id").(string))
getPath += buildGetPublicationQueryParams(d)
```

## Step 4 构造请求体

- 如果API不支持分页， 或者如果API支持分页，但是请求URI中的请求方法不为GET，那么需要生成请求体，参数名为：`getOpt`
- 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

```go
getOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	MoreHeaders: map[string]string{"Content-Type": "application/json"}
}
```

## Step 5 发送请求、解析结果

1. 如果API不支持分页

   - 直接发送请求，解析结果
   - 最后使用`utils.PathSearch`从结果中获取结果中根据`id`获取数据，如果结果为`nil`，就直接返回`common.CheckDeletedDiag`错误，其中包含 `Method`，`URL`，`RequestId`，`Body`

   ```go
   getResp, err := client.Request("GET", getPath, &getOpt)
   if err != nil {
	   return diag.Errorf("error retrieving RDS publication: %s", err)
   }
   
   getRespBody, err := utils.FlattenResponse(getResp)
   if err != nil {
	   return diag.FromErr(err)
   }
   publication := utils.PathSearch(fmt.Sprintf("publications[?id=='%s']|[0]", d.Id()), getRespBody, nil)
   if publication == nil {
	   return common.CheckDeletedDiag(d, golangsdk.ErrDefault404{
		   ErrUnexpectedResponseCode: golangsdk.ErrUnexpectedResponseCode{
			   Method:    "GET",
			   URL:       "/v3/{project_id}/instances/{instance_id}/replication/publications",
			   RequestId: "NONE",
			   Body:      []byte(fmt.Sprintf("the RDS publication (%s) does not exist", d.Id())),
			   },   , "error retrieving RDS publication")
   }
   ```

2. 如果API支持分页，并且URI中的请求方法为 GET，那么直接使用 pagination 包中的 ListAllItems 函数

   - 分页参数为 limit + offset, ListAllItems中第二个参数qType为`offset`
   - 分页参数为 pagesize + page, ListAllItems中第二个参数qType为`page`
   - 分页参数为 limit + marker, ListAllItems中第二个参数qType为`marker`， 如果返回值中包含下一页的marker时，第四个参数中的MarkerField为下一页的marker
   - 最后使用`utils.PathSearch`从结果中获取结果中根据`id`获取数据，如果结果为`nil`，就直接返回`common.CheckDeletedDiag`错误，其中包含 `Method`，`URL`，`RequestId`，`Body`
    
   ```go
   getResp, err := pagination.ListAllItems(
     client,
     "offset",
     listPath,
     &pagination.QueryOpts{MarkerField: ""})
   getRespJson, err := json.Marshal(getResp)
   err != nil {
     return diag.FromErr(err)
   }
   getRespBody interface{}
   err = json.Unmarshal(getRespJson, &getRespBody)
   if err != nil {
      return diag.FromErr(err)
   }
    	
   publication := utils.PathSearch(fmt.Sprintf("publications[?id=='%s']|[0]", d.Id()), getRespBody, nil)
   if publication == nil {
     return common.CheckDeletedDiag(d, golangsdk.ErrDefault404{
        UnexpectedResponseCode: golangsdk.ErrUnexpectedResponseCode{
    		Method:    "GET",
    		URL:       "/v3/{project_id}/instances/{instance_id}/replication/publications",
    		RequestId: "NONE",
			Body:      []byte(fmt.Sprintf("the RDS publication (%s) does not exist", d.Id())),
		},
     }, "error retrieving RDS publication")
   }
   ```

3. 如果API支持分页，并且URI中的请求方法不为 GET

    1. 首先定义一个零时变量 `res` 保存查询结果
    2. 定义分页参数
        - 如果API中包含`limit`， 记录API中允许设置的最大值，记为`maxLimit`，定义变量`offset`
        - 如果API是使用`page+size`分页，那么定义变量`page`，并且赋值为 `1`
    3. 如果请求参数在请求体中，首先需要构造请求体，函数名为 `buildGet{XXX}BodyParams`， 其中 `{XXX}` 为该API要获取的功能信息，其中要包含分页参数， 最后使用`utils.RemoveNil`去除掉值为nil的参数
    4. 在for循环查询所有页的结果，请求参数依次为：URI中的请求方法、请求url、请求体
    5. 通过函数 `flattenGet{XXX}Body` 解析当前查询结果，其中 `{XXX}` 为该参数的驼峰形式
    6. 判断查询到的结果
        - 如果当前查询结果为空，那么就结束循环，禁止使用API返回结果中的总数来判断是否要结束循环
        - 如果当前查询结果不为空，就将解析结果添加到 `res` 中，
    7. 更新分页参数
        - 如果API中包含`offset`，那么`offset`增加`maxLimit`
        - 如果API是使用page+size分页，那么定义变量`page`， 那么`page`增加`1`

    ```go
    offset := 0
	res := make([]map[string]interface{}, 0)
	for {
		getOpt.JSONBody = utils.RemoveNil(buildGetBackupDatabasesBodyParams(d, limit))
		getResp, err := client.Request("POST", getPath, &getOpt)
		if err != nil {
			return diag.Errorf("error retrieving RDS publication: %s", err)
		}
		getRespBody, err := utils.FlattenResponse(getResp)
		if err != nil {
			return diag.FromErr(err)
		}
		res = utils.PathSearch(fmt.Sprintf("publications[?id=='%s']|[0]", d.Id()), getRespBody, nil)
		if res != nil {
			break
		}
		publications = utils.PathSearch("publications", getRespBody, make([]interface{}, 0)).([]interface{})
		if len(publications) == 0 {
			break
		}
		
		offset += 100
	}
	if res == nil {
		return common.CheckDeletedDiag(d, golangsdk.ErrDefault404{
			ErrUnexpectedResponseCode: golangsdk.ErrUnexpectedResponseCode{
				Method:    "GET",
				URL:       "/v3/{project_id}/instances/{instance_id}/replication/publications",
				RequestId: "NONE",
				Body:      []byte(fmt.Sprintf("the RDS publication (%s) does not exist", d.Id())),
            },
        }, "error retrieving RDS publication")
    }
    ```

## Step 6 设置返回参数

- 如果该服务为全局函数，那么就不需要返回`region`，如果不是全局函数，那么就需要返回`region`
- 如果参数类型不为对象或者列表，那么就直接从返回结果中获取后设置即可
- 如果参数类型为对象或者列表， 需要使用函数 `flattenGet{XXX}Body` 解析当前查询结果，其中 `{XXX}` 为该资源的功能信息，如果有嵌套，那么`{XXX}` 为该资源的功能信息加上变量名，为驼峰格式

```go
mErr := multierror.Append(
	d.Set("region", region),
	d.Set("publication_name", utils.PathSearch("publication_name", publication, nil)),
	d.Set("publication_database", utils.PathSearch("publication_database", publication, nil)),
	d.Set("subscription_options", flattenPublicationSubscriptionOptions(publication)),
	d.Set("job_schedule", flattenPublicationJobSchedule(publication)),
	d.Set("is_select_all_table", strconv.FormatBool(
		utils.PathSearch("is_select_all_table", publication, false).(bool))),
	d.Set("extend_tables", utils.PathSearch("extend_tables", publication, nil)),
	d.Set("tables", flattenPublicationTables(publication)),
	d.Set("status", utils.PathSearch("status", publication, nil)),
	d.Set("subscription_count", utils.PathSearch("subscription_count", publication, nil)), 
)

return diag.FromErr(mErr.ErrorOrNil())
```