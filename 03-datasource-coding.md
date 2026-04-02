# Data Source 编码开发技能

## 1. 数据获取

- 调用用户提供的华为云API，从结果中获取到当前服务信息、功能、URI信息（包括URI和参数信息）、请求参数和响应消息
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容

## 2. 生成data source函数

- 该函数的名称格式为 `DataSource{XXX}`，其中 `{XXX}` 为该API要获取的功能信息，采用驼峰状格式，例如：`DataSourceBackupDatabases`
- 该函数参数为空，返回值为 `*schema.Resource`
- 如果该服务为全局函数，那么就不需要添加`region`，如果不是全局函数，那么就需要添加参数`region`，并且放到最前边，并且设置`Optional`和`Computed` 为`true`
- 然后添加URI信息中的参数信息，如果有参数`project_id`，就忽略掉
- 然后添加请求参数，如果是必填，就设置`Required`为 `true`，如果是可选参数，就就设置`Optional`为 `true`，其中分页参数都要忽略掉
- 如果请求参数类型为`bool`，那么就将类型设置为`string`，并且添加`ValidateFunc`限制: `validation.StringInSlice([]string{"true", "false"}, false)`，其他情况都禁止使用`ValidateFunc`
- 最后添加响应消息，所有的参数都设置`Computed`为 `true`，其中表示返回数据数量的参数、请求信息、分页信息不返回，比如总数、请求ID
- 所有参数中，如果参数类型为对象，在这里对应的类型为list，并且在子函数中展示该对象的参数，子函数名格式为`{XXX}{YYY}Schema`，其中`{XXX}`为该API要获取的功能信息， `{YYY}` 为该参数的驼峰形式
- 最后在该函数前边加一个注释，格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

```go
// @API RDS GET /v3/{project_id}/instances/{instance_id}/database/db-table-name
func DataSourceBackupDatabases() *schema.Resource {
    return &schema.Resource{
        ReadContext: dataSourceBackupDatabasesRead,

        Schema: map[string]*schema.Schema{
            "region": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            // ... URI参数
            "instance_id": {
                Type:     schema.TypeString,
				Required: true,
            },
            // ... 请求参数
			 "version_name": {
                Type:     schema.TypeString,
				Required: true,
            },
            "bucket_name": {
                Type:     schema.TypeString,
				Required: true,
            },
			"bucket_exist": {
                Type:     schema.TypeString,
                Optional: true,
                ValidateFunc: validation.StringInSlice([]string{
					"true", "false",
				}, false),
            },
            "bucket_params": {
                Type:     schema.TypeList,
                Optional: true,
				Elem:     &schema.Schema{Type: schema.TypeString},
            },
            // ... 其他查询参数

            // ...响应消息
            "databases": {
                Type:     schema.TypeList,
                Computed: true,
                Elem:     backupDatabaseDatabasesSchema(),
            },
            // ... 其他响应消息
        },
    }
}

func backupDatabaseDatabasesSchema() *schema.Resource {
    return &schema.Resource{
        Schema: map[string]*schema.Schema{
            "database_name": {
                Type:     schema.TypeString,
                Computed: true,
            },
            "backup_id": {
                Type:     schema.TypeInt,
                Computed: true,
            },
            "backup_info": {
                Type:     schema.TypeList,
                Computed: true,
                Elem:     backupDatabaseDatabasesBackupInfoSchema(),
            },
            // ... 其他响应消息
        },
    }
}

func backupDatabaseDatabasesBackupInfoSchema() *schema.Resource {
    return &schema.Resource{
        Schema: map[string]*schema.Schema{
            "name": {
                Type:     schema.TypeString,
                Computed: true,
            },
            // ... 其他响应消息
        },
    }
}
```

## 3. 生成 ReadContext 函数

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

### 3.1 参数定义，创建client

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

### 3.2 构造请求url

- 将path参数替换为具体的值
- 如果查询参数是path参数，那么就需要添加分页参数之外的所有查询参数，通过`buildGet{XXX}QueryParams`方法实现，其中 `{XXX}` 为该API要获取的功能信息

#### 3.2.1 如果该API不支持分页，那么参数名应为：`getPath`

```go
    httpUrl := "v3/{project_id}/instances/{instance_id}/database/db-table-name"
    getPath := client.Endpoint + httpUrl
    getPath = strings.ReplaceAll(getPath, "{project_id}", client.ProjectID)
    getPath = strings.ReplaceAll(getPath, "{instance_id}", d.Get("instance_id").(string))
    getPath += buildGetDatabasesBackupQueryParams(d)
```

#### 3.2.2 如果该API支持分页，那么参数名应为：`listPath`

```go
    httpUrl := "v3/{project_id}/instances/{instance_id}/database/db-table-name"
    listPath := client.Endpoint + httpUrl
    listPath = strings.ReplaceAll(listPath, "{project_id}", client.ProjectID)
    listPath = strings.ReplaceAll(listPath, "{instance_id}", d.Get("instance_id").(string))
    listPath += buildGetDatabasesBackupQueryParams(d)
```

### 3.3 构造请求体

#### 3.3.1 如果API支持分页，并且请求URI中的请求方法不为GET，那么需要生成请求体，参数名为：`listOpt`

- 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

```go
	listOpt := golangsdk.RequestOpts{
        KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}
```

