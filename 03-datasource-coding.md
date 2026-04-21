# Data Source 编码开发技能

## Step 1 数据获取

- 调用用户提供的华为云API，从结果中获取到当前服务信息、功能、URI信息（包括URI和参数信息）、请求参数和响应消息
- 禁止使用 `web reference` 的内容
- 使用 `WebFetch` 获取完整的页面内容

## Step 2 生成data source函数

- 参考 ./references/datasource/schema_function.md
- 严格按照参考文档说明生成
- 禁止随意添加参数或者删除参数

## Step 3 生成 ReadContext 函数

- 参考 ./references/datasource/read_context_function.md
- 文档中的所有流程都必须执行，不能跳过
- 不要随意添加其他的流程

## Step 4 生成参数函数、请求体函数、解析结果函数

- 参考 ./references/datasource/param_and_flatten.md
- 文档中的所有流程都必须执行，不能跳过
- 不要随意生成其他函数

## Step 5. 调整包的顺序

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

## Step 6. 文件的包名为服务名的小写，不是`package huaweicloud`，例如`package rds`

## Step 7. 检查生成代码的合规性

- 如果查询参数类型为bool，需要将其转换为string，并且添加`ValidateFunc`
- 如果查询参数类型不为bool，不能添加`ValidateFunc`
- 分页参数不需要添加

## Step 8. 注册 Data Source 到 Provider

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
