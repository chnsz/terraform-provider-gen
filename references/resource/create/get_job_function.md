# 生成获取数据函数

- 该函数总共包含：构造请求url、构造请求体、构造查询参数、发送请求、解析结果、返回结果

```go
func getInstanceInfo(_ context.Context, d *schema.ResourceData, meta interface{}) error {
    // 构造请求url

	// 构造请求体

	// 发送请求

	// 解析结果

	// 返回结果
}
```

## Step 1 构造请求url

- 将path参数替换为具体的值
- 如果查询参数是path参数，那么就需要添加分页参数之外的所有查询参数，通过`buildGet{XXX}QueryParams`方法实现，其中 `{XXX}` 为该API要获取的功能信息

```go
httpUrl := "v3/{project_id}/instances/{instance_id}/database/db-table-name"
getPath := client.Endpoint + httpUrl
getPath = strings.ReplaceAll(getPath, "{project_id}", client.ProjectID)
getPath = strings.ReplaceAll(getPath, "{instance_id}", d.Get("instance_id").(string))
getPath += buildGetDatabasesBackupQueryParams(d)
```

## Step 2 构造请求体

- 如果API不支持分页，或者API支持分页，但请求URI中的请求方法不为GET，就需要生成请求体，参数名为`getOpt`
- 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

```go
getOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	MoreHeaders: map[string]string{
		"Content-Type": "application/json",
	},
}
```

## Step 3 发送请求、解析结果

1. 如果API不支持分页，直接发送请求，参数依次为：URI中的请求方法、请求URL、请求体、最后解析结果：

   ```go
   getResp, err := client.Request("GET", getPath, &getOpt)
   if err != nil {
   	 return diag.Errorf("error retrieving RDS backup databases: %s", err)
   }
   
   getRespBody, err := utils.FlattenResponse(getResp)
   if err != nil {
   	 return diag.FromErr(err)
   }
   ```

2. 如果API支持分页，并且URI中的请求方法为 GET，根据分页参数选择合适的分页逻辑：

    - 分页参数为 limit + offset, ListAllItems中第二个参数qType为`offset`
    - 分页参数为 pagesize + page, ListAllItems中第二个参数qType为`page`
    - 分页参数为 limit + marker, ListAllItems中第二个参数qType为`marker`， 如果返回值中包含下一页的marker时，第四个参数中的MarkerField为下一页的marker

   ```go
   getResp, err := pagination.ListAllItems(
	   client,
	   "offset",
	   listPath,
	   &pagination.QueryOpts{MarkerField: ""})
   if err != nil {
	   return diag.Errorf("error retrieving RDS publications: %s", err)
   }
   getRespJson, err := json.Marshal(getResp)
   if err != nil {
	   return diag.FromErr(err)
   }
   var getRespBody interface{}
   err = json.Unmarshal(getRespJson, &getRespBody)
   if err != nil {
	   return diag.FromErr(err)
   }
   ```

3. 如果API支持分页，并且URI中的请求方法不为 GET：

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
     getOpt.JSONBody = utils.RemoveNil(buildGetBackupDatabasesBodyParams(d, offset))
	 getResp, err := client.Request("POST", getPath, &getOpt)
	 if err != nil {
	   return diag.Errorf("error retrieving RDS backup databases: %s", err)
	 }         	  
     getRespBody, err := utils.FlattenResponse(getResp)
     if err != nil {
       return diag.FromErr(err)
     }
     databases := flattenGetBackupDatabasesBody(getRespBody)
     if len(databases) == 0 {
       break
     }
     res = append(res, databases...)
     offset += 100
   }
   ```

## Step 4 返回结果

- 将获取到的结果`getRespBody`或 `res`返回