#### 3.3.2 如果API不支持分页，那么就需要生成请求体，参数名为：`getOpt`

- 使用`golangsdk.RequestOpts`生成请求体，禁止使用`utils.BaseRequestOpts()`

```go
	getOpt := golangsdk.RequestOpts{
        KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}
```

### 3.4 发送请求、解析结果

#### 3.4.1 如果API不支持分页，那么直接发送请求，参数依次为：URI中的请求方法、请求url、请求体，最后解析结果

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

#### 3.4.2 如果API支持分页，并且URI中的请求方法为 GET，那么直接使用 pagination 包中的 ListAllItems 函数

- 分页参数为 limit + offset, ListAllItems中第二个参数qType为"offset"
- 分页参数为 pagesize + page, ListAllItems中第二个参数qType为"page"
- 分页参数为 limit + marker, ListAllItems中第二个参数qType为"marker"， 如果返回值中包含下一页的marker时，第四个参数中的MarkerField为下一页的marker

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

#### 3.4.3 如果API支持分页，并且URI中的请求方法不为 GET

- 首先定义一个零时变量 `res` 保存查询结果
- 定义分页参数，如果API中包含`limit`，则定义变量`limit`, 并且其值为API中允许设置的最大值，记为`maxLimit`，如果API是使用page+size分页，那么定义变量`page`，并且赋值为 `1`
- 如果请求参数在请求体中，首先需要构造请求体，函数名为 `buildGet{XXX}BodyParams`， 其中 `{XXX}` 为该API要获取的功能信息，其中要包含分页参数， 最后使用utils.RemoveNil去除掉值为nil的参数
- 在for循环查询所有页的结果，请求参数依次为：URI中的请求方法、请求url、请求体
- 通过函数 `flattenGet{XXX}Body` 解析当前查询结果，其中 `{XXX}` 为该参数的驼峰形式
- 如果当前查询结果为空，那么就结束循环，禁止使用API返回结果中的总数来判断是否要结束循环
- 如果当前查询结果不为空，就将解析结果添加到 `res` 中，
- 最后更新分页参数，如果API中包含`offset`，那么`offset`增加`maxLimit`, 如果API是使用page+size分页，那么定义变量`page`， 那么`page`增加`1`

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

### 3.5 生成资源id

- 需要引入包 `"github.com/hashicorp/go-uuid"`
- 使用`uuid.GenerateUUID()`生成UUID

```go
    id, err := uuid.GenerateUUID()
	if err != nil {
		return diag.Errorf("unable to generate ID: %s", err)
	}
	d.SetId(id)
```

### 3.6 设置返回参数

#### 3.6.1 API支持分页，并且URI中的请求方法不为 GET

- 如果该服务为全局函数，那么就不需要返回`region`，如果不是全局函数，那么就需要返回`region`
- 结果已经在循环中解析完成，直接赋值即可

```go
    mErr = multierror.Append(
		d.Set("region", region),
		d.Set("databases", res),
	)
	return diag.FromErr(mErr.ErrorOrNil())
```

#### 3.6.2 API不支持分页，或者API支持分页，但是请求方法为 GET

- 如果该服务为全局函数，那么就不需要返回`region`，如果不是全局函数，那么就需要返回`region`
- 如果参数类型不为对象或者列表，那么就直接从返回结果中获取后设置即可
- 使用需要函数 `flattenGet{XXX}Body` 解析当前查询结果，其中 `{XXX}` 为该API要获取的功能信息

```go
    mErr = multierror.Append(
		d.Set("region", region),
		d.Set("attribute", utils.PathSearch("attribute", v, nil)),
		d.Set("databases", flattenGetBackupDatabasesBody(getRespBody)),
	)
	return diag.FromErr(mErr.ErrorOrNil())
```

## 4. 生成参数函数、请求体函数、解析结果函数

### 4.1 生成参数函数

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

### 4.2 生成请求体函数

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

### 4.3 解析结果函数

#### 4.3.1 返回参数类型为 list

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
			"database_attr": flattenGetBackupDatabasesDatabaseAttrBody(v),
		})
	}
	return res
}
```

#### 4.3.2 返回参数类型为对象

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

## 5. 调整包的顺序

- 按照go标准库包、github.com/hashicorp、github.com/chnsz、github.com/huaweicloud的顺序，并且中间用空行隔开

```go
import (
	"context"
	"encoding/json"
	"fmt"
	"strings"

	"github.com/hashicorp/go-multierror"
	"github.com/hashicorp/go-uuid"
	"github.com/hashicorp/terraform-plugin-sdk/v2/diag"
	"github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"

	"github.com/chnsz/golangsdk/pagination"

	"github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/config"
	"github.com/huaweicloud/terraform-provider-huaweicloud/huaweicloud/utils"
)
```

## 6. 文件的包名为服务名的小写，,不是`package huaweicloud`，例如`package rds`

## 7. 注册 Data Source 到 Provider

- 在 `provider.go` 中注册
- 找到同一服务的data source，将资源注册到统一服务的最后边

```go
DataSourcesMap: map[string]*schema.Resource{
    "huaweicloud_vpc":      vpc.DataSourceVpc(),
    "huaweicloud_vpcs":     vpc.DataSourceVpcs(),
    "huaweicloud_vpc_ids":  vpc.DataSourceVpcIds(),
    // ... 其他 Data Source
},
```

