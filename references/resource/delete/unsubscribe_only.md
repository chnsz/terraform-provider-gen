# 生成仅支持退订资源的删除resource函数

## Step 1 生成删除资源的函数

- 生成一个删除资源的函数，格式为`resource{XXX}Delete`，其中 `{XXX}` 为该资源的功能信息，例如：`resourceInstanceDelete`
- 该函数总共包含：函数名称、参数定义、创建client、退订资源

```go
func resourceInstanceDelete(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// 参数定义

	// 创建client
	
	// 退订资源
	
	return nil
}
```

## Step 2 参数定义，创建client

- 创建退订资源的client，变量名为`bssClient`

```go
cfg := meta.(*config.Config)
region := cfg.GetRegion(d)

bssClient, err := cfg.BssV2Client(region)
if err != nil {
	return diag.Errorf("error creating bss V2 client: %s", err)
}
```

## Step 3 退订资源

- 判断 `charging_mode` 参数是否设置，如果设置了就判断是否为`prePaid`，如果是就使用`common.UnsubscribePrePaidResource`去退订资源

```go
if v, ok := d.GetOk("charging_mode"); ok && v.(string) == "prePaid" {
	if err = common.UnsubscribePrePaidResource(d, cfg, []string{d.Id()}); err != nil {
		return diag.Errorf("error unsubscribe RDS instance: %s", err)
	}
}
```

## Step 4 添加注释

- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置）：`@API BSS POST /v2/orders/subscriptions/resources/unsubscribe`
