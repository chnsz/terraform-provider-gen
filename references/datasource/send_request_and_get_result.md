# 发送请求、解析结果 流程

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

2. 如果API支持分页，并且URI中的请求方法为 GET，根据分页参数选择合适的分页逻辑

   - 分页参数为 limit + offset, ListAllItems中第二个参数qType为`offset`
   - 分页参数为 pagesize + page, ListAllItems中第二个参数qType为`page`
   - 分页参数为 limit + marker, ListAllItems中第二个参数qType为`marker`， 如果返回值中包含下一页的marker时，第四个参数中的MarkerField为下一页的marker

   ```go
   listResp, err := pagination.ListAllItems(
	   client,
	   "offset",
	   listPath,
	   &pagination.QueryOpts{MarkerField: ""})
   if err != nil {
	   return diag.Errorf("error retrieving RDS publications: %s", err)
   }
   listRespJson, err := json.Marshal(listResp)
   if err != nil {
	   return diag.FromErr(err)
   }
   var listRespBody interface{}
   err = json.Unmarshal(listRespJson, &listRespBody)
   if err != nil {
	   return diag.FromErr(err)
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