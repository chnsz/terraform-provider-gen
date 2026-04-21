# 添加创建resource相关函数，生成创建资源所需方法

## Step 1 获取创建resource的数据

- 调用用户提供的创建资源的API，从结果中获取到当前服务信息、功能、URI信息（包括URI和参数信息）、请求参数和响应消息
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容

## Step 2 添加创建resource的参数

### Step 2.1 添加URI信息中的参数信息到resource函数中（`URI参数`位置）

1. 如果有参数`project_id`，就忽略掉
2. 如果是必填，就设置`Required`为 `true`
3. 如果是可选参数，就就设置`Optional`为 `true`

### Step 2.2 添加请求参数到resource函数中（`请求参数`位置）

1. 如果是必填，就设置`Required`为 `true` 
2. 如果是可选参数，就就设置`Optional`为 `true`
3. 如果类型为`bool`，那么就将类型设置为`string`，并且添加validation限制`ValidateFunc`: `validation.StringInSlice([]string{"true", "false"}, false)`
4. 如果类型为对象，那么就将类型设置为`list`，设置`MaxItems`为 `1`，并且必须在子函数中展示该对象的参数，子函数名格式为`{XXX}{YYY}Schema`，其中`{XXX}`为该资源的功能信息， `{YYY}` 为该参数的驼峰形式，子函数放到resource函数后边 
5. 如果包含计费模式字段，如`charge_mode`或`charge_info`等字段，或者其说明中支持预付费模式，即`包年/包月`，或者取值包含`prePaid`，那么说明该API支持包周期
6. 如果支持包周期，那么就添加包周期参数： `charging_mode`、`period_unit`、`period`、`auto_renew`、`auto_pay`
7. 所有参数都不能设置`ForceNew`，除了`bool`类型之外，所有参数也都不能添加`ValidateFunc`

```go
// 使用到的API
func ResourcePublication() *schema.Resource {
    return &schema.Resource{
		CreateContext: resourcePublicationCreate, 
		UpdateContext: resourcePublicationUpdate, 
		ReadContext:   resourcePublicationRead, 
		DeleteContext: resourcePublicationDelete,

		// 导入信息
		
		// CustomizeDiff 信息
		
		// 超时时间

        Schema: map[string]*schema.Schema{
            "region": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            // URI参数
			"instance_id": {
                Type:     schema.TypeString,
				Required: true,
            },
            // 请求参数
            "bucket_name": {
                Type:     schema.TypeString,
                Optional: true,
            },
			"bucket_exist": {
                Type:     schema.TypeString,
                Optional: true,
                ValidateFunc: validation.StringInSlice([]string{
					"true", "false",
				}, false),
            },
            "subscription_options": {
				Type:     schema.TypeList,
				Optional: true,
				MaxItems: 1,
				Elem:     publicationSubscriptionOptionsSchema(),
			},
            "tables": {
				Type:     schema.TypeList,
				Optional: true,
				Elem:     publicationTablesSchema(),
			},
			// 包周期参数
			"charging_mode": {
				Type:     schema.TypeString,
				Optional: true,
				Computed: true,
				ValidateFunc: validation.StringInSlice([]string{
					"prePaid", "postPaid",
				}, false),
			},
			"period_unit": {
				Type:         schema.TypeString,
				Optional:     true,
				RequiredWith: []string{"period"},
				ValidateFunc: validation.StringInSlice([]string{
					"month", "year",
				}, false),
			},
			"period": {
				Type:         schema.TypeInt,
				Optional:     true,
				RequiredWith: []string{"period_unit"},
			},
			"auto_renew": common.SchemaAutoRenewUpdatable(nil),
			"auto_pay":   common.SchemaAutoPay(nil),
			// ......
        },
		
		// ......
    }
}

func publicationSubscriptionOptionsSchema() *schema.Resource {
	return &schema.Resource{
		Schema: map[string]*schema.Schema{
			"independent_agent": {
				Type:         schema.TypeString,
				Optional:     true,
				ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false),
			},
			"snapshot_always_available": {
				Type:         schema.TypeString,
				Optional:     true,
			},
		},
	}
}

func publicationTablesSchema() *schema.Resource {
	return &schema.Resource{
		Schema: map[string]*schema.Schema{
			"table_name": {
				Type:     schema.TypeString,
				Required: true,
			},
			"schema": {
				Type:     schema.TypeString,
				Optional: true,
			},
		},
	}
}
```

### Step 3 添加注释

- 如果支持包周期，那么在`resource函数`前边加两个注释（示例中的`使用到的API`位置）：`@API BSS POST /v2/orders/subscriptions/resources/autorenew/{instance_id}`和`@API BSS DELETE /v2/orders/subscriptions/resources/autorenew/{instance_id}`
- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

## Step 4 生成创建resource的函数

- 参考 ./create_context_function.md
- 严格按照文档中的步骤执行，所有流程都必须执行，不能跳过
- 不要随意添加其他的流程

## Step 5 生成等待任务函数

- 参考 ./wait_for_function.md
- 严格按照文档中的步骤执行，所有流程都必须执行，不能跳过
- 不要随意添加其他的流程

## Step 6 添加等待时间

- 在`resource函数`中添加创建资源等待时间（示例中的`超时时间`位置）

```go
Timeouts: &schema.ResourceTimeout{
	Create:  schema.DefaultTimeout(30 * time.Minute),
	// ....
},
```
