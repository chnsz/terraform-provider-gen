# Resource 编码开发技能

## Step 1. 生成 resource 函数

- 参考 ./references/resource/create/schema_function.md
- 严格按照文档说明生成schema函数
- 禁止添加参考样例之外的内容

## Step 2. 添加创建resource相关函数，生成创建资源所需方法

- 参考 ./references/resource/create/create_resource.md
- 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
- 禁止添加参考样例之外的内容

## Step 3. 生成查询resource相关函数，生成查询资源所需方法

- 禁止使用`查询任务API`查询resource信息

1. 如果没有提供查询资源API：
   - 生成一个空函数，格式为`resource{XXX}Read`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationRead`

   ```go
   func resourcePublicationRead(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
	   return nil
   }
   ```

2. 如果提供了查询资源API：

   - 参考 ./references/resource/read/read_resource.md
   - 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
   - 禁止添加参考样例之外的内容

## Step 4. 生成更新resource相关函数，生成更新资源所需方法

1. 如果没有提供更新API，并且参数中没有 `enterprise_project_id`、 `tags`和 `auto_renew`：

   - 生成一个空函数，格式为`resource{XXX}Update`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationUpdate`

   ```go
   func resourcePublicationUpdate(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
	   return nil
   }
   ```

2. 如果提供了更新API，或者有 `enterprise_project_id`、 `tags`和 `auto_renew` 至少一个参数：

    - 参考 ./references/resource/update/update_resource.md
    - 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
    - 禁止添加参考样例之外的内容

## Step 5 添加不支持更新的参数列表

- 如果创建API的参数（包含URI参数和请求参数）不被更新API支持更新，那么该参数不支持更新，那么就在`resource函数`中添加不支持修改的参数列表（示例中的`不支持修改的参数列表`位置），类型为字符串数组
- 如果参数类型为list，那么就需要使用星号（`*`）来分开层级

```go
var rdsInstanceNonUpdatableParams = []string{
	"name",
	"images",
	"building_config.*.cluster", "building_config.*.image_pull_secrets",
}
```

## Step 6 添加 CustomizeDiff 信息 和 enable_force_new 参数

- 如果`Step 7`添加了不支持修改的参数列表：
    - 那么就在资源函数中添加CustomizeDiff 信息（示例中的`CustomizeDiff 信息`位置）
    - `resource函数`中添加`enable_force_new`参数

  ```go
  CustomizeDiff: config.FlexibleForceNew(rdsInstanceNonUpdatableParams),
  ```

  ```go
  "enable_force_new": {
  	Type:         schema.TypeString, 
  	Optional:     true,
  	ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false), 
  	Description:  utils.SchemaDesc("", utils.SchemaDescInput{Internal: true}),
  },
  ```

## Step 7. 生成删除resource的函数

1. 如果没有提供删除API，并且没有包周期参数

   - 生成一个返回warn信息的函数，格式为`resource{XXX}Delete`，其中 `{XXX}` 为该资源的功能信息，例如：`resourcePublicationDelete`
      
   ```go
   func resourcePublicationDelete(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
   	  errorMsg := "Deleting RDS publication resource is not supported. The resource is only removed from the state."
      return diag.Diagnostics{
   		 diag.Diagnostic{
   			Severity: diag.Warning,
   			Summary:  errorMsg,
   		 },
   	  }
   }
   ```

2. 如果没有提供删除API，但是有包周期参数

   - 参考 ./references/resource/delete/unsubscribe_only.md
   - 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
   - 禁止添加参考样例之外的内容

3. 如果有提供删除API，但是没有包周期参数：

   - 参考 ./references/resource/delete/delete_only.md
   - 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
   - 禁止添加参考样例之外的内容

4. 如果有提供删除APi，同时也有包周期参数：

    - 参考 ./references/resource/delete/delete_and_unsubscribe.md
    - 严格按照文档步骤执行，不能跳过，也不能随意添加其他步骤
    - 禁止添加参考样例之外的内容

## Step 8. 导入信息

1. 如果没有提供查询resource的API，那么不需要添加导入信息
2. 如果有提供查询resource的API

   1. 如果没有提供导入id格式 
   
      - 直接使用默认导入方法，在`resource函数`中添加导入信息（示例中的`导入信息`位置）

      ```go
      Importer: &schema.ResourceImporter{
      	  StateContext: schema.ImportStatePassthroughContext,
      },
      ```

   2. 如果有提供导入id格式

      - 需要使用自定义导入函数，函数名为： `resource{XXX}ImportState`，其中 `{XXX}` 为该资源的功能信息

      ```go
      Importer: &schema.ResourceImporter{
	      StateContext: resourceInstanceImportState,
      },
      ```

      - 生成自定义导入函数，函数名为： `resource{XXX}ImportState`，其中 `{XXX}` 为该资源的功能信息
      - 函数首先使用`strings.Split`对导入id（`d.Id()`），分割符号为`/`
      - 分割结果长度不符合，那么就直接返回异常
      - 重新设置资源id，一般为分割后结果的最后一个元素
      - 将分割结果设置到对应的参数中，返回结果
      - 将该函数放到最后边

      ```go
      func resourceInstanceImportState(_ context.Context, d *schema.ResourceData, _ interface{}) ([]*schema.ResourceData,
      	 error) {
      	 parts := strings.Split(d.Id(), "/")
      
      	 if len(parts) != 2 {
      		return nil, errors.New("invalid format specified for import ID, must be <instance_id>/<id>")
      	 }
      
      	 d.SetId(parts[1])
      	 mErr := multierror.Append(nil,
      		d.Set("instance_id", parts[0]),
      	 )
      
      	 return []*schema.ResourceData{d}, mErr.ErrorOrNil()
      }
      ```

## Step 9. 调整包的顺序

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

## Step 10. 文件的包名为服务名的小写，不是`package huaweicloud`，例如`package rds`
