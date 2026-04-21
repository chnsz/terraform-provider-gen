# 生成等待任务函数

## Step 1 创建等待任务函数

- 构造请求体，函数名为 `check{XXX}JobFinish`， 其中 `{XXX}` 为该资源的功能信息
- 定义一个变量`stateConf`，其值为`resource.StateChangeConf`的引用，创建该变量时需要设置`Pending`、`Target`、`Refresh`、`Timeout`、`PollInterval`
- 其中`Pending`为一个包含等待状态的字符串数组，其值为 `Pending`， 其中`Target`为一个包含等待状态的字符串数组，其值为 `Completed`
- `Refresh` 为刷新状态函数，调用刷新函数，函数名为`{XXX}JobStatusRefreshFunc`， 其中 `{XXX}` 为该资源的功能信息
- `Timeout` 为等待的超时时长，从参数中获取，`PollInterval` 固定为`10 * time.Second`
- 将函数放到最后边

```go
func checkPublicationJobFinish(ctx context.Context, client *golangsdk.ServiceClient, jobID string,
	timeout time.Duration) error {
	stateConf := &resource.StateChangeConf{
		Pending:      []string{"Pending"},
		Target:       []string{"Completed"},
		Refresh:      publicationJobStatusRefreshFunc(client, jobID),
		Timeout:      timeout,
		PollInterval: 10 * time.Second,
	}
	if _, err := stateConf.WaitForStateContext(ctx); err != nil {
		return fmt.Errorf("error waiting for RDS publication job (%s) to be completed: %s ", jobID, err)
	}
	return nil
}
```

## Step 2 创建刷新状态函数

- 函数名为 `{XXX}JobStatusRefreshFunc`， 其中 `{XXX}` 为该资源的功能信息

### Step 2.1 获取任务信息

- 调用获取任务状态函数`getJobInfo`获取任务信息，如果获取失败，那么直接状态`Failed`，对应的返回格式为`return nil, "Failed", err`

### Step 2.2 根据任务结果判断刷新函数

- 从结果中获取`status`， 根据状态值判断任务是否完成

1. 如果没有获取到`status`，则返回状态值为`Failed`，对应的返回格式为`return nil, "Failed", err`
2. 如果`status`值为`success`、`completed`等类似成功的状态，则返回状态为`Completed`, 对应的返回格式为`return getRespBody, "Completed", nil`
3. 如果`status`值为`fail`、`error`等类似失败的状态，则返回状态为`Failed`, 对应的返回格式为`return getRespBody, "Failed", fmt.Errorf("the job is fail")`
4. 如果`status`为其他值，那么就返回等待状态`Pending`，对应的返回格式为`return getRespBody, "Pending", nil`

```go
func publicationJobStatusRefreshFunc(client *golangsdk.ServiceClient, jobId string) resource.StateRefreshFunc {
	return func() (interface{}, string, error) {
		getRespBody, err := getJobInfo()
		if err != nil {
			return nil, "Failed", err
		}

		status := utils.PathSearch("status", getRespBody, "").(string)
		if status == "ERROR" {
			return nil, status, fmt.Errorf("the job is fail")
		}
		if status == "SUCCESS" {
			return getRespBody, "Completed", nil
		}
		return getRespBody, "Pending", nil
	}
}
```

## Step 3 创建获取任务信息函数

- 参考 ./get_job_function.md
- 严格按照文档中的步骤执行，所有流程都必须执行，不能跳过
- 不要随意添加其他的流程

## Step 4 添加注释

- 在`resource函数`前边加一个注释（示例中的`使用到的API`位置），格式为`// @API {service} {method} {path}`，其中`{service}`为当前服务名，`{method}`为URI中的请求方法，`{path}`为URI中的请求path