# 生成 resource函数 流程

- 该函数的名称格式为 `Resource{XXX}`，其中 `{XXX}` 为该资源的功能信息，采用驼峰状格式，例如：`ResourcePublication`
- 该函数参数为空，返回值为 `*schema.Resource`
- 为 `CreateContext`、`UpdateContext`、`ReadContext`、`DeleteContext` 四个变量设置值，分别为对应的实现函数，格式为`resource{XXX}{YYY}`，其中 `{XXX}` 为该资源的功能信息，`{YYY}`为函数动作
- 如果该服务为全局函数，那么就不需要添加`region`，如果不是全局函数，那么就需要添加参数`region`，并且放到最前边，并且设置`Optional`和`Computed` 为`true`

```go
// 不支持修改的参数列表

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
			
            // 请求参数
			
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
			
            // 更新参数

            // 响应消息
        },
    }
}

// 创建resource相关函数

// 更新resource相关函数

// 查询resource相关函数

// 删除resource相关函数

// 导入resource相关函数
```