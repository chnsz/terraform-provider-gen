# 生成data source函数流程

## step 1. 生成函数名和返回值

- 该函数的名称格式为 `DataSource{XXX}`，其中 `{XXX}` 为该API要获取的功能信息，采用驼峰状格式，例如：`DataSourceBackupDatabases`
- 该函数参数为空，返回值为 `*schema.Resource`

## step 2. 设置`region`参数

- 如果该服务为全局函数，那么就不需要添加`region`
- 如果不是全局函数，那么就需要添加参数`region`，并且放到最前边，并且设置`Optional`和`Computed` 为`true`

## step 3. 添加URI信息中的参数信息

- 如果有参数`project_id`，就忽略掉
- 所有参数都设置`Required`为 `true`

## step 4. 添加请求参数

- 如果是必填，就设置`Required`为 `true`
- 如果是可选参数，就就设置`Optional`为 `true`
- 分页参数都要忽略掉
- 如果类型为`bool`，那么就将类型设置为`string`，并且添加`ValidateFunc`限制: `validation.StringInSlice([]string{"true", "false"}, false)`，其他类型参数都禁止使用`ValidateFunc`
- 如果参数类型为对象，在这里对应的类型为list，并且在子函数中展示该对象的参数，子函数名格式为`{XXX}{YYY}Schema`，其中`{XXX}`为该API要获取的功能信息， `{YYY}` 为该参数的驼峰形式

## step 5. 添加响应消息

- 如果参数类型为对象，在这里对应的类型为list，并且在子函数中展示该对象的参数，子函数名格式为`{XXX}{YYY}Schema`，其中`{XXX}`为该API要获取的功能信息， `{YYY}` 为该参数的驼峰形式
- 所有的参数都设置`Computed`为 `true`，其中表示返回数据数量的参数、请求信息、分页信息不返回，比如总数、请求ID

## step 6. 在该函数前边加一个注释

- 格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

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