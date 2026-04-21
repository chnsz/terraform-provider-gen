# 添加查询resource相关函数，生成创建资源所需方法

## Step 1 获取查询resource的数据

- 调用用户提供的查询资源的API，从结果中获取到当前服务信息、功能、URI信息（包括URI和参数信息）、请求参数和响应消息
- 禁止使用 `web reference` 的内容，使用 `WebFetch` 获取完整的页面内容

## Step 2 添加查询resource的参数

1. 如果API不是分页接口

   - 从响应消息中获取返回的resource信息
   - 如果响应消息中的参数在 `resource函数` 的请求参数中不存在，那么就添加到`resource 函数`中的`响应消息`处，并设置`Computed`为 `true`

2. 如果API是分页接口

   - 从响应消息中获取返回的resource信息，找到要查询的信息列表，并获取列表的元素详情
   - 如果列表元素详情中的参数在 `resource函数` 的请求参数中不存在，那么就添加到`resource 函数`中的`响应消息`处，并设置`Computed`为 `true`

```go
// 使用到的API
func ResourcePublication() *schema.Resource {
    return &schema.Resource{
		CreateContext: resourcePublicationCreate, 
		UpdateContext: resourcePublicationUpdate, 
		ReadContext:   resourcePublicationRead, 
		DeleteContext: resourcePublicationDelete,

		// ......

        Schema: map[string]*schema.Schema{
            "region": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            // ......
            
			// 响应参数
			"create_time": {
                Type:     schema.TypeString,
                Computed: true,
            },
        },
		
		// ......
    }
}
```

## Step 3 添加注释

- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path

## Step 4 生成查询resource的函数

- 参考 ./read_context_function.md
- 严格按照文档中的步骤执行，所有流程都必须执行，不能跳过
- 不要随意添加其他的流程

## Step 5 设置其他API查询参数

- 根据用户提供的更新resource参数API，依次添加参数更新方法

### Step 5.1 生成设置其他API参数函数

- 参考 ./set_field_function.md
- 严格按照文档中的步骤执行，所有流程都必须执行，不能跳过
- 不要随意添加其他的流程
- 生成的函数放到`查询resource的函数`后边

### Step 5.2 添加其他API查询参数

- 在`ReadContext 函数`中`设置其他API查询参数`位置添加其他API查询参数

1. 返回值类型为`error`

   ```go
   func resourcePublicationRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
   	  // 设置其他API查询参数
   	  mErr = multierror.Append(mErr, setAvailabilityZone(d, instance))
   }
   ```

2. 返回值类型为`[]error`

   ```go
   func resourcePublicationRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
   	  // 设置其他API查询参数
   	  mErr = multierror.Append(mErr, setDcsInstanceAutoScaling(d, instance)...)
   }
   ```

## Step 6 生成参数函数、请求体函数、解析结果函数

- 参考 ./param_and_flatten_function.md
- 严格按照文档中的步骤执行
- 不要随意添加其他参数，或者是删除参数
