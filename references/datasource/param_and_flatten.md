# 生成参数函数、请求体函数、解析结果函数 流程

## Step 1 生成参数函数

- 函数名为 `buildGet{XXX}QueryParams`， 其中 `{XXX}` 为该API要获取的功能信息，必须有参数`*schema.ResourceData`
- 如果参数为 `Required`，那么不需要判断，直接添加到结果即可，如果分页参数为必填，也需要添加上
- 如果参数为 `Optional`，那么需要先判断是否为空
- 如果参数类型为`bool`，但是通过`ValidateFunc`函数限制为`true`和`false`，那么需要先将其转换为`bool`，然后再添加到结果
- 如果参数类型为`list`，那么需要遍历该链表，然后逐个添加到结果中
- 将函数放到最后边

```go
func buildGetBackupDatabasesQueryParams(d *schema.ResourceData) string {
    res := ""
	res = fmt.Sprintf("%s&version_name=%v", res, v)
    if v, ok := d.GetOk("bucket_name"); ok {
    	res = fmt.Sprintf("%s&bucket_name=%v", res, v)
    }
    if v, ok := d.GetOk("bucket_exist"); ok {
        bucketExist, _ := strconv.ParseBool(v["bucket_exist"].(string))
    	res = fmt.Sprintf("%s&is_flexus=%v", res, v)
    }
    if v, ok := d.GetOk("bucket_params"); ok {
		for _, param := range v.([]interface{}) {
			res = fmt.Sprintf("%s&bucket_params=%v", res, param)
		}
	}
    // ... 其他查询参数
     
    if res != "" {
    	res = "?" + res[1:]
    }
    return res
}
```

## Step 2 生成请求体函数

- 函数名为 `buildGet{XXX}BodyParams`， 其中 `{XXX}` 为该API要获取的功能信息
- 如果API中包含`offset`，那么使用分页参数`offset`，`limit`为API中允许设置的最大值，记为`maxLimit`，如果API是使用`page+size`分页，那么使用分页参数`page`
- 该请求体要和上边构建请求参数的函数有区别，参数判空不要使用`d.GetOk`，使用`utils.ValueIgnoreEmpty`
- 将函数放到最后边

```go
func buildGetBackupDatabasesBodyParams(d *schema.ResourceData, offset int) map[string]interface{} {
	bodyParams := map[string]interface{}{
		"states":   utils.ValueIgnoreEmpty(d.Get("states")),
		"group_id": utils.ValueIgnoreEmpty(d.Get("group_id")),
		"limit":    100,
		"offset":    offset,
	}
	return bodyParams
}
```

### Step 3 解析结果函数

1. 如果返回参数类型为 list：

   - 函数名为 `flattenGet{XXX}Body`， 其中 `{XXX}` 为该参数的驼峰形式，返回值类型为`[]interface{}`
   - 首先使用`utils.PathSearch`从结果中获取结果，将结果转为数组后遍历，依次将数组对象中的每个元素添加到对应的map中，然后添加到返回对象中
   - 如果数组元素中参数类型为对象或者列表，那么就调调用解析结果函数，在子函数中实现，函数名定义和当前函数名类似规则
   - 如果数组中元素有递归结果，那么就只保留第一层，下边的直接将结果转换为json格式的字符串
   - 最后返回结果
   - 将函数放到最后边

   ```go
   func flattenGetBackupDatabasesBody(resp interface{}) []interface{} {
     curJson := utils.PathSearch("instance_publications", resp, make([]interface{}, 0))
	 curArray := curJson.([]interface{})
	 res := make([]interface{}, 0, len(curArray))
	 for _, v := range curArray {
        res = append(res, map[string]interface{}{
           "database_id":   utils.PathSearch("database_id", v, nil),
		   "database_name": utils.PathSearch("database_name", v, nil),
   			database_attr": flattenGetBackupDatabasesDatabaseAttrBody(v),
   		})
	 }
     return res
   }
   ```

2. 如果返回参数类型为对象：

   - 函数名为 `flattenGet{XXX}Body`， 其中 `{XXX}` 为该参数的驼峰形式，返回值类型为`[]interface{}`
   - 首先使用`utils.PathSearch`从结果中获取结果，奖结果对象中的参数依次添加到一个map中，然后将该map添加到一个数组中
   - 如果对象中参数类型为对象或者列表，那么就调用递归调用解析结果函数，函数名定义和当前函数名类似规则
   - 最后返回结果
   
   ```go
   func flattenGetBackupDatabasesBody(resp interface{}) []interface{} {
     curJson := utils.PathSearch("instance_publications", resp, nil)
	 if curJson == nil {
   	   return nil
	 }
	 res := []interface{}{
       map[string]interface{}{
		   "database_id": utils.PathSearch("database_id", resp, nil),
		   "database_name": utils.PathSearch("database_name", v, nil),
		   "database_attr": flattenGetBackupDatabasesDatabaseAttrBody(v),
       },
     }
	 return res
   }
   ```