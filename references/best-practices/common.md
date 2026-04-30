# 通用规范

## 职责范围

本部分适用于 **Resource 与 Data Source** 的通用代码质量检查，覆盖：方法抽象与可见性、辅助函数、Schema 定义、错误处理、import 分组与排序（已去重，见对应 Skill）、HTTP 请求封装、状态轮询、分页、URL 构建、日志、变量组织、Timeout、标签、注释、并发、性能、类型转换、工程性检查以及企业项目参数等。

## 使用建议（检查顺序）

优先按如下顺序核对，避免遗漏关键约束：

1. **方法抽象与可见性**：主流程简洁、复杂逻辑单一职责、导出/不导出有依据（测试/跨包复用）。
2. **Schema 定义**：参数/属性分类顺序、注释分隔、ForceNew/FlexibleForceNew 等关键约束。
3. **错误处理**：diag/multierror、资源不存在判断、非标准404转换、删除态处理。
4. **HTTP/URL/日志/轮询/分页**：请求封装一致性、占位符替换完整性、日志级别与信息密度、轮询与分页封装合理性。
5. **其余工程项**：并发锁、性能、类型转换、行长度、参数规范、未使用函数检查等。

### 1. 将一至多个API抽象为一个逻辑方法，并根据使用需求设置包内可见或包外可见

**设计原则**：

**必须**将复杂的逻辑处理流程根据场景抽象成独立的方法，只在方法中保留简洁的逻辑调用。
**尽可能地**将可抽象的逻辑操作提取成具备单一职责的方法，但不要过度抽象，具体原则如下：

- **默认包内可见**：除非确实需要包外调用，否则所有方法应设计为包内可见（首字母小写）
- **最小暴露原则**：只暴露必要的方法，避免过度暴露内部实现细节
- **命名一致性**：包外可见的方法命名应与包内可见的方法保持一致，仅首字母大小写不同
- **文档注释**：包外可见的方法应添加清晰的文档注释，说明方法用途、参数和返回值
- **函数命名**：
  * 对于包内可见的方法，首字母小写（不导出）。命名格式通常使用 `{action}{OperateObjectName}`、 `{action}V{VersionNumber}{OperateObjectName}`等精炼的名称，需要能清晰描述出方法的作用
  * 资源方法命名示例：`createV3Agency`、`updateV3Agency`、`deleteV3Agency`、`getV3AgencyById`、`detachProjectRoleFromV3Agency`
  * 数据源方法命名示例：`listV3Roles`、`getV3RoleById`、`buildV3RolesQueryParams`、`flattenV3Roles`
  * 如果需要在包外调用，则首字母大写，比如资源的查询详情方法需要在测试用例的方法中调用
- **函数职责**：保持抽象方法的单一职责
  * 资源方法：只处理API调用逻辑（将隶属于同一个处理逻辑的一至多个API按照顺序封装至一个独立方法中，包含构建HTTP请求、调用API、处理响应，而不包含其他逻辑）
  * 数据源方法：只处理API调用逻辑或数据转换逻辑（查询方法、查询参数构建方法、数据扁平化方法等）
- **函数参数**：通常包括 `client *golangsdk.ServiceClient`（必需）、`d *schema.ResourceData`（如需要从状态获取数据）、以及其他业务相关参数（如 `domainId`, `agencyId` 等）；如果需要处理重试或轮询，则将`ctx context.Context`设为首个入参，并补充超时参数（timeout time.Duration）
- **返回值**：
  * 资源方法：通常返回响应数据（对象类型为`interface{}`，对象列表类型为`[]interface{}`）和 `error`。针对对象类型，如果后续步骤需要使用响应数据则在返回前应使用 `utils.FlattenResponse()` 处理
  * 数据源方法：查询方法返回 `([]interface{}, error)` 或 `(interface{}, error)`，数据转换方法返回 `[]map[string]interface{}`
- **HTTP请求构建**：使用 `golangsdk.RequestOpts` 构建请求选项，设置 `KeepResponseBody: true` 以保留响应体
- **在主函数中使用**：主函数（如 `resourceV3AgencyCreate`、`dataSourceV3RolesRead`）应保持简洁，只包含准备工作、调用抽象方法、处理结果和后续步骤，不包含具体的API调用细节，保持代码可读性

在以下场景中，方法应被设计为包外可见（首字母大写）：

1. **测试用例复用**：测试文件中需要调用资源文件中的查询方法或其他辅助方法
   - **判断方法**：检查对应的测试文件（通常位于 `huaweicloud/services/acceptance/{service}/resource_huaweicloud_{resource}_test.go` 或 `data_source_huaweicloud_{resource}_test.go`），确认方法是否在测试用例中被调用
   - **常见场景**：测试用例中的 `CheckResourceExists()` 或 `CheckResourceDestroy()` 函数通常需要调用资源的查询方法
   - **示例**：如果 `GetMicroserviceEngineById` 在测试文件中被调用（如 `cse.GetMicroserviceEngineById(...)`），则应保持包外可见
2. **跨包复用**：其他服务包需要复用当前包的方法（如查询方法、工具方法等）
3. **数据源复用**：数据源文件需要复用资源文件中的查询方法

- 只有在确实存在包外调用需求时，才将方法设计为包外可见。默认情况下，所有辅助方法应设计为包内可见（首字母小写）。
- **重要**：在代码质量检查时，如果发现一个方法被设计为包外可见，应该检查对应的测试文件或其他包文件，确认该方法是否确实在包外被调用。如果确实被调用，则包外可见的设计是正确的；如果未被调用，则应改为包内可见。
- 包外可见的方法应保持与包内可见方法相同的参数和返回值设计
- 如果方法需要支持多场景，（如重试，在Create阶段需要但Read阶段不需要），可使用可变参数（如 `timeout ...time.Duration`，未传入或为 0 时表示不重试）
- 方法应尽可能返回 `interface{}`、 `error`等通用类型，便于统一处理
- 包外可见的方法通常是对包内可见方法的封装
- 包外可见的方法可以调用包内可见的方法，但应保持清晰的调用关系

**最佳实践**：

**资源方法示例**：

```go
// 包内可见的创建方法
func createV3Agency(client *golangsdk.ServiceClient, d *schema.ResourceData, domainId string) (interface{}, error) {
	httpUrl := "v3.0/OS-AGENCY/agencies"

	createPath := client.Endpoint + httpUrl
	createAgencyOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		JSONBody:         utils.RemoveNil(buildCreateV3AgencyRequestBody(d, domainId)),
	}

	requestResp, err := client.Request("POST", createPath, &createAgencyOpt)
	if err != nil {
		return nil, err
	}

	return utils.FlattenResponse(requestResp)
}

// 包内可见的基础查询方法
func getV3AgencyById(client *golangsdk.ServiceClient, agencyId string) (interface{}, error) {
	httpUrl := "v3.0/OS-AGENCY/agencies/{agency_id}"

	getPath := client.Endpoint + httpUrl
	getPath = strings.ReplaceAll(getPath, "{agency_id}", agencyId)

	getOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	respBody, err := client.Request("GET", getPath, &getOpt)
	if err != nil {
		return nil, err
	}

	return utils.FlattenResponse(respBody)
}

// 包外可见的带重试查询方法（封装包内方法）
func GetV3AgencyByIdWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout ...time.Duration) (interface{}, error) {
	var (
		respBody   interface{}
		err        error
		timeoutVal time.Duration
	)

	if len(timeout) < 1 || timeout[0] <= time.Duration(0) {
		return getV3AgencyById(client, agencyId)
	}
	timeoutVal = timeout[0]

	// lintignore:R006
	err = resource.RetryContext(ctx, timeoutVal, func() *resource.RetryError {
		respBody, err = getV3AgencyById(client, agencyId)
		if _, ok := err.(golangsdk.ErrDefault404); ok {
			// Retrieving agency details may result in a 404 error, requiring appropriate retries.
			// If the details are not retrieved within the timeout period, an error will be returned.
			// lintignore:R018
			time.Sleep(10 * time.Second)
			return resource.RetryableError(err)
		}
		if err != nil {
			return resource.NonRetryableError(err)
		}
		return nil
	})

	return respBody, err
}

// 在主函数中使用
func resourceV3AgencyCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg    = meta.(*config.Config)
		region = cfg.GetRegion(d)
	)

	iamClient, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	domainId := cfg.DomainID
	if domainId == "" {
		return diag.Errorf("the parameter 'domain_id' in provider-level configuration must be specified before creating agency")
	}

	// 创建委托
	respBody, err := createV3Agency(iamClient, d, domainId)
	if err != nil {
		return diag.Errorf("error creating IAM agency: %s", err)
	}

	// 保存资源ID
	agencyId := utils.PathSearch("agency.id", respBody, "").(string)
	if agencyId == "" {
		return diag.Errorf("unable to find the IAM agency ID from the API response")
	}
	d.SetId(agencyId)

	// 其他后续步骤（如角色绑定等）
	// ...

	return resourceV3AgencyRead(ctx, d, meta)
}
```

**数据源方法示例**：

```go
// 查询方法
func listV3Roles(client *golangsdk.ServiceClient, d *schema.ResourceData) ([]interface{}, error) {
	var (
		httpUrl = "v3/roles?per_page={per_page}"
		result  = make([]interface{}, 0)
		page    = 1
		perPage = 300
	)

	listPath := client.Endpoint + httpUrl
	listPath = strings.ReplaceAll(listPath, "{per_page}", strconv.Itoa(perPage))
	listPath += buildV3RolesQueryParams(d)

	opt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	for {
		listPathWithPage := fmt.Sprintf("%s&page=%s", listPath, strconv.Itoa(page))

		resp, err := client.Request("GET", listPathWithPage, &opt)
		if err != nil {
			return nil, err
		}

		respBody, err := utils.FlattenResponse(resp)
		if err != nil {
			return nil, err
		}

		roles := utils.PathSearch("roles", respBody, make([]interface{}, 0)).([]interface{})
		result = append(result, roles...)
		if len(roles) < perPage {
			break
		}
		page++
	}

	return result, nil
}

// 查询参数构建方法
func buildV3RolesQueryParams(d *schema.ResourceData) string {
	res := ""

	if v, ok := d.GetOk("display_name"); ok {
		res = fmt.Sprintf("%s&display_name=%v", res, v)
	}
	if v, ok := d.GetOk("name"); ok {
		res = fmt.Sprintf("%s&name=%v", res, v)
	}
	// ... 其他参数

	return res
}

// 数据扁平化方法
func flattenV3Roles(roles []interface{}) []map[string]interface{} {
	if len(roles) < 1 {
		return nil
	}

	result := make([]map[string]interface{}, 0, len(roles))
	for _, role := range roles {
		result = append(result, map[string]interface{}{
			"id":           utils.PathSearch("id", role, nil),
			"name":         utils.PathSearch("name", role, nil),
			"display_name": utils.PathSearch("display_name", role, nil),
			// ... 其他字段
		})
	}

	return result
}

// 在主函数中使用
func dataSourceV3RolesRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg    = meta.(*config.Config)
		region = cfg.GetRegion(d)
	)

	client, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	roles, err := listV3Roles(client, d)
	if err != nil {
		return diag.Errorf("error querying roles: %s", err)
	}

	randomUUID, err := uuid.GenerateUUID()
	if err != nil {
		return diag.Errorf("unable to generate ID: %s", err)
	}
	d.SetId(randomUUID)

	mErr := multierror.Append(
		d.Set("roles", flattenV3Roles(roles)),
	)

	return diag.FromErr(mErr.ErrorOrNil())
}
```

**包外可见方法命名规范**：

```go
// 查询方法命名（包外可见）：`Get{ResourceName}ById` 或 `GetV{VersionNumber}{ResourceName}ById`
func GetV3AgencyById(client *golangsdk.ServiceClient, agencyId string) (interface{}, error) {
	// ...
}

// 带重试的查询方法命名（包外可见）：`Get{ResourceName}ByIdWithRetry` 或 `GetV{VersionNumber}{ResourceName}ByIdWithRetry`
func GetV3AgencyByIdWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout ...time.Duration) (interface{}, error) {
	// ...
}

// 列表查询方法命名（包外可见）：`List{ResourceName}` 或 `ListV{VersionNumber}{ResourceName}`
func ListV3Agencies(client *golangsdk.ServiceClient) ([]interface{}, error) {
	// ...
}
```

**常见场景**：

1. **资源方法**：创建、更新、删除、查询等API调用方法
2. **数据源方法**：查询方法、查询参数构建方法、数据扁平化方法
3. **包外可见方法**：测试用例复用、跨包复用、数据源复用

**常见错误示例**：

```go
// ❌ 错误：方法只在包内使用，不应设计为包外可见
func GetV3AgencyById(client *golangsdk.ServiceClient, agencyId string) (interface{}, error) {
	// 只在 resourceV3AgencyRead 中使用，应改为 getV3AgencyById
}

// ✅ 正确：方法在测试文件中被调用，应设计为包外可见
func GetV3AgencyByIdWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout ...time.Duration) (interface{}, error) {
	// 在测试文件中被调用，保持包外可见
}
```

**检查清单**：

- [ ] 方法是否确实需要在包外调用？
  + [ ] 对于包外可见的方法，是否检查了对应的测试文件，确认方法在测试用例中被使用？
  + [ ] 对于包外可见的方法，是否检查了其他服务包，确认方法在其他包中被复用？
  + [ ] 对于包外可见的方法，是否检查了数据源文件，确认方法在数据源中被复用？
- [ ] 方法命名是否符合规范（包内可见首字母小写，包外可见首字母大写）？
- [ ] 包外可见的方法是否添加了清晰的文档注释？
- [ ] 方法签名是否合理（参数、返回值）？
- [ ] 包外可见的方法是否封装了包内可见的基础方法？
- [ ] 资源方法是否抽象了API调用逻辑？
- [ ] 数据源方法是否抽象了查询逻辑、查询参数构建逻辑或数据扁平化逻辑？
- [ ] 主函数是否保持简洁，只包含准备工作、调用抽象方法、处理结果和后续步骤？

### 2. 辅助函数

**设计原则**：

- **函数职责单一**：每个辅助函数只做一件事，保持职责清晰
- **纯函数优先**：辅助函数应该是纯函数（无副作用），便于测试和维护
- **命名规范**：函数名应该清晰表达其功能，使用统一的前缀（如 `build*`, `get*`, `parse*`, `attach*`, `detach*`, `flatten*`, `list*`）
- **错误处理**：错误消息应该包含足够的上下文信息，使用 `fmt.Errorf` 包装错误保持错误链（Attach/Detach函数除外）
- **代码复用**：将重复的逻辑抽象为辅助函数，提高代码复用性
- **参数设计**：函数参数应该合理，避免过多参数，必要时使用结构体封装

对于不同类型的辅助函数，分为以下几种不同的**要求**：

#### 2.1 构建请求体函数（资源专用）

**设计原则**：

- 构建请求Body的方法命名为：`build{StepName}{ObjectName}BodyParams` 或 `build{StepName}V{VersionNumber}{ObjectName}BodyParams`
- 子参数对应的方法命名根据层级命名为：`build{StepName}{ObjectName}{ParamName}({SubParamName}...)` 或 `build{StepName}V{VersionNumber}{ObjectName}{ParamName}({SubParamName}...)`
- 方法入参默认为`(d *schema.ResourceData)`，如果参数列表涉及企业项目则入参追加`*config.Config`
- 返回类型为`map[string]interface{}`
- 所有的可选参数都需要额外通过`utils.ValueIgnoreEmpty()`方法进行处理
- 所有的字符串列表（TypeList且Elem为&schema.Schema{Type: schema.TypeString}）的参数可直接通过`d.Get()`方法获取后直接透传
- 所有的字符串集合（TypeSet且Elem为&schema.Schema{Type: schema.TypeString}）的参数需要先进行`*schema.Set`类型的断言和`List()`方法的转换
- 所有对象列表类型（TypeList或TypeSet且Elem为&schema.Resource{Schema: ...}）的参数需要额外定义build方法
- 所有的对象（object）类型（TypeList且Elem为&schema.Resource{Schema: ...}，限制最大长度为1）的参数需要额外定义build方法
- 子方法的参数皆通过`utils.PathSearch()`方法进行获取，注意如果需要对其进行断言，则需要将默认值设为对应类型的空值

**最佳实践**：

```go
func buildCreateV3AgencyDelegatedDomain(d *schema.ResourceData) interface{} {
	if domainName, ok := d.GetOk("delegated_domain_name"); ok {
		return domainName.(string)
	}

	return d.Get("delegated_service_name").(string)
}

// The type of duration can be string or int in Create and Update methods
func buildCreateV3AgencyDuration(d *schema.ResourceData) interface{} {
	raw := d.Get("duration").(string)
	if raw == "" {
		return nil
	}

	// Try to convert duration to int, if success, return the converted value
	if days, err := strconv.Atoi(raw); err == nil {
		return days
	}

	return raw
}

func buildCreateV3AgencyBodyParams(d *schema.ResourceData, domainId string) map[string]interface{} {
	return map[string]interface{}{
		"agency": map[string]interface{}{
			// Required parameters.
			"domain_id":         domainId,
			"name":              d.Get("name").(string),
			"trust_domain_name": buildCreateV3AgencyDelegatedDomain(d),
			// Optional parameters.
			"description": d.Get("description").(string),
			"duration":    buildCreateV3AgencyDuration(d),
		},
	}
}
```

**检查清单**：

- [ ] 构建请求Body的方法命名是否符合规范（`build{StepName}{ObjectName}BodyParams`）？
- [ ] 子参数对应的方法命名是否符合规范（`build{StepName}{ObjectName}{ParamName}`）？
- [ ] 方法入参是否合理（默认为`d *schema.ResourceData`，如涉及企业项目则追加`*config.Config`）？
- [ ] 返回类型是否为`map[string]interface{}`？
- [ ] 可选参数是否使用`utils.ValueIgnoreEmpty()`处理？
- [ ] 字符串集合类型是否先进行类型断言和`List()`方法转换？
- [ ] 对象列表类型和对象类型是否定义了对应的build方法？
- [ ] 子方法的参数是否使用`utils.PathSearch()`方法获取，并提供了正确的默认值？

#### 2.2 查询参数构建函数（资源和数据源通用）

**设计原则**：

- 查询参数构建方法命名为：`build{ObjectName}QueryParams` 或 `buildV{VersionNumber}{ObjectName}QueryParams`
- 方法入参为`(d *schema.ResourceData)`
- 返回类型为`string`（查询参数字符串）
- 使用`d.GetOk()`检查参数是否存在，避免空值
- 查询参数之间使用`&`连接
- 如果第一个参数前需要添加`?`，则在调用方处理

**最佳实践**：

```go
func buildV3RolesQueryParams(d *schema.ResourceData) string {
	res := ""

	if v, ok := d.GetOk("display_name"); ok {
		res = fmt.Sprintf("%s&display_name=%v", res, v)
	}
	if v, ok := d.GetOk("name"); ok {
		res = fmt.Sprintf("%s&name=%v", res, v)
	}
	// ... 其他参数

	return res
}
```

**检查清单**：

- [ ] 查询参数构建方法命名是否符合规范（`build{ObjectName}QueryParams`）？
- [ ] 方法入参是否为`(d *schema.ResourceData)`？
- [ ] 返回类型是否为`string`？
- [ ] 是否使用`d.GetOk()`检查参数是否存在？
- [ ] 查询参数之间是否使用`&`连接？

#### 2.3 数据扁平化函数（资源和数据源通用）

**设计原则**：

- 数据扁平化方法命名为：`flatten{ObjectName}` 或 `flattenV{VersionNumber}{ObjectName}`
- 对象类型的入参固定为`map[string]interface{}`，返回为 `[]interface{}` 或 `[]map[string]interface{}`
- 列表类型的入参固定为`[]interface{}`，返回为 `[]interface{}` 或 `[]map[string]interface{}`
- 输入为空时返回`nil`
- 使用`utils.PathSearch()`提取字段值，提供正确的默认值
- 嵌套对象需要定义对应的扁平化方法

**最佳实践**：

```go
// 对象类型的扁平化
func flattenServerGroupProductInfo(productInfo map[string]interface{}) []map[string]interface{} {
	if len(productInfo) < 1 {
		return nil
	}

	return []map[string]interface{}{
		{
			"product_id": utils.PathSearch("product_id", productInfo, nil),
			"flavor_id":  utils.PathSearch("flavor_id", productInfo, nil),
			// ... 其他字段
		},
	}
}

// 列表类型的扁平化
func flattenV3Roles(roles []interface{}) []map[string]interface{} {
	if len(roles) < 1 {
		return nil
	}

	result := make([]map[string]interface{}, 0, len(roles))
	for _, role := range roles {
		result = append(result, map[string]interface{}{
			"id":           utils.PathSearch("id", role, nil),
			"name":         utils.PathSearch("name", role, nil),
			"display_name": utils.PathSearch("display_name", role, nil),
			// ... 其他字段
		})
	}

	return result
}
```

**检查清单**：

- [ ] 数据扁平化方法命名是否符合规范（`flatten{ObjectName}`）？
- [ ] 方法入参类型是否正确（对象为`map[string]interface{}`，列表为`[]interface{}`）？
- [ ] 方法返回类型是否为`[]map[string]interface{}`？
- [ ] 是否使用`utils.PathSearch()`提取字段值？
- [ ] 输入为空时是否返回`nil`？
- [ ] 嵌套对象是否定义了对应的扁平化方法？

#### 2.4 数据转换函数（资源和数据源通用）

**设计原则**：

- 数据转换函数应该是纯函数，不修改输入参数
- 函数命名使用`parse*`、`normalize*`、`convert*`等前缀，清晰表达转换功能
- 对于复杂的数据结构转换，应该定义专门的转换函数
- 转换过程中应该处理边界情况（如空值、nil值等）
- 使用`utils.PathSearch()`提取嵌套数据，提供正确的默认值
- 对于无效数据，应该记录警告日志并使用`continue`跳过，而不是中断流程

**最佳实践**：

```go
func parseProjectRolesToPairs(projectRoles *schema.Set) map[string]interface{} {
	// The key is the combination of project name and role name, which formatted as "project_name|role_name".
	// Map structure can ensure de-duplication, so there is no need to worry about duplication.
	result := make(map[string]interface{})

	for _, projectRole := range projectRoles.List() {
		projectName := utils.PathSearch("project", projectRole, "").(string)
		roleNames := utils.PathSearch("roles", projectRole, schema.NewSet(schema.HashString, nil)).(*schema.Set)

		// The project name and role names cannot be empty
		if projectName == "" || roleNames.Len() < 1 {
			log.Printf("[WARN] invalid project name (%s) or role names (%v)", projectName, roleNames.List())
			continue
		}

		for _, roleName := range roleNames.List() {
			// The key is the combination of project name and role name, which formatted as "project_name|role_name"
			result[fmt.Sprintf("%s|%v", projectName, roleName)] = true
		}
	}

	return result
}

// 使用utils.PathSearch的keys(@)方法提取map的所有键
func buildProjectRoles(projectRoles *schema.Set) []interface{} {
	return utils.PathSearch("keys(@)", parseProjectRolesToPairs(projectRoles), make([]interface{}, 0)).([]interface{})
}
```

**检查清单**：

- [ ] 数据转换函数是否为纯函数（不修改输入参数）？
- [ ] 函数命名是否清晰表达转换功能（使用`parse*`、`normalize*`、`convert*`等前缀）？
- [ ] 是否处理了边界情况（空值、nil值等）？
- [ ] 是否使用`utils.PathSearch()`提取嵌套数据，并提供了正确的默认值？
- [ ] 对于无效数据，是否记录警告日志并使用`continue`跳过？

#### 2.5 Attach/Detach函数（资源专用）

**设计原则**：

- 对于关联和解除关联操作，**必须**注意错误处理，不要格式化错误以保留原始状态码
- 函数命名使用`attach*`和`detach*`前缀，清晰表达操作类型
- 函数参数应该包含必要的资源标识（如`agencyId`, `projectId`, `roleId`等）
- 使用`golangsdk.RequestOpts`构建请求选项，设置`KeepResponseBody: true`以保留响应体
- 对于特定的HTTP状态码（如204），应该使用`OkCodes`指定
- 错误直接返回，不进行格式化，保留原始错误状态码

**最佳实践**：

对于关联和解除关联操作，需要注意错误处理，不要格式化错误以保留原始状态码：

```go
func attachProjectRoleToV3Agency(client *golangsdk.ServiceClient, agencyId, projectId, roleId string) error {
	httpUrl := "v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}"

	attachPath := client.Endpoint + httpUrl
	attachPath = strings.ReplaceAll(attachPath, "{project_id}", projectId)
	attachPath = strings.ReplaceAll(attachPath, "{agency_id}", agencyId)
	attachPath = strings.ReplaceAll(attachPath, "{role_id}", roleId)

	attachOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
		OkCodes: []int{204},
	}

	_, err := client.Request("PUT", attachPath, &attachOpt)
	// Note: here we cannot format the error, otherwise the original status code will be lost
	return err
}

func detachProjectRoleFromV3Agency(client *golangsdk.ServiceClient, agencyId, projectId, roleId string) error {
	httpUrl := "v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}"

	detachPath := client.Endpoint + httpUrl
	detachPath = strings.ReplaceAll(detachPath, "{project_id}", projectId)
	detachPath = strings.ReplaceAll(detachPath, "{agency_id}", agencyId)
	detachPath = strings.ReplaceAll(detachPath, "{role_id}", roleId)

	detachOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	_, err := client.Request("DELETE", detachPath, &detachOpt)
	// Note: here we cannot format the error, otherwise the original status code will be lost
	return err
}
```

**检查清单**：

- [ ] Attach/Detach函数是否保留了原始错误状态码（不格式化错误）？
- [ ] 函数命名是否清晰表达操作类型（使用`attach*`和`detach*`前缀）？
- [ ] 函数参数是否包含必要的资源标识？
- [ ] 是否使用`golangsdk.RequestOpts`构建请求选项，设置`KeepResponseBody: true`？
- [ ] 对于特定的HTTP状态码，是否使用`OkCodes`指定？
- [ ] 错误是否直接返回，不进行格式化？

#### 2.6 查询辅助函数（资源和数据源通用）

**设计原则**：

- 查询辅助函数应该封装复杂的查询逻辑，包括分页查询
- 函数命名使用`get*`、`list*`、`find*`等前缀，清晰表达查询功能
- 对于分页查询，应该正确处理分页逻辑（page、marker、offset等）
- 查询结果为空时，应该返回明确的错误信息
- 使用`utils.PathSearch()`提取查询结果，提供正确的默认值
- 对于模糊匹配的场景，应该使用强匹配（精确匹配）避免误匹配
- 错误直接返回，不进行格式化，保留原始错误信息

**最佳实践**：

```go
func getProjectIdByName(client *golangsdk.ServiceClient, domainId, name string) (string, error) {
	var (
		httpUrl = "v3/projects?name={name}&domain_id={domain_id}&per_page={per_page}"
		perPage = 100
		page    = 1
		result  = make([]interface{}, 0)
	)

	httpUrl = client.Endpoint + httpUrl
	httpUrl = strings.ReplaceAll(httpUrl, "{name}", name)
	httpUrl = strings.ReplaceAll(httpUrl, "{domain_id}", domainId)
	httpUrl = strings.ReplaceAll(httpUrl, "{per_page}", strconv.Itoa(perPage))

	getProjectOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
	}

	for {
		httpUrlWithPage := fmt.Sprintf("%s&page=%d", httpUrl, page)
		requestResp, err := client.Request("GET", httpUrlWithPage, &getProjectOpt)
		if err != nil {
			return "", err
		}
		respBody, err := utils.FlattenResponse(requestResp)
		if err != nil {
			return "", err
		}
		projects := utils.PathSearch("projects", respBody, make([]interface{}, 0)).([]interface{})
		result = append(result, projects...)
		if len(result) < perPage {
			break
		}
		page++
	}

	if len(result) == 0 {
		return "", fmt.Errorf("unable to find the project with name (%s) under domain (%s)", name, domainId)
	}

	// To prevent fuzzy matching of multiple projects, we use a strong match by name and take the first result.
	return utils.PathSearch(fmt.Sprintf("[?name=='%s']|[0].id", name), result, "").(string), nil
}
```

**检查清单**：

- [ ] 查询辅助函数是否封装了复杂的查询逻辑（包括分页查询）？
- [ ] 函数命名是否清晰表达查询功能（使用`get*`、`list*`、`find*`等前缀）？
- [ ] 分页查询是否正确处理了分页逻辑（page、marker、offset等）？
- [ ] 查询结果为空时，是否返回了明确的错误信息？
- [ ] 是否使用`utils.PathSearch()`提取查询结果，并提供了正确的默认值？
- [ ] 对于模糊匹配的场景，是否使用强匹配（精确匹配）避免误匹配？
- [ ] 错误是否直接返回，不进行格式化？

**辅助函数总体检查清单**：

- [ ] 辅助函数是否为纯函数（无副作用）？
- [ ] 函数名是否清晰表达其功能（使用统一的前缀如`build*`, `get*`, `parse*`, `attach*`, `detach*`, `flatten*`, `list*`）？
- [ ] 错误消息是否包含足够的上下文信息？
- [ ] 是否使用`fmt.Errorf`包装错误保持错误链（Attach/Detach函数除外）？
- [ ] Attach/Detach函数是否保留了原始错误状态码（不格式化错误）？
- [ ] 函数参数设计是否合理（避免过多参数）？
- [ ] 是否将重复的逻辑抽象为辅助函数，提高代码复用性？

### 3. Schema 定义

Schema 定义是 Terraform Provider 资源和数据源的核心部分，**必须**遵循严格的顺序和格式规范。
其中，Schema参数属性主要由以下几个部分组成（并给出了各组成部分的要求）：

- **Type**: 参数或属性的类型，必须指定（TypeString, TypeInt, TypeBool, TypeSet, TypeList, TypeMap 等）
- **Required/Optional/Computed**: 参数或属性的行为，必须至少指定其中的一个（`Required`、`Optional`、`Optional+Computed` 或 `Computed`）
- **Description**: 参数或属性的描述，必须提供
  - 可以直接通过反引号字符串直接定义参数或属性的描述
  - 允许通过utils.SchemaDesc()方法将参数从原有的行为修改为新的行为（例如：Optional -> Required），该方法使得在定义参数或属性描述的同时，使其在代码层面和对外文档的层面具备不同的参数行为（例如将可选参数对外暴露为必选参数）
- **ForceNew**: 是否不支持更新（不推荐使用，仅 `region` 参数可以设置，其他参数通过 `CustomizeDiff` 实现（仅资源））
- **Default**: 可选参数的默认值，只有行为是`Optional` 或 `Optional+Computed`的参数可以设置
- **ValidateFunc**: 验证函数（如 `validation.StringInSlice`、`validation.StringIsJSON`）
  - 包含枚举值的参数不允许声明（字符串类型的布尔值除外，如enable_force_new内部参数除外）
- **DiffSuppressFunc**: 自定义差异抑制函数
- **ExactlyOneOf/AtLeastOneOf/RequiredWith**: 字段间关系约束
- **Sensitive**: 敏感字段设置为 `true`
- **Set**: Set 类型的哈希函数（如 `schema.HashString`）

**数据源的特殊说明**：

- 数据源不包含 `ForceNew`、`CustomizeDiff` 等配置
- 数据源的可选参数通常不包含 `Computed: true`（与资源不同）；但当该参数需要在 `ReadContext` 中根据实际请求上下文、Provider 默认值或服务端返回进行回填时，允许使用 `Optional: true` + `Computed: true`
- 数据源的属性必须包含 `Computed: true`
- 数据源通常只需要定义：region → 必选参数（如果有）→ 可选参数 → 属性

对于不同类型的Schema内容，分为以下几种不同的**要求**：

#### 3.1 Schema 参数和属性的定义顺序

**设计原则**：

- Schema 的参数、属性的定义顺序**必须**严格按照以下顺序从上到下依次定义：
  1. **region**（如果存在）
  2. **特殊参数** （Special parameters，如果存在）
  3. **必选参数**（Required parameters）
  4. **可选参数**（Optional parameters）
  5. **属性**（Attributes，Computed）
  6. **内部参数**（Internal parameters，仅资源）
  7. **内部属性**（Internal attributes，仅资源）
  8. **废弃参数**（Deprecated parameters，仅资源）
  9. **废弃属性**（Deprecated attributes，仅资源）

**最佳实践**：

1. **资源示例（包含特殊参数）**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ... CRUD方法声明、CustomizeDiff声明等

		Schema: map[string]*schema.Schema{
			// region 参数（如果存在）
			"region": {
				// ...
			},

			// 特殊参数如：Authentication parameters
			"connect_address": {
				// ...
			}

			// Required parameters.
			"name": {
				// ...
			},

			// Optional parameters.
			"description": {
				// ...
			},

			// Attributes.
			"expire_time": {
				// ...
			},

			// Internal parameters.
			"enable_force_new": {
				// ...
			},

			// Internal attributes.
			// ...

			// Deprecated parameters.
			// ...

			// Deprecated attributes.
			// ...
		},
	}
}
```

2. **数据源示例（不包含特殊参数）**：

```go
func DataSourceV3Roles() *schema.Resource {
	return &schema.Resource{
		ReadContext: dataSourceV3RolesRead,

		Schema: map[string]*schema.Schema{
			"region": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				Description: `The region where the roles are located.`,
			},

			// Optional parameters.
			"display_name": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The display name of the role to be queried.`,
			},
			"name": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The name of the role to be queried.`,
			},

			// Attributes.
			"roles": {
				Type:        schema.TypeList,
				Computed:    true,
				Description: `The list of the roles.`,
				Elem: &schema.Resource{
					Schema: map[string]*schema.Schema{
						"id": {
							Type:        schema.TypeString,
							Computed:    true,
							Description: `The ID of the role.`,
						},
						// ... 其他属性
					},
				},
			},
		},
	}
}
```

**检查清单**：

- [ ] Schema参数和属性的定义顺序是否严格按照以下顺序：region（如果存在）→ 特殊参数（如果存在）→ 必选参数 → 可选参数 → 属性 → 内部参数（仅资源）→ 内部属性（仅资源）→ 废弃参数（仅资源）→ 废弃属性（仅资源）？
- [ ] 各分类之间是否使用注释分隔（如`// Required parameters.`、`// Optional parameters.`等）？
- [ ] 各分类之间是否保持一个空行？
- [ ] 是否只有`region`参数设置了`ForceNew: true`，其他参数都没有设置`ForceNew`（仅资源）？
- [ ] 如果需要不可更新参数，是否通过`CustomizeDiff`（`config.FlexibleForceNew()`）实现，而不是使用`ForceNew`（仅资源）？
- [ ] 如果定义了`CustomizeDiff`，是否在Schema中提供了对应的内部参数`enable_force_new`（仅资源）？
- [ ] 数据源是否不包含`ForceNew`、`CustomizeDiff`等配置？
- [ ] 数据源可选参数是否遵循以下规则：默认不包含`Computed: true`；仅当参数需要在`ReadContext`中回填时，才使用`Optional: true` + `Computed: true`？
- [ ] 数据源的属性是否包含`Computed: true`？

#### 3.2 项目参数

**设计原则**：

- 如果 API 的 URI 中存在 `project_id` 参数定义，则资源或数据源需要声明 `region` 参数（前提是这个project在代码中不来自于其他查询接口的响应体或常量、全局变量，请根据相关代码注释信息检查是否符合该情况）
- `region` 参数位于 schema 的第一个位置
- Schema 中的描述应该基于 Markdown 文档中的描述：去掉 `Specifies` 前缀，调整语序使其以 `The` 开头，结尾需要有英文句号，并使用反引号囊括
- 描述格式通常为：`"The region where the xxx is located."`（资源）或 `"The region where the xxxs are located."`（数据源，复数）
- 例如：Markdown 文档中为 `Specifies the region where the agency is located.`，Schema 中应为 `The region where the agency is located.`
- `region` 参数必须包含 `Computed` 行为
- `region` 参数必须包含 `ForceNew: true`（仅资源，Region参数不适用NonUpdatable）

**最佳实践**：

**资源示例**：

1. **资源API涉及project_id参数，请求中明确使用了IAM鉴权且需要使用client的ProjectID替换请求URI中的project_id**

```go
// @API FunctionGraph POST /v2/{project_id}/fgs/functions
func ResourceFgsFunction() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceFunctionCreate,
		// ...

		Schema: map[string]*schema.Schema{
			"region": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				ForceNew:    true,
				Description: `The region where the function is located.`,
			},

			// Required parameters.
			// ...
		},
	}
}

// ...

func createFunction(cfg *config.Config, client *golangsdk.ServiceClient, d *schema.ResourceData) (string, error) {
	httpUrl := "v2/{project_id}/fgs/functions"

	createPath := client.Endpoint + httpUrl
	// client中的ProjectID来自provider块或根据region查询的结果（与资源参数中的region参数对应）
	createPath = strings.ReplaceAll(createPath, "{project_id}", client.ProjectID)

	// ...
}

func resourceFunctionCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg                             = meta.(*config.Config)
		region                          = cfg.GetRegion(d)
		functionMetadataObjectParamKeys = []string{
			"lts_custom_tag",
		}
	)

	client, err := cfg.NewServiceClient("fgs", region)
	if err != nil {
		return diag.Errorf("error creating FunctionGraph client: %s", err)
	}

	funcUrn, err := createFunction(cfg, client, d)
	if err != nil {
		return diag.Errorf("error creating function: %s", err)
	}

	// ...
}
```

**2. 资源API不涉及project_id参数**：

```go
// @API IAM PUT /v3.0/OS-SECURITYPOLICY/domains/{domain_id}/api-acl-policy
// ...
func ResourceV3Acl() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceV3AclCreate,
		// ...

		Schema: map[string]*schema.Schema{
			// 因为资源中没有使用到project_id参数，故无需定义region
			// Required parameters.
			// ...
		},
	}
}
```

**3. 资源API涉及project_id参数，但project_id来源为固定值（API文档要求输入某特定固定值）**

```go
var (
	// The project ID of the microservice instance is the fixed value "default".
	microserviceInstanceProjectId = "default"
	// ...
)

// @API CSE GET /v4/{project_id}/registry/microservices/{serviceId}/instances/{instanceId}
// ...
func ResourceMicroserviceInstance() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceMicroserviceInstanceCreate,

		// ...

		Schema: map[string]*schema.Schema{
			// Authentication parameters.
			"auth_address": {
				Type:     schema.TypeString,
				Optional: true,
				Computed: true,
				Description: utils.SchemaDesc(
					`The address that used to request the access token.`,
					utils.SchemaDescInput{
						Required: true,
					}),
			},

			// ...
		}
	}
}


func createInstance(client *golangsdk.ServiceClient, d *schema.ResourceData) (interface{}, error) {
	var (
		httpUrl        = "v4/{project_id}/registry/microservices/{service_id}/instances"
		microserviceId = d.Get("microservice_id").(string)
	)

	createPath := client.Endpoint + httpUrl
	createPath = strings.ReplaceAll(createPath, "{project_id}", microserviceInstanceProjectId)
	// ...
}

```

**3. 资源API涉及project_id参数，但project_id来源为某方法的返回值（API文档要求输入某特定固定值）**

```go
// @API IAM PUT /v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}
// ...
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceV3AgencyCreate,
		// ...

		Schema: map[string]*schema.Schema{
			// Required parameters.
			"name": {
				Type:        schema.TypeString,
				Required:    true,
				Description: `The name of the agency.`,
			},
	}
}

// project_id对应的值来自于其他接口调用的返回值，并非通过region参数获取
func attachProjectRoleToV3Agency(client *golangsdk.ServiceClient, agencyId, projectId, roleId string) error {
	httpUrl := "v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}"

	attachPath := client.Endpoint + httpUrl
	attachPath = strings.ReplaceAll(attachPath, "{project_id}", projectId)
	// ...
}
```

**检查清单**：

- [ ] 如果API的URI中存在`project_id`参数定义，是否声明了`region`参数（前提是project不来自于其他查询接口的响应体或常量、全局变量）？
- [ ] `region`参数是否位于schema的第一个位置？
- [ ] 描述是否以`The`开头，结尾是否有英文句号，并使用反引号囊括？
- [ ] 描述格式是否为`"The region where the xxx is located."`（资源）或`"The region where the xxxs are located."`（数据源）？
- [ ] `region`参数是否包含`Computed: true`？
- [ ] 资源的`region`参数是否包含`ForceNew: true`？
- [ ] 数据源的`region`参数是否不包含`ForceNew`？
- [ ] `region`参数是否设置为`Optional: true`？

#### 3.3 特殊参数

**设计原则**：

- **定义依据**：特殊参数的定义不是根据 Schema 的行为（如 `Optional`、`Computed`、`Required`）和 Description 的配置决定的，而是根据**参数在整个代码逻辑中的作用**决定的
- **判断标准**：如果某些参数在整个资源生命周期的多数特殊逻辑处理中都被调用，且**仅在这些特殊逻辑处理中使用**，则将这一类参数归纳为特殊参数
- **位置要求**：特殊参数必须统一放在特殊参数部分进行声明，通常位于 `region` 参数之后、必选参数之前（特殊参数部分的参数之间也遵循 `#### 3.1` 的排序逻辑）
- **注释标识**：特殊参数部分必须使用注释进行标识，如 `// Authentication parameters.` 等
- **典型场景**：
  - **鉴权参数**：鉴权参数是特殊参数的一个典型分支，适用于需要自定义客户端连接的服务资源（如 CSE 微服务引擎相关资源）
    - 鉴权参数包括 `auth_address`、`connect_address`、`admin_user`、`admin_pass` 等
    - 这些参数在整个生命周期的 CRUD 操作中都被用于创建自定义客户端和获取授权令牌等特殊逻辑处理
    - 这些参数仅在这些特殊逻辑处理中使用，不用于其他业务逻辑
- **Schema 配置**：特殊参数的 Schema 配置（`Required`、`Optional`、`Computed` 等）应根据实际业务需求设置，不受特殊参数分类的影响
- **分类规则**：特殊参数应归类到特殊参数部分，而不是必选参数或可选参数部分，即使某些特殊参数在 Schema 中可能设置为 `Required: true`

**最佳实践**：

1. **特殊参数示例（CSE 微服务实例 - 包含鉴权参数）**：

以下示例展示了特殊参数的设计。鉴权参数（`auth_address`、`connect_address`、`admin_user`、`admin_pass`）是特殊参数的典型代表：
- 这些参数在整个资源生命周期的 CRUD 操作中都被用于特殊逻辑处理（创建自定义客户端、获取授权令牌）
- 这些参数仅在这些特殊逻辑处理中使用，不用于其他业务逻辑
- 因此，这些参数被统一归类为特殊参数，放在特殊参数部分进行声明

```go
func ResourceMicroserviceInstance() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// 该资源没有region参数，因此特殊参数位于顶部
			// Authentication parameters.
			"auth_address": {
				Type:     schema.TypeString,
				Optional: true,
				Computed: true,
				Description: utils.SchemaDesc(
					`The address that used to request the access token.`,
					utils.SchemaDescInput{
						Required: true,
					}),
			},
			"connect_address": {
				Type:        schema.TypeString,
				Required:    true,
				Description: `The address that used to send requests and manage configuration.`,
			},
			"admin_user": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The user name that used to pass the RBAC control.`,
			},
			"admin_pass": {
				Type:         schema.TypeString,
				Optional:     true,
				Sensitive:    true,
				RequiredWith: []string{"admin_user"},
				Description:  `The user password that used to pass the RBAC control.`,
			},

			// Required parameters.
			"microservice_id": {
				Type:        schema.TypeString,
				Required:    true,
				Description: `The ID of the dedicated microservice to which the instance belongs.`,
			},
			// ...
		},
	}
}

...

// 辅助函数：处理 auth_address 的默认值（如果未提供则使用 connect_address）
func getAuthAddress(d *schema.ResourceData) string {
	if v, ok := d.GetOk("auth_address"); ok {
		return v.(string)
	}
	// Using the connect address as the auth address if its empty.
	// The behavior of the connect address is required.
	return d.Get("connect_address").(string)
}

func resourceMicroserviceInstanceCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// 使用自定义客户端，而不是标准的 NewServiceClient
	client := common.NewCustomClient(true, d.Get("connect_address").(string))

	// 获取授权令牌（如果提供了 admin_user 和 admin_pass），且请求过程中getAuthAddress()方法使用到了auth_address参数
	token, err := GetAuthorizationToken(
		getAuthAddress(d),
		d.Get("admin_user").(string),
		d.Get("admin_pass").(string),
	)
	if err != nil {
		return diag.Errorf("error getting authorization token: %s", err)
	}
	if token != "" {
		// 将令牌添加到请求头
		createOpts.MoreHeaders["Authorization"] = token
	}

	// ...
}

// Read 方法：同样使用这些参数进行特殊逻辑处理
func resourceMicroserviceInstanceRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// 特殊逻辑处理：创建自定义客户端
	client := common.NewCustomClient(true, d.Get("connect_address").(string))
	
	// 特殊逻辑处理：获取授权令牌
	authAddress := getAuthAddress(d)
	adminUser := d.Get("admin_user").(string)
	adminPass := d.Get("admin_pass").(string)
	
	respBody, err := GetInstance(client, authAddress, adminUser, adminPass, microserviceId, instanceId)
	// ...
}

// Delete 方法：同样使用这些参数进行特殊逻辑处理
func resourceMicroserviceInstanceDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// 特殊逻辑处理：创建自定义客户端
	client := common.NewCustomClient(true, d.Get("connect_address").(string))
	
	// 特殊逻辑处理：获取授权令牌
	authAddress := getAuthAddress(d)
	adminUser := d.Get("admin_user").(string)
	adminPass := d.Get("admin_pass").(string)
	
	err := deleteInstance(client, authAddress, adminUser, adminPass, microserviceId, instanceId)
	// ...
}
```

**检查清单**：

- [ ] 特殊参数是否在整个资源生命周期的特殊逻辑处理中的多数场景被调用（如 Create、Read、Update、Delete 等方法）？
- [ ] 特殊参数是否仅在这些特殊逻辑处理中使用，不用于其他业务逻辑？
- [ ] 特殊参数是否统一放在特殊参数部分进行声明？
- [ ] 特殊参数是否位于正确的位置（region 之后、必选参数之前）？
- [ ] 是否使用注释（如 `// Authentication parameters.`）标识特殊参数部分？
- [ ] 特殊参数是否归类到特殊参数部分，而不是必选参数或可选参数部分？
- [ ] 描述是否以 `The` 开头，结尾是否有英文句号，并使用反引号囊括？
- [ ] **对于鉴权参数**：鉴权参数是否在整个 CRUD 操作中都用于创建自定义客户端和获取授权令牌等特殊逻辑处理？
- [ ] **对于鉴权参数**：是否使用注释 `// Authentication parameters.` 或 `// Authentication and request parameters.` 标识鉴权参数部分？
- [ ] **对于鉴权参数**：鉴权参数是否按照 `auth_address` → `connect_address` → `admin_user` → `admin_pass` 的顺序定义？
- [ ] **对于鉴权参数**：代码中是否在所有 CRUD 方法中使用 `common.NewCustomClient()` 创建客户端（而不是 `cfg.NewServiceClient()`）？
- [ ] **对于鉴权参数**：代码中是否在所有 CRUD 方法中使用 `GetAuthorizationToken()` 等方法获取授权令牌？

#### 3.4 必选参数

**设计原则**：

- 必选参数声明的顺序位于特殊参数之后（如果存在特殊参数），如果资源没有特殊参数则位于 `region` 之后（如果存在 `region` 参数）和可选参数之前
- Schema 中的参数描述应该基于 Markdown 文档中的描述：去掉 `Specifies` 前缀，调整语序使其以 `The` 或 `Whether`（bool 类型）开头，结尾需要有英文句号，并使用反引号囊括
- 例如：Markdown 文档中为 `Specifies the name of agency.`，Schema 中应为 `The name of the agency.`
- 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述位于 `Elem` 声明行之后
- `TypeBool` 的参数无需提供默认值
- 必选参数之间紧凑排列（参数声明顺序与 API 的顺序保持一致，如果创建 API 与更新或其他 API 不一致，则优先以创建 API 为准）
- 所有包含枚举值的参数不允许声明 `ValidateFunc`，仅定义类型
- 与 `region` 参数之间保持一个空行
- 子参数参数规则与上述规则一致，另外需按照 Required、Optional、Computed、Internal 的参数顺序排列
- `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Required: true}` 的视为必选参数，应当被分到必选参数范围的schema定义中；
  这种定义方式不与 `Optional: true, Computed: true` 冲突，代表着该参数在文档中对外体现为必填参数，但保留了其可选的schema行为。

**最佳实践**：

**1. 标准类型定义**

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			"region": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				ForceNew:    true,
				Description: `The region where the agency is located.`,
			},

			// Required parameters.
			"name": {
				Type:        schema.TypeString,
				Required:    true,
				Description: `The name of the agency.`,
			},
			"scaling_policy_by_session": {
				Type:     schema.TypeList,
				Required: true,
				MaxItems: 1,
				Elem: &schema.Resource{
					Schema: map[string]*schema.Schema{
						"session_usage_threshold": {
							Type:        schema.TypeInt,
							Required:    true,
							Description: `The total session usage threshold of the server group.`,
						},
						"shrink_after_session_idle_minutes": {
							Type:        schema.TypeInt,
							Required:    true,
							Description: `The number of minutes to wait before releasing instances with no session connections.`,
						},
					},
				},
				Description: `The session-based scaling policy configuration.`,
			},

			// Optional parameters.
			// ...
		},
	}
}
```

**2. 通过utils.SchemaDescInput{Required: true}将参数对外暴露为必选参数**

```go
func ResourceFgsFunction() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			"region": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				ForceNew:    true,
				Description: `The region where the function is located.`,
			},

			// Required parameters.
			"app": {
				Type:          schema.TypeString,
				Optional:      true,
				ConflictsWith: []string{"package"},
				// 如果API定义了参数是一个可选参数但是实际资源逻辑或文档中要求了该参数是一个必填参数，则必须进行以下声明
				Description: utils.SchemaDesc(
					`The group to which the function belongs.`,
					utils.SchemaDescInput{
						Required: true, // 允许使用SchemaDescInput将一个参数从原有的类型修改为新的类型，必须保证当前参数声明位于修改后的类型分类下
					},
				),
				// Optional+utils.SchemaDescInput{Required: true}或Optional+Computed+utils.SchemaDescInput{Required: true}是合法的，且需要分类到必填参数的Schema部分
			},
			// ...
		}
	}
}
```

**检查清单**：
- [ ] 必选参数是否位于`region`之后（如果存在`region`参数）和可选参数之前？
- [ ] 描述是否以`The`或`Whether`（bool类型）开头，结尾是否有英文句号，并使用反引号囊括？
- [ ] 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述是否位于`Elem`声明行之后？
- [ ] `TypeBool`的参数是否未提供默认值？
- [ ] 必选参数之间是否紧凑排列（参数声明顺序与API的顺序保持一致）？
- [ ] 包含枚举值的参数是否未声明`ValidateFunc`，仅定义类型？
- [ ] 与`region`参数之间是否保持一个空行？
- [ ] 子参数是否按照Required、Optional、Computed、Internal的顺序排列？
- [ ] 使用`utils.SchemaDesc()`且定义了`utils.SchemaDescInput{Required: true}`的参数是否被分到必选参数范围？

#### 3.5 可选参数

**设计原则**：

- 可选参数声明的顺序位于必选参数之后与属性声明之前
- Schema 中的参数描述应该基于 Markdown 文档中的描述：去掉 `Specifies` 前缀，调整语序使其以 `The` 或 `Whether`（bool 类型）开头，结尾需要有英文句号，并使用反引号囊括
- 例如：Markdown 文档中为 `Specifies the description (supplementary information) of the agency.`，Schema 中应为 `The description (supplementary information) of the agency.`
- 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述位于 `Elem` 声明行之后
- `TypeBool` 的参数无需提供默认值
- 可选参数之间紧凑排列（参数声明顺序与 API 的顺序保持一致，如果创建 API 与更新或其他 API 不一致，则优先以创建 API 为准）
- 所有包含枚举值的参数不允许声明 `ValidateFunc`，仅定义类型
- 与必选参数之间保持一个空行
- 子参数参数规则与上述规则一致，另外需按照 Required、Optional、Computed、Internal 的参数顺序排列
- **数据源的特殊要求**：数据源的可选参数不包含 `Computed: true`（与资源不同）

**最佳实践**：

**资源示例**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Required parameters.
			// ...

			// Optional parameters.
			"description": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The description (supplementary information) of the agency.`,
			},
			"duration": {
				Type:     schema.TypeString,
				Optional: true,
				Default:  "FOREVER",
				DiffSuppressFunc: func(_, oldValue, newValue string, _ *schema.ResourceData) bool {
					return parseOneDayDuration(oldValue) == parseOneDayDuration(newValue)
				},
				Description: `The validity period of the agency.`,
			},
			"project_role": {
				Type:     schema.TypeSet,
				Optional: true,
				Elem: &schema.Resource{
					Schema: map[string]*schema.Schema{
						"project": {
							Type:        schema.TypeString,
							Required:    true,
							Description: `The name of the project.`,
						},
						"roles": {
							Type:        schema.TypeSet,
							Required:    true,
							Elem:        &schema.Schema{Type: schema.TypeString},
							Description: `The list of role names used for assignment in a specified project.`,
						},
					},
				},
				Description: `The roles assignment for the agency which the projects are used to grant.`,
			},

			// Attributes.
			// ...
		},
	}
}
```

**数据源示例**：

```go
func DataSourceV3Roles() *schema.Resource {
	return &schema.Resource{
		ReadContext: dataSourceV3RolesRead,

		Schema: map[string]*schema.Schema{
			"region": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				Description: `The region where the roles are located.`,
			},

			// Optional parameters.
			"display_name": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The display name of the role to be queried.`,
			},
			"name": {
				Type:        schema.TypeString,
				Optional:    true,
				Description: `The name of the role to be queried.`,
			},

			// Attributes.
			// ...
		},
	}
}
```

**检查清单**：

- [ ] 可选参数是否位于必选参数之后与属性声明之前？
- [ ] 描述是否以`The`或`Whether`（bool类型）开头，结尾是否有英文句号，并使用反引号囊括？
- [ ] 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述是否位于`Elem`声明行之后？
- [ ] `TypeBool`的参数是否未提供默认值？
- [ ] 可选参数之间是否紧凑排列（参数声明顺序与API的顺序保持一致）？
- [ ] 包含枚举值的参数是否未声明`ValidateFunc`，仅定义类型？
- [ ] 与必选参数之间是否保持一个空行？
- [ ] 子参数是否按照Required、Optional、Computed、Internal的顺序排列？
- [ ] 数据源的可选参数是否不包含`Computed: true`？

#### 3.6 属性

**设计原则**：

- 属性声明的顺序位于可选参数之后与内部参数之前（如果有的话，如果不存在内部参数则位于内部属性之前，依次类推）
- Schema 中的属性描述应该基于 Markdown 文档中的描述：去掉 `Specifies` 前缀，调整语序使其以 `The` 或 `Whether`（bool 类型）开头，结尾需要有英文句号
- 例如：Markdown 文档中为 `Specifies the expiration time of the agency.`，Schema 中应为 `The expiration time of the agency.`
- 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述位于 `Elem` 声明行之后
- `TypeBool` 的参数无需提供默认值
- 属性之间紧凑排列（参数声明顺序与 API 的顺序保持一致，如果创建 API 与更新或其他 API 不一致，则优先以创建 API 为准）
- 所有包含枚举值的参数不允许声明 `ValidateFunc`，仅定义类型
- 与可选参数之间保持一个空行
- 子参数参数规则与上述规则一致，另外需按照 Required、Optional、Computed、Internal 的参数顺序排列
- 仅具备 `Computed:true` （不具备 `Optional:true`） 且 `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Computed: true}` 的视为属性，应当被分到属性范围的schema定义中
- **注意**：属性描述不应体现该属性是一个主动的操作参数，如：
  * `enable: Whether the scaling policy is enabled.` # 正确
  * `enable: Whether to enable the scaling policy.` # 错误

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Optional parameters.
			// ...

			// Attributes.
			"expire_time": {
				Type:        schema.TypeString,
				Computed:    true,
				Description: `The expiration time of the agency.`,
			},
			"create_time": {
				Type:        schema.TypeString,
				Computed:    true,
				Description: `The creation time of the agency.`,
			},

			// Internal parameters.
			// ...
		},
	}
}
```

**检查清单**：

- [ ] 属性是否位于可选参数之后与内部参数之前（如果存在内部参数）？
- [ ] 描述是否以`The`或`Whether`（bool类型）开头，结尾是否有英文句号？
- [ ] 对于结构体参数（`MaxItems=1`）、结构体列表参数（`MaxItems>1`），其参数描述是否位于`Elem`声明行之后？
- [ ] `TypeBool`的参数是否未提供默认值？
- [ ] 属性之间是否紧凑排列（参数声明顺序与API的顺序保持一致）？
- [ ] 包含枚举值的参数是否未声明`ValidateFunc`，仅定义类型？
- [ ] 与可选参数之间是否保持一个空行？
- [ ] 子参数是否按照Required、Optional、Computed、Internal的顺序排列？
- [ ] 仅具备`Computed:true`（不具备`Optional:true`）且`Description`中调用了`utils.SchemaDesc()`方法且定义了`utils.SchemaDescInput{Computed: true}`的参数是否被分到属性范围？
- [ ] 属性描述是否不体现该属性是一个主动的操作参数？

#### 3.7 内部参数

**设计原则**：

- 具备 `Optional:true` 且 `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Internal: true}` 的视为内部参数，应当被分到内部参数范围的schema定义中
- 内部参数声明的顺序位于属性之后与内部属性之前
- `enable_force_new` 作为特殊内部参数有以下固定写法
- 参数定义、描述等与以上属性部分基本保持一致

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Attributes.
			// ...

			// Internal parameters.
			"enable_force_new": {
				Type:         schema.TypeString,
				Optional:     true,
				ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false),
				Description:  utils.SchemaDesc("", utils.SchemaDescInput{Internal: true}),
			},
			"delegated_service_name": {
				Type:     schema.TypeString,
				Optional: true,
				Description: utils.SchemaDesc(
					`The name of the delegated service.`,
					utils.SchemaDescInput{
						Internal: true,
					},
				),
			},

			// Internal attributes.
			// ...
		},
	}
}
```

**检查清单**：

- [ ] 具备`Optional:true`且`Description`中调用了`utils.SchemaDesc()`方法且定义了`utils.SchemaDescInput{Internal: true}`的参数是否被分到内部参数范围？
- [ ] 内部参数是否位于属性之后与内部属性之前？
- [ ] `enable_force_new`作为特殊内部参数是否使用了固定写法（`Type: schema.TypeString`, `Optional: true`, `ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false)`）？
- [ ] 参数定义、描述等是否与属性部分基本保持一致？
- [ ] 描述是否使用`utils.SchemaDesc()`方法并声明`Internal: true`？

#### 3.8 内部属性

**设计原则**：

- 仅具备 `Computed:true` （不具备 `Optional:true`） 且 `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Internal: true}` 的视为内部属性，应当被分到内部属性范围的schema定义中
- 内部属性声明的顺序位于内部参数之后与废弃参数之前
- 属性定义、描述等与以上属性部分基本保持一致
- 描述为固定模板，仅替换单引号中的参数名称

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Internal parameters.
			// ...

			// Internal attributes.
			"strategy_origin": {
				Type:     schema.TypeString,
				Computed: true,
				Description: utils.SchemaDesc(
					`The script configuration value of this change is also the original value used for comparison with
 the new value next time the change is made. The corresponding parameter name is 'strategy'.`,
					utils.SchemaDescInput{
						Internal: true,
					},
				),
			},

			// Deprecated parameters.
			// ...
		},
	}
}
```

**检查清单**：

- [ ] 仅具备`Computed:true`（不具备`Optional:true`）且`Description`中调用了`utils.SchemaDesc()`方法且定义了`utils.SchemaDescInput{Internal: true}`的参数是否被分到内部属性范围？
- [ ] 内部属性是否位于内部参数之后与废弃参数之前？
- [ ] 属性定义、描述等是否与属性部分基本保持一致？
- [ ] 描述是否为固定模板，仅替换单引号中的参数名称？
- [ ] 描述是否使用`utils.SchemaDesc()`方法并声明`Internal: true`？

#### 3.9 废弃参数（新增资源和数据源不涉及）

**设计原则**：

- 具备 `Optional:true` 且 `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Deprecated: true}` 的视为废弃参数，应当被分到废弃参数范围的schema定义中
- 废弃参数定义于内部属性之后与废弃属性之前
- 参数定义、描述等与以上属性部分基本保持一致
- 废弃参数是由资源在后续维护的过程中（基于设计的考量）所进行的参数行为变更产生的，其参数行为与旧参数保持一致

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Internal attributes.
			// ...

			// Deprecated parameters.
			"old_parameter": {
				Type:     schema.TypeString,
				Optional: true,
				Computed: true,
				Description: utils.SchemaDesc(
					`The old parameter description.`,
					utils.SchemaDescInput{
						Deprecated: true,
					},
				),
			},

			// Deprecated attributes.
			// ...
		},
	}
}
```

**检查清单**：

- [ ] 具备`Optional:true`且`Description`中调用了`utils.SchemaDesc()`方法且定义了`utils.SchemaDescInput{Deprecated: true}`的参数是否被分到废弃参数范围？
- [ ] 废弃参数是否定义于内部属性之后与废弃属性之前？
- [ ] 参数定义、描述等是否与属性部分基本保持一致？
- [ ] 描述是否使用`utils.SchemaDesc()`方法并声明`Deprecated: true`？
- [ ] 废弃参数的行为是否与旧参数保持一致？

#### 3.10 废弃属性（新增资源和数据源不涉及）

**设计原则**：

- 仅具备 `Computed:true` （不具备 `Optional:true`） 且 `Description` 中调用了 `utils.SchemaDesc()` 方法且定义了 `utils.SchemaDescInput{Deprecated: true}` 的视为废弃属性，应当被分到废弃属性范围的schema定义中
- 废弃属性定义于废弃参数之后
- 废弃属性是由资源在后续维护的过程中（基于设计的考量）所进行的参数行为变更产生的，其参数行为与旧属性保持一致

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...

		Schema: map[string]*schema.Schema{
			// Deprecated parameters.
			// ...

			// Deprecated attributes.
			"create_timestamp": {
				Type:     schema.TypeString,
				Computed: true,
				Description: utils.SchemaDesc(
					`The timestamp of the agency creation.`,
					utils.SchemaDescInput{
						Deprecated: true,
					},
				),
			},
		},
	}
}
```

**检查清单**：

- [ ] 仅具备`Computed:true`（不具备`Optional:true`）且`Description`中调用了`utils.SchemaDesc()`方法且定义了`utils.SchemaDescInput{Deprecated: true}`的参数是否被分到废弃属性范围？
- [ ] 废弃属性是否定义于废弃参数之后？
- [ ] 属性定义、描述等是否与属性部分基本保持一致？
- [ ] 描述是否使用`utils.SchemaDesc()`方法并声明`Deprecated: true`？
- [ ] 废弃属性的行为是否与旧属性保持一致？

**Schema定义总体检查清单**：

- [ ] Schema字段是否指定了Type（TypeString, TypeInt, TypeBool等）？
- [ ] Schema字段是否指定了Required/Optional/Computed之一？
- [ ] Schema字段是否提供了Description（使用反引号字符串）？
- [ ] 是否只有`region`参数设置了ForceNew，其他参数通过CustomizeDiff实现（仅资源）？
- [ ] 参数顺序是否符合规范（region → 必选 → 可选 → 属性 → 内部参数（仅资源）→ 内部属性（仅资源）→ 废弃参数（仅资源）→ 废弃属性（仅资源））？
- [ ] 描述是否以`The`或`Whether`（bool类型）开头，结尾是否有英文句号？
- [ ] 结构体参数的描述是否位于`Elem`声明行之后？
- [ ] 属性描述是否不体现该属性是一个主动的操作参数？
- [ ] 子参数顺序是否符合规范（Required → Optional → Computed → Internal）？
- [ ] 包含枚举值的参数是否未声明ValidateFunc？
- [ ] 数据源是否不包含`ForceNew`、`CustomizeDiff`等配置？
- [ ] 数据源的可选参数是否不包含`Computed: true`？
- [ ] 数据源的属性是否包含`Computed: true`？

### 4. 错误处理规范

**设计原则**：

- **禁止**在子方法向上传递的中间路径破坏错误的类型
- 错误变量命名清晰
- 使用复数形式（`ErrCodes`）对错误变量进行命名表示可能包含多个错误码
- 在上层方法中格式化处理错误内容时，消息格式采用小写开头，如：`"error creating..."` 而不是 `"Error creating..."`
- 使用 `%s`, `%v`, `%+v` 等对错误上下文进行格式化

对于不同类型的错误错里，分为以下几种不同的**要求**：

#### 4.1 错误处理

CRUD主方法**使用 diag.Diagnostics**错误类型作为相关子方法处理后的返回：

**设计原则**：

- **错误处理分层**：
  + CRUD主方法（如`resourceV3AgencyCreate`, `dataSourceV3RolesRead`）必须使用`diag.Diagnostics`作为返回类型
  + 子方法（如`createV3Agency`、`getV3AgencyById`）返回`(interface{}, error)`或`([]interface{}, error)`
  + **错误格式化**（使用`fmt.Errorf`等添加上下文信息）应在父方法中进行，子方法不应进行错误格式化
  + **错误类型转换**（使用`ConvertExpectedXXXErrInto404Err`统一错误类型）**允许**在子方法中进行，特别是在查询方法、状态刷新函数等需要统一错误类型的场景
  + 子方法中的API调用错误可以直接返回，也可以进行错误类型转换后返回，但不应进行错误格式化

- **单个错误处理**：
  + 使用`diag.Errorf()`格式化单个错误
  + 错误消息格式：小写开头，包含操作上下文
  + 例如：`return diag.Errorf("error creating agency: %s", err)`

- **多个错误处理**：
  + 使用`multierror.Append()`收集多个错误
  + 使用`diag.FromErr(mErr.ErrorOrNil())`将多个错误转换为`diag.Diagnostics`
  + 适用于多个`d.Set()`操作等场景

- **错误消息格式规范**：
  + 使用小写开头：`"error creating..."` 而不是 `"Error creating..."`
  + 使用 `%s`, `%v`, `%+v` 等对错误进行格式化
  + 包含操作上下文：`"error creating agency: %s"`、`"error fetching objects: %v"`
  + 错误消息应该清晰描述问题，便于排查

- **错误传递原则**：
  + 子方法中的API调用错误可以直接返回，也可以进行错误类型转换（如`ConvertExpectedXXXErrInto404Err`）后返回
  + 子方法**不应**进行错误格式化（使用`fmt.Errorf`等添加上下文信息），但**可以**进行错误类型转换或显式地通过代码判断和构造新错误的方式以统一错误类型
  + 父方法负责错误格式化和上下文信息添加
  + 确保错误链的完整性，便于问题追踪

**最佳实践**：

```go
func createV3Agency(client *golangsdk.ServiceClient, d *schema.ResourceData, domainId string) (interface{}, error) {
	httpUrl := "v3.0/OS-AGENCY/agencies"

	createPath := client.Endpoint + httpUrl
	createAgencyOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
		JSONBody: utils.RemoveNil(buildCreateV3AgencyRequestBody(d, domainId)),
	}

	requestResp, err := client.Request("POST", createPath, &createAgencyOpt)
	if err != nil {
        // 不在进行接口调用的子方法中进行错误的格式化，而交由负责进行逻辑处理的父方法去执行该操作
        // 但可以进行错误类型转换（如ConvertExpectedXXXErrInto404Err）以统一错误类型
		return nil, err
	}

	return utils.FlattenResponse(requestResp)
}

func resourceV3AgencyCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 处理单个错误，且在此处（逻辑调用层）对API返回的错误进行格式化
	respBody, err := createV3Agency(iamClient, d, domainId)
	if err != nil {
		return diag.Errorf("error creating agency: %s", err)
	}

    // ...
}

func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 多个错误（使用 multierror）
	mErr := multierror.Append(nil,
		d.Set("name", utils.PathSearch("agency.name", agency, "")),
		d.Set("description", utils.PathSearch("agency.description", agency, "")),
		d.Set("expire_time", utils.PathSearch("agency.expire_time", agency, "")),
		d.Set("create_time", utils.PathSearch("agency.create_time", agency, "")),
		d.Set("duration", normalizeAgencyDuration(utils.PathSearch("agency.duration", agency, ""))),
	)

    return diag.FromErr(mErr.ErrorOrNil())

    // ...
}
```

#### 4.2 判断资源不存在的错误处理（查询接口正常返回404错误）

当资源的查询接口在资源不存在时返回404错误时，其资源代码的ReadContext方法中**必须**使用common包中的`CheckDeletedDiag()`方法对错误进行处理。该方法会将资源不存在的情况转换为警告，而不是错误，确保该资源在通过 `terraform` 命令进行管理的过程中不会因为资源已被删除而报错。

```go
func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	agency, err := getV3AgencyById(client, d.Id())
	if err != nil {
		return common.CheckDeletedDiag(d, err, "error retrieving agency")
	}
	// ...
}
```

当查询接口可能在资源刚创建后短暂返回404错误（资源可能正在创建中）时，需要在查询方法中添加重试逻辑。重试应该只在资源是新创建的情况下进行：通过可变参数传入 timeout，当未传入或 timeout 为 0 时仅调用一次查询接口，当传入且大于 0 时在 404 时进行重试。

```go
// 查询方法（带重试）
func GetV3AgencyByIdWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout ...time.Duration) (interface{}, error) {
	var (
		respBody   interface{}
		err        error
		timeoutVal time.Duration
	)

	if len(timeout) < 1 || timeout[0] <= time.Duration(0) {
		return getV3AgencyById(client, agencyId)
	}
	timeoutVal = timeout[0]

	// lintignore:R006
	err = resource.RetryContext(ctx, timeoutVal, func() *resource.RetryError {
		respBody, err = getV3AgencyById(client, agencyId)
		if _, ok := err.(golangsdk.ErrDefault404); ok {
			// Retrieving agency details may result in a 404 error, requiring appropriate retries.
			// If the details are not retrieved within the timeout period, an error will be returned.
			// lintignore:R018
			time.Sleep(10 * time.Second)
			return resource.RetryableError(err)
		}
		if err != nil {
			return resource.NonRetryableError(err)
		}
		return nil
	})

	return respBody, err
}

// ReadContext方法中：根据具体的业务需求进行设置，这里agency的逻辑是仅在创建新资源时需要将查询失败（返回404错误）的查询接口进行重试
// 当传入 timeout 则表示当前需要进行重试（为创建资源后的下一次查询，通过d.IsNewResource()进行判断），否则不传或传 0，则只查询一次
func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var timeout time.Duration
	if d.IsNewResource() {
		timeout = d.Timeout(schema.TimeoutRead)
	}
	agency, err := GetV3AgencyByIdWithRetry(ctx, client, agencyId, timeout)
	if err != nil {
		return common.CheckDeletedDiag(d, err, "error retrieving agency")
	}
	// ...
}
```

#### 4.3 判断资源不存在的错误处理（查询接口返回其他类型错误或除了返回404错误以外还会返回其他类型错误）

当资源的查询接口在资源不存在时返回400错误（或其他非404错误）时，**必须**使用common包中的`ConvertExpected400ErrInto404Err()`方法将非标准404错误转换为标准404错误，然后再使用`CheckDeletedDiag()`方法对错误进行处理。

**设计原则**：

- **错误转换位置**：错误类型转换（`ConvertExpectedXXXErrInto404Err`）**允许**在子方法中进行，特别是在查询方法、状态刷新函数等需要统一错误类型的场景。错误格式化（使用`fmt.Errorf`等）应在父方法中进行。
- 错误码列表需要定义为全局变量，位于资源主函数上方
- 变量命名格式：`{resourceName}NotFoundErrCodes` 或 `v{VersionNumber}{ResourceName}NotFoundErrCodes`，其中`{resourceName}`为当前资源对象的驼峰命名
- 如果存在多种资源不存在错误（资源不存在场景及父资源不存在场景）且各不相同，则使用common的对应转换方法嵌套调用
- 错误码路径字段名需要根据API实际返回的错误结构确定（常见的有`error_code`、`error.code`、`code`、`errCode`等）

**最佳实践**：

```go
// 全局变量定义（位于资源主函数上方）
// 无论错误码有一个或者多个，都定义成字符串列表，全局变量的命名为xxxNotFoundErrCodes，xxx为资源对象名称
var instanceNotFoundCodes = []string{
	"DDS.0010", // Instance does not exist
	"DDS.0011", // Instance is deleted
}

// ReadContext方法中
func resourceDdsInstanceRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	instance, err := getDdsInstanceById(client, d.Id())
	if err != nil {
		return common.CheckDeletedDiag(d,
			common.ConvertExpected400ErrInto404Err(err, "error_code", instanceNotFoundCodes...),
			"error retrieving DDS instance",
		)
	}
	// ...
}
```

当资源不存在可能由多种原因导致（如资源本身不存在、父资源不存在等），且返回不同的错误码和状态码时，需要使用嵌套调用：

```go
// 全局变量定义（位于资源主函数上方）
var trustedServiceNotFoundErrCodes = []string{
	"ORG.0001", // Trusted service does not exist
	"ORG.0002", // Trusted service is deleted
}

var organizationNotFoundErrCodes = []string{
	"ORG.1001", // Organization does not exist
	"ORG.1002", // Organization is deleted
}

// ReadContext方法中
func resourceTrustedServiceRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	trustedService, err := getTrustedServiceById(client, d.Id())
	if err != nil {
		return common.CheckDeletedDiag(d,
			common.ConvertExpected401ErrInto404Err(
				common.ConvertExpected400ErrInto404Err(err, "error_code", trustedServiceNotFoundErrCodes...),
				"error_code",
				organizationNotFoundErrCodes...,
			),
			"error retrieving trusted service",
		)
	}
	// ...
}
```

根据API返回的错误结构，错误码路径字段名可能不同：

```go
// error_code 路径（最常见）
common.ConvertExpected400ErrInto404Err(err, "error_code", errCodes...)

// error.code 路径（嵌套结构）
common.ConvertExpected400ErrInto404Err(err, "error.code", errCodes...)

// code 路径（简单结构）
common.ConvertExpected400ErrInto404Err(err, "code", errCodes...)

// errCode 路径
common.ConvertExpected403ErrInto404Err(err, "errCode", errCodes...)

// errors|[0].errorCode 路径（数组结构）
common.ConvertExpected400ErrInto404Err(err, "errors|[0].errorCode", errCodes...)
```

**错误转换在子方法中的使用场景**：

错误类型转换可以在子方法中进行，特别是在以下场景：

1. **查询方法中统一错误类型**：在查询方法中进行错误转换，便于父方法统一处理

```go
// 子方法中进行错误转换
func GetMicroserviceEngineById(client *golangsdk.ServiceClient, engineId, epsId string) (interface{}, error) {
	// ... API调用
	requestResp, err := client.Request("GET", getPath, &getOpt)
	if err != nil {
		// 在子方法中进行错误类型转换，统一错误类型
		return nil, common.ConvertExpected400ErrInto404Err(
			common.ConvertExpected401ErrInto404Err(err, "error_code", engineNotFoundCodes...),
			"error_code",
			engineNotFoundCodes...,
		)
	}
	return utils.FlattenResponse(requestResp)
}

// 父方法中使用转换后的错误
func resourceMicroserviceEngineRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	respBody, err := GetMicroserviceEngineById(client, engineId, enterpriseProjectId)
	if err != nil {
		// 父方法中进行错误格式化（使用CheckDeletedDiag）
		return common.CheckDeletedDiag(d, err, "error retrieving microservice engine")
	}
	// ...
}
```

2. **状态刷新函数中判断资源是否存在**：在状态刷新函数中进行错误转换，用于判断资源是否已删除

```go
// 状态刷新函数中进行错误转换
func refreshMicroserviceEngineJobFunc(client *golangsdk.ServiceClient, engineId, jobId, enterpriseProjectId string,
	targets []string) resource.StateRefreshFunc {
	return func() (interface{}, string, error) {
		resp, err := getMicroserviceEngineJob(client, engineId, jobId, enterpriseProjectId)
		if err != nil {
			// 在状态刷新函数中进行错误类型转换，用于判断资源是否存在
			parsedErr := common.ConvertExpected400ErrInto404Err(
				common.ConvertExpected401ErrInto404Err(err, "error_code", engineNotFoundCodes...),
				"error_code",
				engineNotFoundCodes...,
			)
			if _, ok := parsedErr.(golangsdk.ErrDefault404); ok && len(targets) < 1 {
				// 资源不存在且targets为空（删除操作），返回COMPLETED
				return "RESOURCE_NOT_FOUND", "COMPLETED", nil
			}
			return nil, "ERROR", parsedErr
		}
		// ... 状态判断逻辑
	}
}
```

**检查清单**：

- [ ] 非标准404错误是否使用`ConvertExpectedXXXErrInto404Err()`进行转换？
- [ ] 错误转换是否在合适的位置进行（可以在子方法中进行，也可以在父方法中进行）？
- [ ] 错误码列表是否定义为全局变量（位于资源主函数上方）？
- [ ] 错误码变量命名是否符合规范（`{resourceName}NotFoundErrCodes` 或 `v{VersionNumber}{ResourceName}NotFoundErrCodes`）？
- [ ] 错误码路径字段名是否根据API实际返回结构确定？
- [ ] 嵌套调用时是否正确处理了资源本身和父资源不存在的错误？
- [ ] 转换后的错误是否通过`CheckDeletedDiag()`统一处理（在父方法中）？

#### 4.4 资源不存在不报错但详情状态标记为诸如DELETED等枚举值的情况

当查询接口返回成功，但资源状态为`DELETED`（也可能存在其他代表着资源不存在的枚举值）时，**必须**手动构造404错误，**禁止**以该状态提示超时报错。

**设计原则**：

- 所有查询操作都应该使用`CheckDeletedDiag()`处理404错误
- 如果资源可能在创建后短暂返回404，使用重试机制
- 重试逻辑应该封装在查询方法中，而不是在ReadContext中
- 重试间隔通常设置为10秒，超时时间使用`d.Timeout(schema.TimeoutXXX)`，注意获取正确步骤的超时时间，如：schema.TimeoutRead
- 如果资源状态字段显示为DELETED，需要手动构造404错误
- 错误消息应该清晰、一致，使用统一的格式
- 使用`golangsdk.ErrDefault404{}`手动构造404错误时，确保错误类型正确

```go
func resourcePrivateCARead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	getRespBody, err := readPrivateCA(client, d)
	if err != nil {
		return common.CheckDeletedDiag(d, err, "error retrieving CCM private CA")
	}

	status := utils.PathSearch("status", getRespBody, nil).(string)
	if status == "DELETED" {
		// 资源状态为DELETED时，手动构造404错误
		return common.CheckDeletedDiag(d, golangsdk.ErrDefault404{}, "error retrieving private CA")
	}
	// ...
}
```

**检查清单**：

- [ ] ReadContext方法中是否使用`CheckDeletedDiag()`处理404错误？
- [ ] 如果资源可能在创建后短暂返回404，是否实现了重试机制？
- [ ] 重试逻辑是否封装在查询方法中？
- [ ] 错误消息格式是否符合规范（小写开头，包含操作上下文）？
- [ ] 如果资源状态为DELETED，是否手动构造了404错误？

**错误处理规范总体检查清单**：

- [ ] CRUD主方法是否使用`diag.Diagnostics`作为返回类型？
- [ ] 子方法是否返回`(interface{}, error)`或`([]interface{}, error)`，不在子方法中格式化错误？
- [ ] 错误格式化是否在父方法（逻辑调用层）进行？
- [ ] 单个错误是否使用`diag.Errorf()`格式化？
- [ ] 多个错误是否使用`multierror.Append()`和`diag.FromErr()`处理？
- [ ] 错误消息格式是否使用小写开头（`"error creating..."`）？
- [ ] 错误消息是否包含操作上下文（`"error creating agency: %s"`）？
- [ ] 是否禁止在子方法向上传递的中间路径破坏错误的类型？
- [ ] ReadContext方法中是否使用`CheckDeletedDiag()`处理404错误？
- [ ] 如果资源可能在创建后短暂返回404，是否实现了重试机制？
- [ ] 非标准404错误是否使用`ConvertExpectedXXXErrInto404Err()`进行转换？
- [ ] 错误码列表是否定义为全局变量（位于资源主函数上方）？
- [ ] 错误码变量命名是否符合规范（`{resourceName}NotFoundErrCodes`或`v{VersionNumber}{ResourceName}NotFoundErrCodes`）？
- [ ] 错误码路径字段名是否根据API实际返回结构确定？
- [ ] 嵌套调用时是否正确处理了资源本身和父资源不存在的错误？
- [ ] 如果资源状态为DELETED，是否手动构造了404错误？

### 5. 包的导入按照稳定度排序

该部分与以下 Skill 内容重复，已移除并统一以 Skill 文档为准：

- `02-resource-coding.md`（资源编码技能）中的“调整包的顺序”
- `03-datasource-coding.md`（数据源编码技能）中的“调整包的顺序”

如需检查 import 分组与排序，请直接参见上述两个 Skill。

### 6. HTTP请求封装

HTTP 请求的构造需要遵循统一的封装模式，确保路径构建、请求头配置和请求体处理的规范性。

**设计原则**：

- **路径构建**：使用 `httpUrl` 定义 API 路径（URI 中首个斜杠后的内容），通过 `client.Endpoint + httpUrl` 拼接完整 URL，使用 `strings.ReplaceAll` 替换路径占位符（如 `{project_id}`、`{engine_id}`等，务必确保所有的占位符都在后续的逻辑中一一替换）
- **请求选项**：使用 `golangsdk.RequestOpts` 配置 `KeepResponseBody: true`、`MoreHeaders` 和 `JSONBody`
- **请求体处理**：如果请求存在请求体内容，则JSON 请求体需使用 `utils.RemoveNil` 清理 nil 值，避免 API 拒绝或产生歧义
- **请求头抽象**：将 `Content-Type`（必须提供）、`Accept`等通用请求头和API要求的特定请求头通过`golangsdk.RequestOpts`的`MoreHeaders`等传入（如无特殊请求头要求则传入通用请求头即可）请求中，如存在较为复杂的逻辑处理，也可以将这些逻辑抽象为 `buildRequestMoreHeaders` 等辅助函数
- **前置依赖**：若请求体依赖其他 API 的返回（如 VPC、子网信息），应在发送主请求前完成前置查询，并将结果传入请求体构建方法
- **请求状态码**：若API中声明的正常状态码不在代码默认请求中状态码列表中时，则需要通过`golangsdk.RequestOpts`的`OkCodes`传入（各类型请求包含的默认状态码如下：），禁止重复声明属于默认状态码列表中的状态码信息。
  - **GET**, **HEAD**: `200`
  - **POST**: `200`, `201`, `202`
  - **PUT**: `200`, `201`, `202`
  - **PATCH**: `200`, `202`, `204`
  - **DELETE**: `200`, `202`, `204`

**最佳实践**：

**1. 标准项目级请求**

适用于采用IAM鉴权（URI中包含 `project_id`）且无特殊状态码需要处理的API

```go
// 标准DELETE接口，没有请求体且MoreHeaders仅需要传入Content-Type，接口返回的状态码为204，属于默认状态码，无需额外定义OkCodes
func deleteV3Application(client *golangsdk.ServiceClient, applicationId string) error {
	httpUrl := "v3/{project_id}/cas/applications/{application_id}"
	deletePath := client.Endpoint + httpUrl
	deletePath = strings.ReplaceAll(deletePath, "{project_id}", client.ProjectID)
	deletePath = strings.ReplaceAll(deletePath, "{application_id}", appId)

	opt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	_, err = client.Request("DELETE", deletePath, &opt)
	return err
}

func resourceV3ApplicationDelete(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg           = meta.(*config.Config)
		region        = cfg.GetRegion(d)
		applicationId = d.Id()
	)

	client, err := cfg.NewServiceClient("servicestage", region)
	if err != nil {
		return diag.Errorf("error creating ServiceStage client: %s", err)
	}

	_, err = deleteV3Application(client, applicationId)
	if err != nil {
		return common.CheckDeletedDiag(d, common.ConvertExpected401ErrInto404Err(err, "error_code", v3AppNotFoundCodes...),
			fmt.Sprintf("error deleting application (%s)", applicationId))
	}

	return nil
}
```

**2. 项目级请求，但存在特殊的状态码需要处理**

适用于采用IAM鉴权（URI中包含 `project_id`）且存在非默认请求状态码需要处理的API

```go
func buildCreateUserPasswordResetRequestBody(d *schema.ResourceData) map[string]interface{} {
	return map[string]interface{}{
		"new_password": d.Get("new_password"),
	}
}

// 包含请求体的PUT接口，MoreHeaders仅需要传入Content-Type但接口返回的状态码为204，不属于PUT请求的默认状态码，需额外定义OkCodes
func createUserPasswordReset(client *golangsdk.ServiceClient, d *schema.ResourceData) error {
	httpUrl := "v2/{project_id}/instances/{instance_id}/users/{user_name}"

	createPath := client.Endpoint + httpUrl
	createPath := strings.ReplaceAll(httpUrl, "{project_id}", client.ProjectID)
	createPath = strings.ReplaceAll(createPath, "{instance_id}", d.Get("instance_id").(string))
	createPath = strings.ReplaceAll(createPath, "{user_name}", d.Get("user_name").(string))

	opt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
		JSONBody: utils.RemoveNil(buildCreateUserPasswordResetRequestBody(d)),
		OkCodes:  []int{204},
	}

	_, err = client.Request("PUT", createPath, &opt)
	return err
}

func resourceUserPasswordResetCreate(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg        = meta.(*config.Config)
		instanceId = d.Get("instance_id").(string)
		userName   = d.Get("user_name").(string)
	)
	client, err := cfg.NewServiceClient("dmsv2", cfg.GetRegion(d))
	if err != nil {
		return diag.Errorf("error creating DMS client: %s", err)
	}

	err = createUserPasswordReset(client, d)
	if err != nil {
		return diag.Errorf("error resetting password for user (%s) under instance (%s): %s", userName, instanceId, err)
	}

	randUUID, err := uuid.GenerateUUID()
	if err != nil {
		return diag.Errorf("unable to generate resource ID: %s", err)
	}
	d.SetId(randUUID)

	return nil
}
```

**3. 项目级请求，包含特殊的请求头**

适用于采用IAM鉴权（URI中包含 `project_id`）且请求体内容需要通过多个接口查询进行组装，请求头内容较默认内容多了一部分输入（这里为`Accept`和`X-Enterprise-Project-ID`）

```go
// CSE服务要求请求头中除了Content-Type必须设值成“application/json;charset=UTF-8”外还要求必须携带Accept和X-Enterprise-Project-ID，故将该逻辑处理抽象成一个单独的方法
func buildRequestMoreHeaders(enterpriseProjectId string) map[string]string {
	moreHeaders := map[string]string{
		"Content-Type": "application/json;charset=UTF-8",
		"Accept":       "application/json",
	}

	if enterpriseProjectId != "" {
		moreHeaders["X-Enterprise-Project-ID"] = enterpriseProjectId
	}
	return moreHeaders
}

// ...

// 创建请求的请求体构造需要依赖其他查询接口返回的响应体字段值（如subnet名称、子网CIDR需要根据子网ID通过“查询子网详情”接口获取，且该接口中仅包含VPC ID，没有VPC的名称，故
// 需要额外再通过子网返回的所属VPC ID通过“查询VPC详情”接口获取VPC的详细信息，其中包括名称）
func buildCreateMicroserviceEngineRequestBody(d *schema.ResourceData, vpcInfo, subnetInfo interface{}) map[string]interface{} {
	authType := d.Get("auth_type").(string)
	return map[string]interface{}{
		"name":        d.Get("name").(string),
		"description": utils.ValueIgnoreEmpty(d.Get("description").(string)),
		"payment":     "1", // PostPaid
		"flavor":      d.Get("flavor").(string),
		"azList":      utils.ExpandToStringListBySet(d.Get("availability_zones").(*schema.Set)),
		"authType":    authType,
		"vpc":         utils.PathSearch("vpc.name", vpcInfo, ""),         // 从查询到的VPC详情中获取VPC的名称
		"vpcId":       utils.PathSearch("subnet.vpc_id", subnetInfo, ""), // 从查询到的子网详情中获取其所属的VPC的ID
		"networkId":   d.Get("network_id").(string),
		"subnetCidr":  utils.PathSearch("subnet.cidr", subnetInfo, ""),   // 从查询到的子网详情中获取子网的CIDR
		"publicIpId":  utils.ValueIgnoreEmpty(d.Get("eip_id").(string)),
		"auth_cred":   buildAuthCred(authType, d.Get("admin_pass").(string)),
		"specType":    d.Get("version").(string),
		"inputs":      d.Get("extend_params").(map[string]interface{}),
	}
}

// 接口请求返回状态码为200，属于默认请求状态码，无需定义OkCodes
func createMicroserviceEngine(cseClient, vpcClient *golangsdk.ServiceClient, d *schema.ResourceData,
	enterpriseProjectId string) (interface{}, error) {
	networkId := d.Get("network_id").(string)
	subnetInfo, err := getSubnetById(vpcClient, networkId)
	if err != nil {
		return nil, err
	}
	vpcInfo, err := getVpcById(vpcClient, utils.PathSearch("subnet.vpc_id", subnetInfo, "").(string))
	if err != nil {
		return nil, err
	}

	httpUrl := "v2/{project_id}/enginemgr/engines"
	createPath := cseClient.Endpoint + httpUrl
	createPath = strings.ReplaceAll(createPath, "{project_id}", cseClient.ProjectID)

	createOpts := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders:      buildRequestMoreHeaders(enterpriseProjectId),
		JSONBody:         utils.RemoveNil(buildCreateMicroserviceEngineRequestBody(d, vpcInfo, subnetInfo)),
	}

	requestResp, err := cseClient.Request("POST", createPath, &createOpts)
	if err != nil {
		return nil, err
	}
	return utils.FlattenResponse(requestResp)
}
```

**4. 非IAM项目请求且URI中使用某固定值作为项目ID**

部分请求URI使用的是非IAM项目且部分场景下请求头中包含服务自定义的鉴权信息而非IAM鉴权信息

```go
// 包级（全局）变量：CSE 微服务注册中心使用固定 project_id（CSE服务的部分资源不使用IAM鉴权，故这里的project非IAM project）
var microserviceDefaultProjectId = "default"

// 接口请求返回状态码为200，属于默认请求状态码，无需定义OkCodes
func createMicroservice(client *golangsdk.ServiceClient, d *schema.ResourceData) (interface{}, error) {
	httpUrl := "v4/{project_id}/registry/microservices"
	createPath := client.Endpoint + httpUrl
	createPath = strings.ReplaceAll(createPath, "{project_id}", microserviceDefaultProjectId)

	createOpts := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders:      buildRequestMoreHeaders(client.ProjectID),
		JSONBody:         buildMicroserviceCreateOpts(d),
	}

	// When a user configures both the `admin_user` and `admin_pass` fields, it indicates that the microservice engine
	// has enabled RBAC authentication. Subsequent requests will require the token information obtained via the
	// `GET /v4/token` interface.
	token, err := GetAuthorizationToken(getAuthAddress(d), d.Get("admin_user").(string), d.Get("admin_pass").(string))
	if err != nil {
		return nil, err
	}
	// 如果微服务启用了RBAC鉴权则需要根据鉴权用户信息（admin_user和admin_pass）获取token并在请求头中替换原有的IAM鉴权信息（AKSK鉴权）
	// If the microservice has RBAC authentication enabled, the Authorization header will use a special token provided
	// by the CSE service to replace the original IAM authentication information (AKSK authentication) in the request
	// header.
	if token != "" {
		createOpts.MoreHeaders["Authorization"] = token
	}

	requestResp, err := client.Request("POST", createPath, &createOpts)
	if err != nil {
		return nil, err
	}
	return utils.FlattenResponse(requestResp)
}
```

**检查清单**：

- [ ] 是否使用 `httpUrl` + `client.Endpoint` 构建完整请求路径？
- [ ] 路径占位符是否通过 `strings.ReplaceAll` 正确替换？
- [ ] 是否设置 `KeepResponseBody: true` 以便解析响应？
- [ ] 请求体是否使用 `utils.RemoveNil` 清理 nil 值（或 build 方法内部已处理）？
- [ ] 请求头是否都包含 `Content-Type` ？
- [ ] 非标且逻辑设计复杂的请求头是否通过辅助函数（如 `buildRequestMoreHeaders`）进行构建？
- [ ] 若需固定 `project_id` 或条件性 `Authorization`，是否在代码中定义对应的常量并通过注释说明设计原因或注释使用条件性 `Authorization` 的原因？

### 7. 状态轮询

当创建、删除或更新等异步接口返回任务 `ID/名称`（或订单 `ID/名称` 等）时，需要基于该 `ID/名称` 轮询查询接口，直到任务完成或失败。轮询逻辑应遵循统一的设计模式，确保错误处理、超时控制和状态判断的规范性。

**设计原则**：

- **方法抽象**：将状态刷新逻辑抽象为 `resource.StateRefreshFunc` 类型的独立函数，命名格式为：
  - 如果轮询是基于某任务ID或订单ID，则方法命名格式为：`get{ResourceName}{Job/Task}By{Id/Name}`
  - 如果轮询是基于资源对象查询详情接口的状态信息，则方法命名格式为： `refresh{ResourceName}{Job/Task}Func` 或 `{ResourceName}StateRefreshFunc`，如果需要包含版本号则在资源名称前进行补充
- **错误转换**：使用 `common.ConvertExpected400ErrInto404Err`、`common.ConvertExpected401ErrInto404Err` 等将非标错误码（代表资源不存在）转为 404，便于统一处理资源不存在
- **失败状态检查**：从 API 响应中提取状态字段，若为失败状态（如 `CreateFail`、`DeleteFailed`），返回 `"ERROR"` 并附带明确错误信息
- **targets 参数**：当轮询采用的是资源状态轮询时，需要在轮询方法中设计用于指定目标值的入参（targets []string），以供创建/更新场景传入目标状态列表（如 `[]string{"Finished"}`）；删除场景传 `nil`
- **返回值约定**：刷新函数返回 `(interface{}, string, error)`，第二个返回值必须为 `"PENDING"`、`"COMPLETED"`、`"ERROR"` 之一
  - 当查询请求返回错误时返回`ERROR`作为状态字符串（如果是通过资源状态进行轮询且函数输入的targets长度小于1时（说明此时为删除阶段），则返回`COMPLETED`作为状态字符串，且interface{}必须返回`RESOURCE_NOT_FOUND`、`NOT_FOUND`等标记资源不存在的字符串（不能返回请求的Response，因为其大概率为nil）
  - 当查询请求响应中的特定检查字段属于异常值时（通常会根据API的描述信息将其中异常的枚举值提取出来进行黑名单判定）直接返回`ERROR`作为状态字符串
  - 当查询请求响应中的特定检查字段属于目标值时，返回`COMPLETED`作为状态字符串
  - 其余情况返回`PENDING`作为状态字符串
- **超时与间隔**：使用与阶段对应的超时时间作为轮询的超时时间值 （`d.Timeout(schema.TimeoutCreate/Read/Update/Delete)`）；根据任务耗时设置 `Delay`（首次轮询前等待）和 `PollInterval`（轮询间隔）
- **默认超时时间**：根据使用轮询的阶段，为其在主函数中配置合理默认超时时间（根据实际业务的执行时间长短设置2-3倍的默认时间）
- **ID 设置时机**：采用任务进度轮询或资源状态轮询时，需在发起轮询**之前**调用 `d.SetId()` 设置资源 ID，以便轮询失败时 Terraform 能正确标记资源状态
- **状态检查顺序**：资源状态轮询时，应先进行**黑名单检查**（失败状态，如 `CreateFail`、`DeleteFailed`），再进行**目标状态检查**（白名单），避免将失败状态误判为完成
- **包周期订单**：当 `charging_mode` 或 `changing_mode` 为 `PrePaid` 时，应使用 `common.WaitOrderComplete` 和 `common.WaitOrderResourceComplete` 进行订单轮询，通过 `cfg.BssV2Client(region)` 获取 BSS 客户端
- **日志记录**：轮询开始前可以使用 `log.Printf("[DEBUG] Waiting for ...")` 记录轮询意图与关键 ID，便于排查问题

**最佳实践**

**1. 通过跟踪任务进度的方式进行轮询**

通常创建接口返回 `id`（资源 ID，也可能是其他名称，如`reousce_id`，甚至可能处于某结构体的嵌套中）和 `jobId`（任务 ID，也可能是其他名称，如`job_id`，甚至可能处于某结构体的嵌套中），在保存完资源的ID后进行轮询。
而删除接口通常只返回 `jobId`（任务 ID，也可能是其他名称，如`job_id`，甚至可能处于某结构体的嵌套中），请求完成后立刻进行轮询。
通过设计一个专用于查询任务状态的方法（如微服务引擎的 `getMicroserviceEngineJobById` 查询微服务引擎任务状态方法），当任务目标为 `Finished` 时表示资源任务已执行完毕（创建、查询、更新或删除）。

```go
func ResourceMicroserviceEngine() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceMicroserviceEngineCreate,
		ReadContext:   resourceMicroserviceEngineRead,
		UpdateContext: resourceMicroserviceEngineUpdate,
		DeleteContext: resourceMicroserviceEngineDelete,

		Importer: &schema.ResourceImporter{
			StateContext: resourceEngineImportState,
		},

		Timeouts: &schema.ResourceTimeout{
			Create: schema.DefaultTimeout(60 * time.Minute),
			Delete: schema.DefaultTimeout(20 * time.Minute),
		},
		// ...
	}
}

// ...

func getMicroserviceEngineJobById(client *golangsdk.ServiceClient, engineId, jobId, enterpriseProjectId string) (interface{}, error) {
	httpUrl := "v2/{project_id}/enginemgr/engines/{engine_id}/jobs/{job_id}"
	getPath := client.Endpoint + httpUrl
	getPath = strings.ReplaceAll(getPath, "{project_id}", client.ProjectID)
	getPath = strings.ReplaceAll(getPath, "{engine_id}", engineId)
	getPath = strings.ReplaceAll(getPath, "{job_id}", jobId)

	getOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders:      buildRequestMoreHeaders(enterpriseProjectId),
	}
	requestResp, err := client.Request("GET", getPath, &getOpt)
	if err != nil {
		return nil, err
	}
	return utils.FlattenResponse(requestResp)
}

// refreshMicroserviceEngineJobFunc returns a state refresh function for polling the microservice engine job status.
// It handles error conversion (converting 400/401 errors to 404 when the engine is not found),
// checks for failure statuses, and determines if the job has reached the target status.
// The targets parameter specifies the list of status values that indicate completion.
// If targets is empty and a 404 error occurs, it returns COMPLETED (used for delete operations).
func refreshMicroserviceEngineJobFunc(client *golangsdk.ServiceClient, engineId, jobId, enterpriseProjectId string,
	targets []string) resource.StateRefreshFunc {
	return func() (interface{}, string, error) {
		resp, err := getMicroserviceEngineJobById(client, engineId, jobId, enterpriseProjectId)
		if err != nil {
			parsedErr := common.ConvertExpected400ErrInto404Err(
				common.ConvertExpected401ErrInto404Err(err, "error_code", microserviceEngineNotFoundCodes...),
				"error_code",
				microserviceEngineNotFoundCodes...,
			)
			if _, ok := parsedErr.(golangsdk.ErrDefault404); ok && len(targets) < 1 {
				return "RESOURCE_NOT_FOUND", "COMPLETED", nil
			}
			return nil, "ERROR", parsedErr
		}

		status := utils.PathSearch("status", resp, "").(string)
		// 先进行状态的黑名单检查
		if utils.StrSliceContains([]string{"CreateFail", "DeleteFailed", "UpgradeFailed", "ModifyFailed"}, status) {
			return resp, "ERROR", fmt.Errorf("unexpect status (%s)", status)
		}

		// 再进行目标状态的检查
		if utils.StrSliceContains(targets, status) {
			return resp, "COMPLETED", nil
		}
		return resp, "PENDING", nil
	}
}

func resourceMicroserviceEngineCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	
	createOpts, err := createMicroserviceEngine(cseClient, vpcClient, d, enterpriseProjectId)
	if err != nil {
		return diag.Errorf("error creating microservice engine: %s", err)
	}

	engineId := utils.PathSearch("id", createOpts, "").(string)
	if engineId == "" {
		return diag.Errorf("unable to find the microservice engine ID from the API response")
	}
	// 在创建请求完成后（轮询前）设置资源ID
	d.SetId(engineId)

	// 创建场景：传入 targets 为 []string{"Finished"}
	log.Printf("[DEBUG] Waiting for the microservice engine to become running, the engine ID is %s", engineId)
	stateConf := &resource.StateChangeConf{
		Pending: []string{"PENDING"},
		Target:  []string{"COMPLETED"},
		Refresh: refreshMicroserviceEngineJobFunc(cseClient, engineId,
			strconv.Itoa(int(utils.PathSearch("jobId", createOpts, float64(0)).(float64))), enterpriseProjectId, []string{"Finished"}),
		Timeout:      d.Timeout(schema.TimeoutCreate), // 创建阶段使用创建阶段的超时时间
		Delay:        180 * time.Second,
		PollInterval: 15 * time.Second,
	}
	_, err = stateConf.WaitForStateContext(ctx)
	if err != nil {
		return diag.Errorf("error waiting for the creation of microservice engine (%s) to complete: %s", engineId, err)
	}

	return resourceMicroserviceEngineRead(ctx, d, meta)
}

// ...

func resourceMicroserviceEngineDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...

	resp, err := deleteMicroserviceEngine(client, engineId, enterpriseProjectId)
	if err != nil {
		return common.CheckDeletedDiag(d, common.ConvertExpected400ErrInto404Err(
			common.ConvertExpected401ErrInto404Err(err, "error_code", microserviceEngineNotFoundCodes...),
			"error_code",
			microserviceEngineNotFoundCodes...,
		), fmt.Sprintf("error deleting microservice engine (%s)", engineId))
	}

	// 删除场景：传入 targets 为 nil，404 时返回 COMPLETED
	log.Printf("[DEBUG] Waiting for the Microservice engine delete complete, the engine ID is %s.", engineId)
	stateConf := &resource.StateChangeConf{
		Pending: []string{"PENDING"},
		Target:  []string{"COMPLETED"},
		Refresh: refreshMicroserviceEngineJobFunc(client, engineId,
			strconv.Itoa(int(utils.PathSearch("jobId", resp, float64(0)).(float64))), enterpriseProjectId, nil),
		Timeout:      d.Timeout(schema.TimeoutDelete),
		Delay:        120 * time.Second,
		PollInterval: 15 * time.Second,
	}
	_, err = stateConf.WaitForStateContext(ctx)
	if err != nil {
		return diag.Errorf("error waiting for the deletion of microservice engine (%s) to complete: %s", engineId, err)
	}

	return nil
}
```

**2. 通过跟踪资源状态的方式进行轮询**

部分产品的创建、更新、删除接口不返回异步任务的相关信息，这就要求代码中必须通过不断地调用查询资源详情接口去跟踪资源状态来确认资源的操作是否完成。

```go
func ResourceApigInstance() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceInstanceCreate,
		ReadContext:   resourceInstanceRead,
		UpdateContext: resourceInstanceUpdate,
		DeleteContext: resourceInstanceDelete,

		Importer: &schema.ResourceImporter{
			StateContext: schema.ImportStatePassthroughContext,
		},

		Timeouts: &schema.ResourceTimeout{
			Create: schema.DefaultTimeout(40 * time.Minute),
			// ...
		},
	}
}

// ...

func instanceStateRefreshFunc(client *golangsdk.ServiceClient, instanceId string, targets []string) resource.StateRefreshFunc {
	return func() (interface{}, string, error) {
		respBody, err := QueryInstanceDetail(client, instanceId)
		if err != nil {
			if _, ok := err.(golangsdk.ErrDefault404); ok && len(targets) < 1 {
				return "not_found", "COMPLETED", nil
			}
			return respBody, "ERROR", err
		}

		statusResp := utils.PathSearch("status", respBody, "").(string)
		if utils.StrSliceContains([]string{"CreateFail", "InitingFailed", "RegisterFailed", "InstallFailed",
			"UpdateFailed", "RollbackFailed", "UnRegisterFailed", "DeleteFailed", "RestartFail", "ResizeFailed"},
			statusResp) {
			return respBody, "ERROR", fmt.Errorf("unexpect status (%s)", statusResp)
		}

		if utils.StrSliceContains(targets, statusResp) {
			return respBody, "COMPLETED", nil
		}
		return "continue", "PENDING", nil
	}
}

func resourceInstanceCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	d.SetId(instanceId)

	// 设置完ID后等待一分钟，随后执行实例资源的状态轮询
	stateConf := &resource.StateChangeConf{
		Pending:      []string{"PENDING"},
		Target:       []string{"COMPLETED"},
		Refresh:      instanceStateRefreshFunc(client, instanceId, []string{"Running"}), // Running为目标状态
		Timeout:      d.Timeout(schema.TimeoutCreate),
		Delay:        1 * time.Minute,
		PollInterval: 20 * time.Second,
	}
	_, err = stateConf.WaitForStateContext(ctx)
	if err != nil {
		return diag.Errorf("error waiting for the status of dedicated instance (%s) to become running: %s", instanceId, err)
	}

	if v, ok := d.GetOk("custom_ingress_ports"); ok {
		if err := addCustomIngressPorts(client, instanceId, v.(*schema.Set).List()); err != nil {
			return diag.FromErr(err)
		}
	}

	return resourceInstanceRead(ctx, d, meta)
}
```

**3. 跟踪包周期订单状态**

部分资源支持包周期的计费模式，在创建、更新、删除时需要通过不断地查询CBC订单以跟踪资源的操作是否完成（是否按照包周期轮询逻辑对资源状态进行跟踪的判断条件是：`changing_mode` 的参数值是否等于 `PrePaid`）
订单轮询已通过公共方法实现，但需要将三个步骤依次组合成新的方法（主要是要根据当前资源定制错误内容）：
- 构造CBC（BSS）的客户端（config中的 `BssV2Client` 方法）
- 等待订单完成（common包下的 `WaitOrderComplete` 方法）
- 等待CBC侧生成资源ID（部分资源会预设资源ID，则此步骤返回的资源ID可以忽略）（common包下的 `WaitOrderResourceComplete` 方法）

```go
func waitForPrePaidServerComplete(ctx context.Context, cfg *config.Config, region string, timeOut time.Duration, orderId string) (string, error) {
	if orderId == "" {
		return "", fmt.Errorf("unable to find order ID from API response")
	}

	bssClient, err := cfg.BssV2Client(region)
	if err != nil {
		return "", fmt.Errorf("error creating BSS v2 client: %s", err)
	}

	if err := common.WaitOrderComplete(ctx, bssClient, orderId, timeOut); err != nil {
		return "", err
	}

	resourceId, err := common.WaitOrderResourceComplete(ctx, bssClient, orderId, timeOut)
	if err != nil {
		return "", fmt.Errorf("error waiting for Workspace APP server order (%s) complete: %s", orderId, err)
	}

	return resourceId, nil
}

func waitForAppServerJobCompleted(ctx context.Context, client *golangsdk.ServiceClient, timeout time.Duration, jobId string) (interface{},
	error) {
	stateConf := &resource.StateChangeConf{
		Pending:      []string{"RUNNING"},
		Target:       []string{"SUCCESS"},
		Refresh:      refreshAppServerJobStatusFunc(client, jobId),
		Timeout:      timeout,
		Delay:        10 * time.Second,
		PollInterval: 30 * time.Second,
	}

	serverResp, err := stateConf.WaitForStateContext(ctx)
	return serverResp, err
}

func refreshAppServerJobStatusFunc(client *golangsdk.ServiceClient, jobId string) resource.StateRefreshFunc {
	return func() (interface{}, string, error) {
		httpUrl := "v2/{project_id}/job/{job_id}"
		getJobPath := client.Endpoint + httpUrl
		getJobPath = strings.ReplaceAll(getJobPath, "{project_id}", client.ProjectID)
		getJobPath = strings.ReplaceAll(getJobPath, "{job_id}", jobId)
		getOpt := golangsdk.RequestOpts{
			KeepResponseBody: true,
		}

		resp, err := client.Request("GET", getJobPath, &getOpt)
		if err != nil {
			return resp, "ERROR", err
		}

		respBody, err := utils.FlattenResponse(resp)
		if err != nil {
			return resp, "ERROR", err
		}

		return respBody, utils.PathSearch("status", respBody, nil).(string), nil
	}
}

func resourceAppServerCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...

	if d.Get("charging_mode").(string) == "prePaid" {
		orderId := utils.PathSearch("order_id", respBody, "").(string)
		resourceId, err := waitForPrePaidServerComplete(ctx, cfg, region, d.Timeout(schema.TimeoutCreate), orderId)
		if err != nil {
			return diag.FromErr(err)
		}
		d.SetId(resourceId)
	} else {
		jobId := utils.PathSearch("job_id", respBody, "").(string)
		if jobId == "" {
			return diag.Errorf("unable to find job ID from API response")
		}

		serverResp, err := waitForAppServerJobCompleted(ctx, client, d.Timeout(schema.TimeoutCreate), jobId)
		if err != nil {
			return diag.Errorf("error waiting for the job (%s) completed: %s", jobId, err)
		}

		serverId := utils.PathSearch("sub_jobs|[0].job_resource_info.resource_id", serverResp, "").(string)
		if serverId == "" {
			return diag.Errorf("unable to find server ID from API response")
		}

		d.SetId(serverId)
	}

	if err := updateAppServer(client, d, d.Id()); err != nil {
		return diag.Errorf("error updating the server (%s): %s", d.Id(), err)
	}

	return resourceAppServerRead(ctx, d, meta)
}
```

**检查清单**：

**1. 通用项**：

- [ ] 状态刷新函数是否抽象为独立的 `resource.StateRefreshFunc`？
- [ ] 返回值是否符合约定（PENDING/COMPLETED/ERROR，或与 StateChangeConf 的 Pending/Target 匹配的实际状态）？
- [ ] `Timeout` 是否使用 `d.Timeout(schema.TimeoutCreate/Read/Update/Delete)`？
- [ ] `Delay` 和 `PollInterval` 是否根据任务耗时合理设置？
- [ ] 主函数是否配置了与轮询阶段对应的默认超时时间？
- [ ] 轮询开始前是否使用 `log.Printf("[DEBUG] Waiting for ...")` 记录？
- [ ] `WaitForStateContext` 返回的 `err` 是否已格式化并返回 `diag.Diagnostics`？

**2. 任务进度轮询**（创建/删除接口返回 jobId 等任务 ID）：

- [ ] 是否设计专用的任务状态查询方法（如 `get{ResourceName}JobById`）？
- [ ] 是否检查 API 返回的失败状态（黑名单）并返回明确错误？
- [ ] 创建/更新场景是否传入正确的 `targets` 列表（如 `[]string{"Finished"}`）？
- [ ] 删除场景是否传入 `targets` 为 `nil`，并在 404 时返回 `COMPLETED`？
- [ ] 是否在轮询**之前**调用 `d.SetId()` 设置资源 ID？
- [ ] 任务 ID 提取与类型转换是否正确（如 `jobId` 的 float64 → int → string）？
- [ ] 是否使用 `ConvertExpected*ErrInto404Err` 处理非标错误？

**3. 资源状态轮询**（通过查询资源详情接口跟踪状态）：

- [ ] 是否通过查询资源详情接口（如 `QueryInstanceDetail`）进行轮询？
- [ ] 是否先进行黑名单检查（失败状态），再进行目标状态检查？
- [ ] 创建/更新场景是否传入正确的 `targets` 列表（如 `[]string{"Running"}`）？
- [ ] 删除场景是否传入 `targets` 为 `nil`，并在 404 时返回 `COMPLETED`？
- [ ] 是否在轮询**之前**调用 `d.SetId()` 设置资源 ID？

**4. 包周期订单轮询**（`charging_mode`/`changing_mode` 为 `PrePaid`）：

- [ ] 是否通过 `charging_mode` 或 `changing_mode == "PrePaid"` 判断需走订单轮询？
- [ ] 是否使用 `cfg.BssV2Client(region)` 获取 BSS 客户端？
- [ ] 是否依次调用 `common.WaitOrderComplete` 和 `common.WaitOrderResourceComplete`？
- [ ] 订单 ID 是否从创建响应中正确提取？

### 8. 分页查询处理

分页查询方法的设计需要遵循统一的标准，确保代码的一致性和可维护性。根据不同的分页参数类型，分为以下三种模式：page分页、marker分页和offset分页。

**参考条件**：

- 当 API 接口涉及分页参数（如 `page`/`limit`/`offset`/`marker`/`per_page` 等），但对应的查询函数/方法中**并未设计分页遍历逻辑**（仅请求单页或遗漏翻页处理）时，应参考本节补齐分页处理。

**设计原则**：

- **方法抽象**：所有分页查询方法必须抽象为独立的逻辑方法，方法命名格式为 `list{ObjectName}` 或 `list{ObjectName}By{Filter}` ...
- **返回类型**：方法返回类型为 `([]interface{}, error)`，便于统一处理
- **错误处理**：请求错误和响应解析错误直接返回，不进行格式化，以保留原始错误信息
- **数据提取**：使用 `utils.PathSearch` 提取分页数据，默认值必须为 `make([]interface{}, 0)`（不能为 `nil`，否则会触发 panic）
- **类型断言**：分页列表必须断言为 `[]interface{}` 类型
- **最后一页判断**：必须判断当前页元素数，禁止判断 `result` 总数
- **性能优化**：根据 API 文档设置合适的单页最大返回对象数（通常为 100-5000），如果已知大概的数据量，可以预分配切片容量

对于不同类型的分页设计，分为以下几种不同的**要求**：

#### 8.1 使用 page 的分页

**设计原则**：

- 变量声明时在 `httpUrl` 中包含所有必要参数（必填参数和 `per_page` 参数），使用占位符 `{param_name}`
- 页码参数 `page` 初始值为 1，在循环中动态添加到 URL
- 判断最后一页：`len(currentPageItems) < perPage`（必须判断当前页元素数，禁止判断 `result` 总数）
- 先将当前页元素追加到 `result`，再判断是否为最后一页（以避免额外查询造成的浪费）
- 未到最后一页时执行 `page++`

**最佳实践**：

```go
func listProjects(client *golangsdk.ServiceClient, domainId string) ([]interface{}, error) {
	var (
		// 对应@API 注释的 GET /v3/projects 接口
		// 设计分页查询请求URI时需要在变量声明时加上所有必要的参数输入和分页的单页最大返回对象数的定义
		// 如当前GET /v3/projects 接口，包括了必填参数domain_id，对应填上domain_id={domain_id}
		// 单页最大返回对象数per_page参数，对应填上per_page={per_page}
		// 所有的参数通过与字符（&）进行连接并通过问号（？）与主URI连接在一起
		httpUrl = "v3/projects?domain_id={domain_id}&per_page={per_page}"
		perPage = 5000 // 根据API文档设置合适的单页最大返回对象数
		page    = 1    // 页码从1开始
		result  = make([]interface{}, 0)
	)

	// 构建基础URL：替换endpoint和路径参数
	httpUrl = client.Endpoint + httpUrl
	httpUrl = strings.ReplaceAll(httpUrl, "{domain_id}", domainId)
	httpUrl = strings.ReplaceAll(httpUrl, "{per_page}", strconv.Itoa(perPage))

	getProjectOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	// 循环获取所有分页数据
	for {
		// 页码参数page则在循环中进行添加
		httpUrlWithPage := fmt.Sprintf("%s&page=%d", httpUrl, page)
		requestResp, err := client.Request("GET", httpUrlWithPage, &getProjectOpt)
		if err != nil {
			return nil, err
		}
		respBody, err := utils.FlattenResponse(requestResp)
		if err != nil {
			return nil, err
		}
		// 分页列表必须断言为[]interface{}类型，并注意utils.PathSearch的默认值不能是nil（必须对应设置成make([]interface{}, 0)），否则会触发panic
		projects := utils.PathSearch("projects", respBody, make([]interface{}, 0)).([]interface{})
		// 先将当前页的元素存入result中，随后再判断当前页是否为最后一页
		result = append(result, projects...)
		// 如果当前页的元素数小于单页最大返回对象数则表示当前页已经是最后一页
		// 注意这里必须是判断当前分页返回的元素数是否小于单页最大返回对象数，禁止将result的总数与perPage进行比较
		if len(projects) < perPage {
			break
		}
		page++
	}

	return result, nil
}
```

#### 8.2 使用 marker 的分页

**设计原则**：

- 变量声明时在 `httpUrl` 中包含 `limit` 参数（使用占位符）
- `marker` 初始值为空字符串 `""`，表示从第一页开始
- 第一页请求时不添加 marker 参数（仅在 marker 不为空时添加）
- 从响应中获取下一个 marker（根据 API 响应结构确定路径，常见为 `page_info.next_marker` 或 `next_marker`）
- 每次循环都根据响应体中获取并更新的下一页的marker判断当前是否为最后一页：`marker == ""`（marker 为空表示已到最后一页）以避免额外查询造成的浪费

**最佳实践**：

```go
func listV5AgencyAssociatedPolicies(client *golangsdk.ServiceClient, agencyId string) ([]interface{}, error) {
	var (
		// 对应@API 注释的 GET /v5/agencies/{agency_id}/attached-policies 接口
		// marker分页使用limit参数指定单页最大返回对象数
		httpUrl = "v5/agencies/{agency_id}/attached-policies?limit={limit}"
		limit   = 100 // 根据API文档设置合适的单页最大返回对象数
		marker  = ""  // marker初始值为空字符串，表示从第一页开始
		result  = make([]interface{}, 0)
	)

	// 构建基础URL：替换endpoint、路径参数和limit参数
	listPath := client.Endpoint + httpUrl
	listPath = strings.ReplaceAll(listPath, "{agency_id}", agencyId)
	listPath = strings.ReplaceAll(listPath, "{limit}", strconv.Itoa(limit))

	opt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	// 循环获取所有分页数据
	for {
		// marker分页较为特殊，其首页的输入中不能携带marker，因此构建带marker的URL（仅在marker不为空时添加）时需要额外的判断逻辑
		// 第一页请求时marker为空，不添加marker参数
		// 注意：如果需要在循环中修改listPath，应该使用临时变量currentPath，避免影响后续循环
		currentPath := listPath
		if marker != "" {
			currentPath = fmt.Sprintf("%s&marker=%s", listPath, marker)
		}
		resp, err := client.Request("GET", currentPath, &opt)
		if err != nil {
			return nil, err
		}
		respBody, err := utils.FlattenResponse(resp)
		if err != nil {
			return nil, err
		}
		// 分页列表必须断言为[]interface{}类型，默认值必须为make([]interface{}, 0)
		policies := utils.PathSearch("attached_policies", respBody, make([]interface{}, 0)).([]interface{})
		result = append(result, policies...)
		// 从响应中获取下一个marker（根据API响应结构确定路径，常见为page_info.next_marker或next_marker）
		marker = utils.PathSearch("page_info.next_marker", respBody, "").(string)
		// marker为空表示已到最后一页，退出循环
		if marker == "" {
			break
		}
	}

	return result, nil
}
```

#### 8.3 使用 offset 的分页

**设计原则**：

- 变量声明时在 `httpUrl` 中包含 `limit` 参数（使用占位符）
- `offset` 初始值为 0，表示从第一条记录开始
- offset 参数在循环中动态添加到 URL
- 判断最后一页：`len(currentPageItems) < limit`（必须判断当前页元素数）
- 先将当前页元素追加到 `result`，再判断是否为最后一页（以避免额外查询造成的浪费）
- 未到最后一页时执行 `offset += len(currentPageItems)`（累加当前页元素数，而非固定增量如 `limit`）

**最佳实践**：

```go
func listEnterpriseProjects(client *golangsdk.ServiceClient) ([]interface{}, error) {
	var (
		// 对应@API 注释的 GET /v1.0/enterprise-projects 接口
		// offset分页使用limit参数指定单页最大返回对象数
		httpUrl = "v1.0/enterprise-projects?limit={limit}"
		limit   = 1000 // 根据API文档设置合适的单页最大返回对象数
		offset  = 0    // offset初始值为0，表示从第一条记录开始
		result  = make([]interface{}, 0)
	)

	// 构建基础URL：替换endpoint和limit参数
	httpUrl = client.Endpoint + httpUrl
	httpUrl = strings.ReplaceAll(httpUrl, "{limit}", strconv.Itoa(limit))

	getEnterpriseProjectOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
	}

	// 循环获取所有分页数据
	for {
		// offset参数在循环中动态添加
		httpUrlWithOffset := fmt.Sprintf("%s&offset=%d", httpUrl, offset)
		requestResp, err := client.Request("GET", httpUrlWithOffset, &getEnterpriseProjectOpt)
		if err != nil {
			return nil, err
		}
		respBody, err := utils.FlattenResponse(requestResp)
		if err != nil {
			return nil, err
		}
		// 分页列表必须断言为[]interface{}类型，默认值必须为make([]interface{}, 0)
		enterpriseProjects := utils.PathSearch("enterprise_projects", respBody, make([]interface{}, 0)).([]interface{})
		// 先将当前页的元素存入result中，随后再判断当前页是否为最后一页
		result = append(result, enterpriseProjects...)
		// 如果当前页的元素数小于单页最大返回对象数则表示当前页已经是最后一页
		// 注意这里必须是判断当前分页返回的元素数是否小于limit，禁止将result的总数与limit进行比较
		if len(enterpriseProjects) < limit {
			break
		}
		// offset累加当前页的元素数，而非固定增量（如limit）
		offset += len(enterpriseProjects)
	}

	return result, nil
}
```

**检查清单**：

- [ ] 分页查询方法是否抽象为独立的逻辑方法？
- [ ] 方法命名是否符合规范（`list{ObjectName}`或`list{ObjectName}By{Filter}`）？
- [ ] 方法返回类型是否为`([]interface{}, error)`？
- [ ] 错误处理是否正确（直接返回，不格式化）？
- [ ] 分页列表的默认值是否为`make([]interface{}, 0)`（不能为nil）？
- [ ] page分页是否正确判断最后一页（判断当前页元素数）？
- [ ] marker分页是否正确处理了首页（不添加marker参数）？
- [ ] offset分页是否正确累加（累加当前页元素数）？
- [ ] 是否根据API文档设置了合适的单页最大返回对象数？
- [ ] 是否预分配了切片容量以提高性能？

### 9. URL 构建和路径参数替换

**设计原则**：

- **占位符格式**：使用 `{variable_name}` 格式定义路径参数占位符（下划线格式）
- **占位符一致性**：路径占位符格式应与API注释中的路径参数格式保持一致，都使用下划线格式 `{variable_name}`
- **变量名格式**：用于替换占位符的Go语言变量名应遵循Go语言命名约定（驼峰格式，如 `engineId`, `jobId`），这与路径占位符格式是不同的概念
- **参数替换**：使用 `strings.ReplaceAll` 替换路径参数
- **动态参数**：使用 `fmt.Sprintf` 添加动态查询参数（如分页）
- **逻辑封装**：将 URL 构建逻辑封装在独立函数中
- **命名规范**：使用有意义的变量名（如 `attachPath`, `listPath`, `updatePath`）

**最佳实践**：

1. **路径参数替换**

```go
func attachProjectRoleToV3Agency(client *golangsdk.ServiceClient, agencyId, projectId, roleId string) error {
	// 路径占位符使用下划线格式 {variable_name}，与API注释保持一致
	httpUrl := "v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}"

	attachPath := client.Endpoint + httpUrl
	// 使用Go语言变量名（驼峰格式）所对应的值替换占位符（下划线格式）
	attachPath = strings.ReplaceAll(attachPath, "{project_id}", projectId) // projectId对应URL中的project_id，其命名符合Go语言规范
	attachPath = strings.ReplaceAll(attachPath, "{agency_id}", agencyId)
	attachPath = strings.ReplaceAll(attachPath, "{role_id}", roleId)

	attachOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
	}

	_, err := client.Request("PUT", attachPath, &attachOpt)
	if err != nil {
		return fmt.Errorf("error attaching role (%s) to agency (%s) by project (%s): %s",
			roleId, projectId, agencyId, err)
	}

	return nil
}
```

2. **查询参数构建**

```go
var (
	httpUrl = "v3/projects?name={name}&domain_id={domain_id}&per_page={per_page}"
	perPage = 100
)

httpUrl = client.Endpoint + httpUrl
httpUrl = strings.ReplaceAll(httpUrl, "{name}", name)
httpUrl = strings.ReplaceAll(httpUrl, "{domain_id}", domainId)
httpUrl = strings.ReplaceAll(httpUrl, "{per_page}", strconv.Itoa(perPage))

// 在循环中添加分页参数
for {
	httpUrlWithPage := fmt.Sprintf("%s&page=%d", httpUrl, page)
	// ...
}
```

**检查清单**：

- [ ] 路径参数占位符是否使用`{variable_name}`格式？
- [ ] 是否使用`strings.ReplaceAll`替换路径参数？
- [ ] 动态查询参数是否使用`fmt.Sprintf`添加？
- [ ] URL构建逻辑是否封装在独立函数中？
- [ ] 变量名是否有意义且清晰（如`attachPath`, `listPath`）？

### 10. 日志记录

**设计原则**：

- 资源代码中使用的日志记录**必须**包含`[LEVEL]`前缀，保持格式统一，其中`[LEVEL]`符合以下三种格式：
  + DEBUG: 记录操作流程和关键数据，便于调试
  + WARN: 记录非致命性问题（如资源不存在但可以继续、数据格式无效等）
  + ERROR: 记录错误但不中断流程（通常在 Read 函数中，某些查询失败不影响主流程）
- 包含足够的上下文信息（资源ID、操作类型、具体错误等）
- 在循环中记录警告时，使用`continue`跳过当前项而不是中断整个流程
- 错误消息应该清晰描述问题，便于排查

**最佳实践**：

```go
// DEBUG 级别 - 调试信息
log.Printf("[DEBUG] detaching roles %v in project scope from agency %s, the roles are %v", projectRoles, agencyId, projectRoles)
log.Printf("[DEBUG] attaching roles %v in project scope to agency %s", projectRoles, agencyId)

// WARN 级别 - 警告信息
log.Printf("[WARN] invalid project name (%s) or role names (%v)", projectName, roleNames.List())
log.Printf("[WARN] invalid project role (%v): invalid format", projectRole)
log.Printf("[WARN] the role (%s) to be detached does not exist", roleName)
log.Printf("[WARN] the policy (%s) was already detached from the agency (%s)", policyId, agencyId)

// ERROR级别 - 记录错误但不中断流程（通常在Read函数中）
log.Printf("[ERROR] error querying the roles attached on project(%s): %s", projectName, err)
log.Printf("[ERROR] error querying the roles attached on project for agency (%s): %s", agencyId, err)
log.Printf("[ERROR] error querying the roles attached on domain for agency (%s): %s", agencyId, err)
log.Printf("[ERROR] error querying the roles attached on project(%s): %s", projectName, err)
```

**检查清单**：

- [ ] 日志级别是否使用正确（DEBUG/WARN/ERROR）？
- [ ] 是否使用`[LEVEL]`前缀，格式统一？
- [ ] 日志消息是否包含足够的上下文信息（资源ID、操作类型等）？
- [ ] 在循环中记录警告时是否使用`continue`而不是中断流程？
- [ ] 错误消息是否清晰描述问题，便于排查？

### 11. 变量声明和变量组织

**设计原则**：

- 使用有意义的变量名
- 避免过长的变量名
- 相关变量可以分组声明
- 使用 `var` 块声明多个相关变量
- 变量的命名不能与导入的包名重复
- 变量的命名不能与Go语言**关键字**名称重复，其包含以下内容：
  - **包管理关键字**：`import`, `package` 
  - **声明与定义**：`chan`, `const`, `func`, `interface`, `map`, `struct`, `type`, `var` 
  - **流程控制**：`break`, `case`, `continue`, `default`, `defer`, `else`, `fallthrough`, `for`, `go`, `goto`, `if`, `range`, `return`, `select`, `switch` 

**最佳实践**：

1. **分组声明**

```go
var (
    cfg      = meta.(*config.Config)
    region   = cfg.GetRegion(d)
    agencyId = d.Id()
)
```

2. **短变量声明**

```go
httpUrl := "v3.0/OS-AGENCY/agencies/{agency_id}"
```

**检查清单**：

- [ ] 相关变量是否使用`var`块分组声明？
- [ ] 变量名是否有意义且清晰？
- [ ] 是否避免了过长的变量名？
- [ ] 变量声明是否符合Go语言规范？

### 12. 使用 utils.RemoveNil 清理请求体

**设计原则**：

- **清理nil值**：使用 `utils.RemoveNil` 清理请求体中的 nil 值，避免发送不必要的空字段到 API
- **保持简洁**：保持请求体的简洁性，只发送有意义的字段

**最佳实践**：

主要适用于资源的创建/更新操作，数据源如有需要也可参考：

**1. 在构建请求体方法的结果返回前调用**

```go
func buildCreateV3AgencyBodyParams(d *schema.ResourceData, domainId string) map[string]interface{} {
	return utils.RemoveNil(map[string]interface{}{
		"agency": map[string]interface{}{
			"domain_id":         domainId,
			"name":              d.Get("name").(string),
			"trust_domain_name": buildCreateV3AgencyDelegatedDomain(d),
			"description":       d.Get("description").(string),  // 可能为空字符串
			"duration":          buildCreateV3AgencyDuration(d), // 可能为 nil
		},
	}）
}

// 在 API 调用时使用 utils.RemoveNil 移除 nil 值
createAgencyOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	JSONBody:         utils.RemoveNil(buildCreateV3AgencyBodyParams(d, domainId)),
}
```

**2. 在构建RequestOpts的JSONBody时调用**

```go
func buildCreateV3AgencyBodyParams(d *schema.ResourceData, domainId string) map[string]interface{} {
	return map[string]interface{}{
		"agency": map[string]interface{}{
			"domain_id":         domainId,
			"name":              d.Get("name").(string),
			"trust_domain_name": buildCreateV3AgencyDelegatedDomain(d),
			"description":       d.Get("description").(string),  // 可能为空字符串
			"duration":          buildCreateV3AgencyDuration(d), // 可能为 nil
		},
	}）
}

// 在 API 调用时使用 utils.RemoveNil 移除 nil 值
createAgencyOpt := golangsdk.RequestOpts{
	KeepResponseBody: true,
	JSONBody:         utils.RemoveNil(buildCreateV3AgencyBodyParams(d, domainId)),
}
```

**检查清单**：

- [ ] 是否在API调用时使用`utils.RemoveNil()`清理请求体？
- [ ] 是否避免了发送不必要的空字段到API？
- [ ] 请求体是否保持简洁？

### 13. 正确使用Timeout

Timeout用于控制资源操作的超时时间，确保长时间运行的操作不会无限期等待。正确使用Timeout可以提高代码的健壮性和用户体验。

**设计原则**：

- **统一获取**：使用`d.Timeout(schema.Timeout{Create|Update|Delete|Read})`获取超时时间，而不是硬编码
- **参数传递**：将Timeout作为参数传递给重试方法和状态轮询方法
- **正确使用**：在`resource.RetryContext`和`resource.StateChangeConf`中正确使用Timeout
- **合理设置**：根据操作类型和资源复杂度合理设置Timeout值
- **状态稳定**：对于需要等待资源状态稳定的操作，Timeout应该足够长以覆盖状态转换时间
- **注释说明**：在重试方法中添加注释说明重试的原因和超时时间的用途
- **条件重试**：查询操作的重试应该只在资源是新创建的情况下进行：调用方仅在新资源时传入非零 timeout，重试方法通过 timeout 是否传入且大于 0 决定是否启用重试
- **多阶段支持**：如果一个方法需要使用到timeout信息，且该方法适用于多个调用阶段（Create、Update、Delete），则应将timeout定义于方法的入参中，根据不同的调用时机传入不同的超时时间

**最佳实践**：

主要适用于资源，数据源如有需要也可参考：

1. **在Resource函数中定义Timeout**

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceV3AgencyCreate,
		ReadContext:   resourceV3AgencyRead,
		UpdateContext: resourceV3AgencyUpdate,
		DeleteContext: resourceV3AgencyDelete,

		Timeouts: &schema.ResourceTimeout{
			Read: schema.DefaultTimeout(2 * time.Minute),
			Update: schema.DefaultTimeout(1 * time.Minute),
			Delete: schema.DefaultTimeout(1 * time.Minute),
		},

		Schema: map[string]*schema.Schema{
			// ...
		},
	}
}
```

2. **在CRUD方法中获取Timeout**

```go
// Read方法中
func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var timeout time.Duration
	if d.IsNewResource() {
		timeout = d.Timeout(schema.TimeoutRead)
	}
	agency, err := GetV3AgencyByIdWithRetry(ctx, client, agencyId, timeout)
	if err != nil {
		return common.CheckDeletedDiag(d, err, "error retrieving agency")
	}
	// ...
}

// Update方法中
func resourceV3AgencyUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	if d.HasChanges("delegated_domain_name", "delegated_service_name", "description", "duration") {
		if err = updateV3Agency(ctx, iamClient, agencyId, d, d.Timeout(schema.TimeoutUpdate)); err != nil {
			return diag.Errorf("error updating agency (%s): %s", agencyId, err)
		}
	}
	// ...
}

// Delete方法中
func resourceV3AgencyDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg      = meta.(*config.Config)
		region   = cfg.GetRegion(d)
		agencyId = d.Id()
		timeout  = d.Timeout(schema.TimeoutDelete)
	)
	// ...
}
```

3. **在重试方法中使用Timeout**

```go
// 查询方法中的重试（timeout 为可变参数：未传或为 0 时只查询一次，传入且 >0 时在 404 时重试）
func GetV3AgencyByIdWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout ...time.Duration) (interface{}, error) {
	var (
		respBody   interface{}
		err        error
		timeoutVal time.Duration
	)

	if len(timeout) < 1 || timeout[0] <= time.Duration(0) {
		return getV3AgencyById(client, agencyId)
	}
	timeoutVal = timeout[0]

	// lintignore:R006
	err = resource.RetryContext(ctx, timeoutVal, func() *resource.RetryError {
		respBody, err = getV3AgencyById(client, agencyId)
		if _, ok := err.(golangsdk.ErrDefault404); ok {
			// Retrieving agency details may result in a 404 error, requiring appropriate retries.
			// If the details are not retrieved within the timeout period, an error will be returned.
			// lintignore:R018
			time.Sleep(10 * time.Second)
			return resource.RetryableError(err)
		}
		if err != nil {
			return resource.NonRetryableError(err)
		}
		return nil
	})

	return respBody, err
}

// 更新方法中的重试
func updateV3Agency(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, d *schema.ResourceData,
	timeout time.Duration) error {
	// ...
	// lintignore:R006
	err := resource.RetryContext(ctx, timeout, func() *resource.RetryError {
		_, retryErr := client.Request("PUT", updatePath, &updateOpt)
		if retryErr != nil {
			return common.CheckForRetryableError(retryErr)
		}
		// Wait for the update to take effect
		// lintignore:R018
		time.Sleep(10 * time.Second)
		return nil
	})
	// ...
}

// 删除方法中的重试
func deleteV3AgencyWithRetry(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, timeout time.Duration) error {
	// lintignore:R006
	return resource.RetryContext(ctx, timeout, func() *resource.RetryError {
		err := deleteV3Agency(client, agencyId)
		if err != nil {
			if _, ok := err.(golangsdk.ErrDefault404); ok {
				return resource.NonRetryableError(err)
			}
			// Retrieving agency details may result in a 404 error, requiring appropriate retries.
			// If the details are not retrieved within the timeout period, an error will be returned.
			// lintignore:R018
			time.Sleep(10 * time.Second)
			return resource.RetryableError(err)
		}
		return nil
	})
}
```

4. **在状态轮询中使用Timeout**

```go
func waitForTaskStartedORCompleted(ctx context.Context, client *golangsdk.ServiceClient, taskID string, timeout time.Duration) error {
	stateConf := &resource.StateChangeConf{
		Pending:      []string{"PENDING"},
		Target:       []string{"COMPLETED"},
		Refresh:      refreshTaskStatus(client, taskId, []string{"success"}),
		Timeout:      timeout,
		Delay:        5 * time.Second,
		MinTimeout:   3 * time.Second,
		PollInterval: 10 * time.Second,
	}

	_, err := stateConf.WaitForStateContext(ctx)
	return err
}

// 在Create方法中使用
func resourceMigrationTaskCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	if err = waitForTaskStartedORCompleted(ctx, client, taskID, d.Timeout(schema.TimeoutCreate)); err != nil {
		return diag.Errorf("error waiting for the migration task (%s) to be started: %s", taskID, err)
	}
	// ...
}
```

**检查清单**：

- [ ] 是否使用`d.Timeout()`获取超时时间而不是硬编码？
- [ ] 是否将Timeout作为参数传递给重试方法和状态轮询方法？
- [ ] 是否在`resource.RetryContext`和`resource.StateChangeConf`中正确使用Timeout？
- [ ] Timeout值是否根据操作类型和资源复杂度合理设置？
- [ ] 查询操作的重试是否只在资源是新创建时进行？
- [ ] 删除操作中的404错误是否返回`NonRetryableError`？
- [ ] 重试方法中是否添加了注释说明重试原因和超时时间用途？

### 14. 标签管理

资源中的 `tags` 参数用于为资源绑定键值对标签，需遵循统一的 Schema 定义、默认标签合并及 CRUD 处理规范。

**设计原则**：

- **Schema 定义**：使用 `common.TagsSchema(description)` 定义 tags 参数，类型为 `TypeMap`，`Optional: true`，`Computed: true`，`Elem: &schema.Schema{Type: schema.TypeString}`
- **默认标签合并**：必须同时支持 Provider 级默认标签，在 `CustomizeDiff` 中声明 `config.MergeDefaultTags()`，与 `config.FlexibleForceNew` 等组合时使用 `customdiff.All()` 包裹
- **创建请求体**：使用 `utils.ExpandResourceTagsMap(d.Get("tags").(map[string]interface{}))` 将 Terraform 的 map 转为 API 所需的列表格式（`[{"key":"k","value":"v"}]`）
- **读取回填**：使用 `utils.FlattenTagsToMap(utils.PathSearch("tags", respBody, make([]interface{}, 0)).([]interface{}))` 将 API 返回的标签列表转为 map 并回填
- **更新逻辑**：根据 API 设计分两种方式：
  - 标签的创建、更新能力包含在资源的创建、更新接口中时，在对应请求体中构造tags
  - 标签的管理享有独立接口（如 `PUT /xxx/tags`）时，通过 `d.HasChange("tags")` 判断并单独调用标签更新接口（可能是`创建标签+删除标签`的功能组合实现的标签更新，抽象而成的公共方法适用于CUD方法）
- **API 注释**：标签使用独立接口时，需在资源上方添加对应的 `@API` 注释（如 `PUT /v1/{project_id}/instances/{instance_id}/tags`）

**最佳实践**

**1. 部分标签操作通过单独的接口完成，另一部分则直接通过资源本身的操作接口完成**

创建时 tags 随主接口传入，读取时从响应中解析，更新时通过独立接口 `PUT /v1/{project_id}/instances/{instance_id}/tags` 全量覆盖。

```go
// @API LakeFormation POST /v1/{project_id}/instances
// @API LakeFormation GET /v1/{project_id}/instances/{instance_id}
// @API LakeFormation PUT /v1/{project_id}/instances/{instance_id}
// @API LakeFormation PUT /v1/{project_id}/instances/{instance_id}/tags
// @API LakeFormation DELETE /v1/{project_id}/instances/{instance_id}
func ResourceInstance() *schema.Resource {
	return &schema.Resource{
		// ...

		CustomizeDiff: customdiff.All(
			config.FlexibleForceNew(instanceNonUpdatableParams),
			config.MergeDefaultTags(),
		),

		Schema: map[string]*schema.Schema{
			// ...
			"tags": common.TagsSchema(`The key/value pairs to associate with the instance.`),
			// ...
		},
	}
}

// 创建：tags 随资源的创建请求传入
func buildCreateInstanceBodyParams(cfg *config.Config, d *schema.ResourceData) map[string]interface{} {
	return map[string]interface{}{
		"name":  d.Get("name"),
		"tags":  utils.ExpandResourceTagsMap(d.Get("tags").(map[string]interface{})),
		// ...
	}
}

// 读取：直接从资源的详情返回的结构体中解析标签内容并回填
func resourceInstanceRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...

	respBody, err := GetInstanceById(client, d.Id())
	if err != nil {
		return common.CheckDeletedDiag(d, err, "error retrieving LakeFormation instance")
	}

	mErr := multierror.Append(nil,
		// ...
		d.Set("tags", utils.FlattenTagsToMap(utils.PathSearch("tags", respBody,
			make([]interface{}, 0)).([]interface{}))),
	)

	return diag.FromErr(mErr.ErrorOrNil())
}

func updateInstanceTags(client *golangsdk.ServiceClient, d *schema.ResourceData) error {
	httpUrl := "v1/{project_id}/instances/{instance_id}/tags"
	// ...
	JSONBody: map[string]interface{}{
		"tags": utils.ExpandResourceTagsMap(d.Get("tags").(map[string]interface{})),
	},
}

// 更新：标签有独立接口时，通过 d.HasChange 判断并单独调用
func resourceInstanceUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...

	if d.HasChange("tags") {
		err = updateInstanceTags(client, d)
		if err != nil {
			return diag.FromErr(err)
		}
	}

	return resourceInstanceRead(ctx, d, meta)
}
```

**2. 更新标签通过独立的接口实现（包含创建标签和删除标签接口）**

```go

```

**检查清单**：

- [ ] tags 是否使用 `common.TagsSchema` 定义？
- [ ] 支持默认标签时是否在 CustomizeDiff 中声明 `config.MergeDefaultTags()`？
- [ ] 创建时是否使用 `utils.ExpandResourceTagsMap` 构建请求体？
- [ ] 读取时是否使用 `utils.FlattenTagsToMap` 解析并回填？
- [ ] 更新时是否根据 API 设计正确处理（主接口包含 tags 或独立标签接口）？
- [ ] 标签使用独立接口时是否添加对应 `@API` 注释？

### 15. API 注释规范

**必须**在资源函数或数据源函数上方添加 `@API` 注释，标注所有使用的 API 端点。API注释应该在所有逻辑方法都确定好设计和各自的顺序后再进行补充，以确保注释的顺序与API在代码中的实际使用顺序一致。

**设计原则**：

- **添加时机**：API注释应该在所有逻辑方法设计完成后再添加
- **排序规则**：
  + 资源：按照CRUD操作顺序（Create → Read → Update → Delete）排列API注释
  + 数据源：通常只有一个Read操作，API注释按使用顺序排列
- **唯一性**：每个API只注释一次，即使在不同方法中被多次调用
- **一致性**：确保注释的顺序与代码中API的实际调用顺序一致
- **准确性**：使用准确的产品名称缩写和HTTP方法
- **格式要求**：
  + 格式：`// @API Product httpMethod requestPath`
  + 每个 API 定义一行
  + Product使用实际的产品名称缩写（如 IAM、VPC、ECS 等）
  + HTTP 方法必须大写（GET、POST、PUT、DELETE、PATCH 等）
  + 路径中的变量使用 `{variable_name}` 格式（下划线格式）
  + **重要**：API注释中的路径占位符格式应与代码中httpUrl的占位符格式保持一致，都使用下划线格式 `{variable_name}`。这与Go语言变量名格式（驼峰格式）是不同的概念
  + API注释根据API在当前资源/数据源中的使用顺序依次排列

**最佳实践**：

**资源示例**：

```go
// @API IAM GET /v3/projects
// @API EPS GET /v1.0/enterprise-projects
// @API IAM GET /v3/roles
// @API IAM POST /v3.0/OS-AGENCY/agencies
// @API IAM PUT /v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}
// @API IAM PUT /v3.0/OS-AGENCY/domains/{domain_id}/agencies/{agency_id}/roles/{role_id}
// @API IAM PUT /v3.0/OS-INHERIT/domains/{domain_id}/agencies/{agency_id}/roles/{role_id}/inherited_to_projects
// @API IAM PUT /v3.0/OS-PERMISSION/subjects/agency/scopes/enterprise-project/role-assignments
// @API IAM GET /v3.0/OS-AGENCY/agencies/{agency_id}
// @API IAM GET /v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles
// @API IAM GET /v3.0/OS-AGENCY/domains/{domain_id}/agencies/{agency_id}/roles
// @API IAM PUT /v3.0/OS-AGENCY/agencies/{agency_id}
// @API IAM DELETE /v3.0/OS-AGENCY/projects/{project_id}/agencies/{agency_id}/roles/{role_id}
// @API IAM DELETE /v3.0/OS-AGENCY/domains/{domain_id}/agencies/{agency_id}/roles/{role_id}
// @API IAM DELETE /v3.0/OS-INHERIT/domains/{domain_id}/agencies/{agency_id}/roles/{role_id}/inherited_to_projects
// @API IAM DELETE /v3.0/OS-PERMISSION/subjects/agency/scopes/enterprise-project/role-assignments
// @API IAM GET /v5/agencies/{agency_id}/attached-policies
// @API IAM POST /v5/policies/{policy_id}/detach-agency
// @API IAM DELETE /v3.0/OS-AGENCY/agencies/{agency_id}
func ResourceV3Agency() *schema.Resource {
    // ...
}
```

**数据源示例**：

```go
// @API IAM GET /v3/roles
func DataSourceV3Roles() *schema.Resource {
	// ...
}
```

**检查清单**：

- [ ] 是否在资源函数或数据源函数上方添加了`@API`注释？
- [ ] API注释是否按照正确的顺序排列（资源：Create → Read → Update → Delete；数据源：按使用顺序）？
- [ ] 每个API是否只注释一次？
- [ ] 注释的顺序是否与代码中API的实际调用顺序一致？
- [ ] 产品名称缩写是否准确（如IAM、VPC、ECS等）？
- [ ] HTTP方法是否大写（GET、POST、PUT、DELETE等）？
- [ ] 路径中的变量是否使用`{variable_name}`格式？

### 16. 代码注释

**设计原则**：

- **函数注释**：**要求**具备复杂功能的函数，其注释应清晰描述函数的功能，如果参数和返回值也具备特殊意义，也需要一并说明；而对于职责单一的函数，如果其名称已说明用途且无特殊处理，则无需注释
- **内联注释**：内联注释应解释复杂的逻辑或特殊处理
- **Linter忽略注释**：Linter忽略注释应说明忽略的原因
- **简洁性**：注释应保持简洁，避免冗余
- **包外可见方法**：包外可见的方法（Schema主函数例外）**必须**添加文档注释，文档注释使用完整的句子，以方法名开头
- **包内可见**的方法如无特殊的逻辑处理，无需注释（特别是CRUD主方法）

**最佳实践**：

1. **函数注释**

```go
// 不同于普通函数，该build函数没有将对象列表进行返回，而是提取了map的所有key，组成了新的列表进行返回，因此需要通过注释特殊说明keys(@)达成的效果
// buildProjectRoles builds the project roles from the project roles set, the format of each element is "project_name|role_name"
func buildProjectRoles(projectRoles *schema.Set) []interface{} {
	return utils.PathSearch("keys(@)", parseProjectRolesToPairs(projectRoles), make([]interface{}, 0)).([]interface{})
}

// 该函数将数组中每个对象的id和display_name按照固定格式拼接成新的元素组成新数组，该特殊逻辑需要通过注释特殊说明
// parseRolesToPairs parses the roles to pairs, the key is the role name, the value is the role ID.
// This method used to convert the list of characters into a mapping from character names to character IDs, which
// facilitates fast indexing later. The tree structure of the map is superior to slice in terms of performance.
func parseRolesToPairs(roles []interface{}) map[string]string {
	result := make(map[string]string)

	for _, role := range roles {
		roleName := utils.PathSearch("display_name", role, "").(string)
		roleId := utils.PathSearch("id", role, "").(string)
		if roleName == "" || roleId == "" {
			log.Printf("[WARN] invalid role name (%s) or ID (%s)", roleName, roleId)
			continue
		}
		result[roleName] = roleId
	}

	return result
}
```

2. **内联注释**

```go
// The key is the combination of project name and role name, which formatted as "project_name|role_name".
// Map structure can ensure de-duplication, so there is no need to worry about duplication.
result := make(map[string]bool)

// MOS is a special project for CBC service which is used for billing, not visible to the user
if projectId == "MOS" {
    continue
}

// the provider will query the roles in all projects, but the API rate limit threshold is 10 times per second.
// so we should wait for some time to avoid exceeding the rate limit.
// lintignore:R018
time.Sleep(200 * time.Millisecond)

// Retrieving agency details may result in a 404 error, requiring appropriate retries.
// If the details are not retrieved within the timeout period, an error will be returned.
// lintignore:R018
time.Sleep(10 * time.Second)
```

3. **Linter 忽略注释**

```go
// lintignore:R006
err = resource.RetryContext(ctx, timeout, func() *resource.RetryError {
    // ...
})

// lintignore:R018
time.Sleep(10 * time.Second)
```

**检查清单**：

- [ ] 包外可见的函数（Schema主函数例外）是否添加了清晰的文档注释？
- [ ] 复杂逻辑是否添加了内联注释说明？
- [ ] Linter忽略注释是否说明了忽略的原因？
- [ ] 注释是否保持简洁，避免冗余？

### 17. 并发控制

部分资源同一时刻只能创建、变更或删除一个对象（串行管理），在批量操作场景下会产生并发冲突。需在资源 CRUD 方法中加并发锁，避免多个 Terraform 操作同时修改同一串行范围导致 API 报错或数据不一致。

**设计原则**：

- **适用场景**：当 API 文档或实际验证表明，同一资源/域/实例等串行范围内不支持并发创建、更新或删除时，应加锁
- **锁机制**：使用 `config.MutexKV.Lock(key)` 与 `config.MutexKV.Unlock(key)`，通过 `key` 唯一标识串行范围
- **defer 释放**：必须使用 `defer config.MutexKV.Unlock(key)`，确保无论正常返回、提前 return 或 panic 都能释放锁，避免死锁
- **锁的粒度**：`key` 通常为串行范围的唯一标识，如 `domainId`、`clusterId`、`instanceId`、`resourceId`、`certificateId` 等
- **加锁位置**：在 CRUD 方法开头、在获取 client 之前或之后立即加锁，确保整个操作期间持有锁
- **注释说明**：必须在加锁前添加注释，说明为何需要加锁（并发问题、相关 API 错误码等）
- **多锁顺序**：需要锁定多个资源时，锁的获取顺序应全局一致（如按字母序或固定顺序），避免死锁

**最佳实践**：

1. **单锁场景**（以 IAM ACL 为例，域级串行）

```go
func resourceV3AclUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg      = meta.(*config.Config)
		domainId = cfg.DomainID
	)

	// ACL policy change operations may encounter concurrency issues (causing other ACL policy changes to fail),
	// so, it is necessary to lock the domain ID to prevent concurrent changes.
	config.MutexKV.Lock(domainId)
	defer config.MutexKV.Unlock(domainId)

	client, err := cfg.IAMV3Client(cfg.GetRegion(d))
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	if err := updateV3AclPolicy(client, d, domainId); err != nil {
		return diag.Errorf("error updating identity ACL: %s", err)
	}

	// ...
	return resourceV3AclRead(ctx, d, meta)
}
```

2. **多锁场景**（以云硬盘挂载为例，需同时锁定实例和云盘）

```go
// The ECS instances do not support mounting multiple volumes at the same time.
instanceId := d.Get("instance_id").(string)
config.MutexKV.Lock(instanceId)
defer config.MutexKV.Unlock(instanceId)
// The EVS volumes also do not support being mounted to multiple instances at the same time.
volumeId := d.Get("volume_id").(string)
config.MutexKV.Lock(volumeId)
defer config.MutexKV.Unlock(volumeId)
```

**检查清单**：

- [ ] 是否在需要串行操作的 CRUD 方法中加锁？
- [ ] 是否使用 `defer config.MutexKV.Unlock(key)` 确保锁释放？
- [ ] 锁的 `key` 是否正确标识串行范围（如 domainId、clusterId、instanceId 等）？
- [ ] 加锁位置是否在方法开头、client 获取前后？
- [ ] 是否添加注释说明加锁原因（并发问题、API 错误码等）？
- [ ] 多锁场景下，锁的获取顺序是否全局一致，避免死锁？

### 18. 性能优化

**设计原则**：

- **预分配容量**：预分配切片容量以提高性能
- **批量操作**：批量操作减少API调用次数
- **避免冗余**：避免不必要的API调用
- **按需查询**：只在需要时查询数据

**最佳实践**：

1. **预分配切片容量**

```go
result := make([]string, 0, len(pairs))
```

2. **批量操作**

```go
// 预分配切片容量
roleAssignments := make([]interface{}, 0, len(enterpriseProjectRoles))

// 批量构建请求体
for _, enterpriseProjectRole := range enterpriseProjectRoles {
    // ... 处理逻辑
    assignments = append(assignments, map[string]interface{}{
        "agency_id":             agencyId,
        "enterprise_project_id": epsId,
        "role_id":               roleId,
    })
}

// 批量提交
attachOpt := golangsdk.RequestOpts{
    KeepResponseBody: true,
    JSONBody: map[string]interface{}{
        "role_assignments": assignments,
    },
}
```

3. **避免不必要的 API 调用**

```go
// 只在需要时查询角色列表
var parsedRolePairs map[string]string
if d.HasChanges("project_role", "domain_roles", "all_resources_roles", "enterprise_project_roles") {
    // get all of the role IDs, include system-defined roles and custom roles
    allRoles, err := listAllRoles(iamClient, domainId)
    if err != nil {
        return diag.FromErr(err)
    }
    parsedRolePairs = parseRolesToPairs(allRoles)
}

// 分别处理每个字段的变更
if d.HasChange("project_role") {
	if err = updateProjectRolesForV3Agency(iamClient, d, parsedRolePairs, domainId, agencyId); err != nil {
		return diag.FromErr(err)
	}
}
```

**检查清单**：

- [ ] 是否预分配了切片容量以提高性能？
- [ ] 是否使用批量操作减少API调用次数？
- [ ] 是否避免了不必要的API调用？
- [ ] 是否只在需要时查询数据？

### 19. 类型转换和断言

**说明**：本规范适用于资源和数据源，提供统一的类型转换和断言规范。
**约定**：
1. 通过`utils.PathSearch()`处理过的数据被认为是类型可靠，后续的断言类型如果与默认值类型匹配则允许忽略其风险检测（如果JMESPATH Key未找到会返回默认值，且无需检查具体的断言类型是否与参数命名匹配），如：
  - `utils.PathSearch("instance.status", respBody, "").(string)` （默认值空字符串的类型为string，与断言类型一致，可忽略断言风险）
  - `utils.PathSearch("instance.paths", respBody, make([]interface{}, 0)).([]interface{})` （默认值空数组的类型为`[]interface{}`，与断言类型一致，可忽略断言风险，注意列表一定要开辟空间）
  - `utils.PathSearch("instance.properties", respBody, make(map[string]interface{})).(map[string]interface{})` （默认值空字典的类型为`map[string]interface{}`，与断言类型一致，可忽略断言风险，注意字典一定要开辟空间）
2. 在API保证值在合理范围内的情况下，强制类型转换（如`float64`到`int`、`int64`到`int`等）带来的精度丢失或范围溢出风险可以被忽略。例如，当API文档明确说明某个字段的值在`int`范围内，或者实际业务场景中该值不可能超出目标类型范围时，可以直接进行强制类型转换。
3. CRUD方法中对`meta interface{}`的`*config.Config`断言，如（cfg = meta.(*config.Config)），忽略对其的断言风险检测

**设计原则**：

- 使用`utils.PathSearch()`方法代替传统Go语言类型值解析或`d.Get()`获取结构体子参数值及查询请求的响应值
- 为 `utils.PathSearch` 提供**类型匹配的**默认值，在对结果进行断言的场景下**禁止**使用nil作为默认值，避免 nil 指针断言抛出panic，如果不存在后续断言则允许使用nil作为其默认值
  - 注：明确允许在请求体构建函数中以及状态值存储的相关方法中，无论是d.Get方法还是d.Set方法，只要后续值没有进行断言或者被引用于非interface参数输入类型的方法中时，允许使用nil作为默认值
- **重要约定**：所有对`utils.PathSearch()`和`d.Get()`返回值的直接类型断言（如`.(string)`、`.(int)`、`.(float64)`等）都被认为是**安全的、非风险的**，无需进行`ok`值检查。这是因为`utils.PathSearch()`已经提供了类型匹配的默认值，`d.Get()`是依照schema进行设计的，确保返回值的类型与默认值类型一致。
- 类型断言前必须确保类型正确，默认值必须为对应类型的空值
- 使用 JSONPath 表达式或 JMES 方法进行复杂的数据提取
- 如果代码中**必须**使用断言进行类型转换（非`utils.PathSearch()`和`d.Get()`返回值），则应采用安全的断言模式（使用类型断言时检查 `ok` 值）
- 对于已知类型，可以直接转换（schema映射，对于 `schema.ResourceData` 的 `Get` 方法，通常可以直接转换）

**默认值规范**：

- 字符串类型默认值：`""`
- 整数类型默认值：`0`
- 布尔类型默认值：`false`
- 字典类型默认值：`make(map[string]interface{})`
- 列表类型默认值：`make([]interface{}, 0)`

**最佳实践**：

**注意**：对于`utils.PathSearch()`和`d.Get()`返回值的类型断言，无需进行`ok`值检查，直接断言即可（见第1、2示例）。

1. **使用 utils.PathSearch() 提取嵌套数据（查询响应和结构体子参数）**

```go
// 从查询响应中提取简单字段
agencyId := utils.PathSearch("agency.id", respBody, "").(string)

// 从查询响应中提取数组字段
projects := utils.PathSearch("projects", respBody, make([]interface{}, 0)).([]interface{})

// 使用 JSONPath 表达式从查询响应中进行复杂查询
projectId := utils.PathSearch(fmt.Sprintf("[?name=='%s']|[0].id", name), result, "").(string)

// 提取嵌套数组中的字段
roleNames := utils.PathSearch("[*].display_name", attachedProjectRoles, make([]interface{}, 0)).([]interface{})

// 提取分页信息
marker := utils.PathSearch("page_info.next_marker", respBody, "").(string)

// 使用 JMES 方法提取map对象的所有key，组成一个新的列表（注意：这要求被检对象的类型为map[string]interface{})
return utils.PathSearch("keys(@)", parseProjectRolesToPairs(projectRoles), make([]interface{}, 0)).([]interface{})
```

2. **使用 d.Get 获取的**

```go
// d.Get() 返回值由 schema 类型约束，允许直接断言
name := d.Get("name").(string)
networkId := d.Get("network_id").(string)
azList := d.Get("availability_zones").(*schema.Set).List()
tags := d.Get("tags").(map[string]interface{})
```

3. **安全的类型断言（针对非`utils.PathSearch()`和`d.Get()`返回值）**

对于非`utils.PathSearch()`和`d.Get()`返回值的类型断言，应该使用安全的断言模式：

```go
// 对于从其他来源获取的interface{}值，需要检查ok值
projectRoleMap, ok := projectRole.(map[string]interface{})
if !ok || len(projectRoleMap) < 1 {
    continue
}
```

4. **类型转换**

```go
// 字符串类型
name := utils.PathSearch("name", role, "").(string)

// 整数类型
count := utils.PathSearch("count", respBody, 0).(int)

// 布尔类型
enabled := utils.PathSearch("enabled", respBody, false).(bool)

// 字典类型
links := utils.PathSearch("links", role, make(map[string]interface{})).(map[string]interface{})

// 列表类型
roles := utils.PathSearch("roles", respBody, make([]interface{}, 0)).([]interface{})

// schema.ResourceData 的类型转换（已知类型，可直接转换）
name := d.Get("name").(string)
domainRoles := d.Get("domain_roles").(*schema.Set)
```

4. **强制类型转换（精度丢失风险可忽略的场景）**

当API保证值在合理范围内时，可以直接进行强制类型转换：

```go
// float64 到 int 的转换（API保证jobId值在int范围内）
jobIdStr := strconv.Itoa(int(utils.PathSearch("jobId", resp, float64(0)).(float64)))

// int64 到 int 的转换（API保证值在int范围内）
count := int(utils.PathSearch("count", respBody, int64(0)).(int64))

// float64 到 int32 的转换（API保证值在int32范围内）
port := int32(utils.PathSearch("port", respBody, float64(0)).(float64))
```

**注意**：只有在API文档明确说明值在目标类型范围内，或者实际业务场景中该值不可能超出目标类型范围时，才允许忽略强制类型转换的精度丢失风险。如果不确定，应该使用更安全的转换方式（如`strconv.FormatFloat`、范围检查等）。

**检查清单**：

- [ ] 对于`utils.PathSearch()`和`d.Get()`返回值的类型断言，是否直接使用（无需`ok`值检查）？
- [ ] 对于非`utils.PathSearch()`返回值的类型断言，是否检查了`ok`值？
- [ ] `utils.PathSearch`是否提供了正确的默认值（避免nil指针错误）？
- [ ] 默认值是否为对应类型的空值（字符串为`""`，整数为`0`，布尔为`false`，字典为`make(map[string]interface{})`，列表为`make([]interface{}, 0)`）？
- [ ] 是否使用JSONPath表达式进行复杂的数据提取？
- [ ] 类型转换是否正确处理了各种情况？
- [ ] 强制类型转换是否在API保证值在合理范围内的场景下使用？

**重要说明（用于消除检查歧义）**：
- **`utils.PathSearch()`返回值的类型断言**：当默认值与断言类型一致时，直接断言被视为安全（无需`ok`检查）。例如：
  - `utils.PathSearch("reference.serviceLimit", engineDetail, "").(string)`（默认值为`""`）
  - `utils.PathSearch("framework", microservice, make(map[string]interface{})).(map[string]interface{})`（默认值为`make(map[string]interface{})`）
- **`d.Get()`返回值的类型断言**：直接断言被视为安全（由 schema 类型约束保证），无需`ok`检查。  
  例如：`d.Get("name").(string)`、`d.Get("domain_roles").(*schema.Set)`。
- **其他来源的`interface{}`断言**：仅针对非`utils.PathSearch()`和非`d.Get()`来源，才要求使用安全断言模式（检查`ok`值）。
- **强制类型转换的精度丢失风险**：在API保证值在合理范围内的情况下，强制类型转换（如`float64`到`int`、`int64`到`int`等）带来的精度丢失或范围溢出风险可以被忽略。例如，当API返回的`jobId`值保证在`int`范围内时，`int(utils.PathSearch("jobId", resp, float64(0)).(float64))`这种转换是允许的，无需进行范围检查或使用更安全的转换方式。

### 20. 代码行长度规范

**说明**：本规范适用于资源、数据源和测试用例的代码，确保代码的可读性和可维护性。

**设计原则**：

- 单行代码最大长度不超过**150**个字符
- 如果单行代码超过**150**个字符，应在适当位置进行换行，保持代码美观和可读性
- 换行时应遵循Go语言的代码风格规范，保持合理的缩进
- 优先在操作符、逗号、括号等位置进行换行
- 换行后的代码应保持逻辑清晰，避免过度拆分导致理解困难

**最佳实践**：

1. **函数调用参数过长时的换行**

```go
// ❌ 错误：单行代码超过150个字符
parsedErr := common.ConvertExpected400ErrInto404Err(common.ConvertExpected401ErrInto404Err(err, "error_code", microserviceEngineNotFoundCodes...), "error_code", microserviceEngineNotFoundCodes...)

// ❌ 错误：单行代码超过150个字符
func createMicroserviceEngine(cseClient, vpcClient *golangsdk.ServiceClient, d *schema.ResourceData, enterpriseProjectId string) (interface{}, error) {

// ✅ 正确：在适当位置换行
parsedErr := common.ConvertExpected400ErrInto404Err(
	common.ConvertExpected401ErrInto404Err(err, "error_code", microserviceEngineNotFoundCodes...),
	"error_code",
	microserviceEngineNotFoundCodes...,
)

// ✅ 正确：多个相关参数可以放在同一行
func buildMicroserviceEngineCreateOpts(d *schema.ResourceData, enterpriseProjectId string,
	vpcInfo, subnetInfo interface{}) map[string]interface{} {

func createMicroserviceEngine(cseClient, vpcClient *golangsdk.ServiceClient, d *schema.ResourceData,
	enterpriseProjectId string) (interface{}, error) {
```

2. **复杂表达式过长时的换行**

```go
// ❌ 错误：单行代码超过150个字符
mErr = multierror.Append(mErr, d.Set("domain_roles", utils.PathSearch("[*].display_name", domainRoles, make([]interface{}, 0)).([]interface{})))

// ✅ 正确：在适当位置换行
mErr = multierror.Append(mErr, d.Set("domain_roles", utils.PathSearch("[*].display_name", domainRoles,
			make([]interface{}, 0)).([]interface{})))
```

3. **字符串拼接过长时的换行**

```go
// ❌ 错误：单行代码超过150个字符
errorMsg := fmt.Sprintf("error creating microservice engine (%s) in region (%s) with enterprise project (%s): %s", engineId, region, enterpriseProjectId, err)

// ✅ 正确：在适当位置换行
errorMsg := fmt.Sprintf("error creating microservice engine (%s) in region (%s) with enterprise project (%s): %s",
    engineId, region, enterpriseProjectId, err)
```

4. **结构体初始化过长时的换行**

```go
// ❌ 错误：将结构体初始化置于单行导致代码超过150个字符
stateConf := &resource.StateChangeConf{Pending: []string{"PENDING"}, Target: []string{"COMPLETED"}, Refresh: refreshFunc, Timeout: d.Timeout(schema.TimeoutCreate), Delay: 180 * time.Second, PollInterval: 15 * time.Second}

// ✅ 正确：在适当位置换行，保持代码整洁
stateConf := &resource.StateChangeConf{
    Pending:    []string{"PENDING"},
    Target:     []string{"COMPLETED"},
    Refresh:    refreshFunc,
    Timeout:    d.Timeout(schema.TimeoutCreate),
    Delay:      180 * time.Second,
    PollInterval: 15 * time.Second,
}
```

**检查清单**：

- [ ] 单行代码长度是否不超过150个字符？
- [ ] 超过150个字符的代码是否在适当位置进行了换行？
- [ ] 换行后的代码是否保持了良好的可读性和逻辑清晰性？
- [ ] 换行位置是否合理（操作符、逗号、括号等）？
- [ ] 换行后的缩进是否保持一致？

### 21. 函数参数使用规范

**说明**：本规范适用于资源、数据源和测试用例的函数定义，确保函数参数的精简和有效性。

**设计原则**：

- **自定义辅助函数**：如果函数的入参未被使用，则**不应该被定义在参数列表中**，应从参数列表中移除
- **必须符合接口签名的函数**（如CRUD方法、接口实现等）：对于未使用的参数，应使用下划线`_`进行标记，表示该参数被有意忽略
- 对于Context等标准参数，如果未被使用，应使用`_ context.Context`的形式
- 保持函数签名的简洁性，避免冗余参数
- **重要**：检查函数体内部是否使用了所有定义的参数，确保没有未使用的参数

**最佳实践**：

1. **未使用的Context参数**

```go
// ❌ 错误：Context参数未被使用但未标记
func resourceMicroserviceEngineRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ctx未被使用
    // ...
}

// ✅ 正确：使用下划线标记未使用的Context参数
func resourceMicroserviceEngineRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
}

// ❌ 错误：ctx、d和meta参数均未被使用但未标记
func resourceMicroserviceEngineUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ctx、d和meta参数均未被使用
    return nil
}

// ✅ 正确：使用下划线标记未使用的ResourceData参数
func resourceMicroserviceEngineUpdate(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
    return nil
}
```

2. **自定义辅助函数中未使用的参数**

```go
// ❌ 错误：参数列表中存在未被使用的入参
func buildAuthCred(authType, adminPass string) interface{} {
    // authType未被使用
    if adminPass != "" {
        return map[string]interface{}{"pwd": adminPass}
    }
    return nil
}

// ✅ 正确：只保留需要被用到的入参
func buildAuthCred(adminPass string) interface{} {
    if adminPass != "" {
        return map[string]interface{}{"pwd": adminPass}
    }
    return nil
}

// ❌ 错误：buildMicroserviceEngineCreateOpts中enterpriseProjectId参数未被使用
func buildMicroserviceEngineCreateOpts(d *schema.ResourceData, enterpriseProjectId string,
	vpcInfo, subnetInfo interface{}) map[string]interface{} {
    // enterpriseProjectId未被使用
    return map[string]interface{}{
        "name": d.Get("name").(string),
        // ...
    }
}

// ✅ 正确：从参数列表中移除未使用的enterpriseProjectId参数
func buildMicroserviceEngineCreateOpts(d *schema.ResourceData,
	vpcInfo, subnetInfo interface{}) map[string]interface{} {
    return map[string]interface{}{
        "name": d.Get("name").(string),
        // ...
    }
}
```

**检查清单**：

- [ ] **自定义辅助函数**的参数列表中是否定义了从未被使用过的入参？
- [ ] **必须符合接口签名的函数**（如CRUD方法）中未使用的参数是否使用下划线`_`进行了标记？
- [ ] Context参数如果未使用，是否标记为`_ context.Context`？
- [ ] ResourceData参数如果未使用，是否标记为`_ *schema.ResourceData`？
- [ ] meta参数如果未使用，是否标记为`_ interface{}`？
- [ ] 函数签名是否保持简洁，没有冗余参数？
- [ ] 是否检查了函数体内部是否使用了所有定义的参数？
- [ ] 测试用例中未使用的参数是否进行了适当标记？

### 22. 未使用函数检查

**说明**：本规范适用于资源、数据源和测试用例的代码，确保代码库中没有未被使用的函数定义，保持代码的简洁性和可维护性。

**设计原则**：

- 资源、数据源和测试用例文件中不应存在未被使用的函数定义
- 未被使用的函数包括：
  + 未被任何地方调用的辅助函数
  + 未被CRUD方法引用的函数（如未在Schema中引用的CreateContext、ReadContext等）
  + 未被测试用例调用的测试辅助函数
- 如果函数确实需要保留（如未来可能使用、接口实现要求等），应添加注释说明原因
- 包外可见的函数（首字母大写）如果未被使用，应检查是否需要在其他包中使用，如果不需要则应删除或改为包内可见

**最佳实践**：

1. **未被使用的辅助函数**

```go
// ❌ 错误：定义了函数但未被使用
func getSubnetById(client *golangsdk.ServiceClient, networkId string) (interface{}, error) {
    // ...
}

func createMicroserviceEngine(cseClient, vpcClient *golangsdk.ServiceClient, d *schema.ResourceData, enterpriseProjectId string) (interface{}, error) {
    // getSubnetById 未被调用
    // ...
}

// ✅ 正确：函数被正确使用
func getSubnetById(client *golangsdk.ServiceClient, networkId string) (interface{}, error) {
    // ...
}

func createMicroserviceEngine(cseClient, vpcClient *golangsdk.ServiceClient, d *schema.ResourceData, enterpriseProjectId string) (interface{}, error) {
    subnetInfo, err := getSubnetById(vpcClient, networkId)
    // ...
}
```

2. **未被使用的CRUD方法**

```go
// ❌ 错误：定义了UpdateContext但未在Schema中引用
func resourceMicroserviceEngineUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    return nil
}

func ResourceMicroserviceEngine() *schema.Resource {
    return &schema.Resource{
        CreateContext: resourceMicroserviceEngineCreate,
        ReadContext:   resourceMicroserviceEngineRead,
        // UpdateContext 未定义，但上面定义了函数
        DeleteContext: resourceMicroserviceEngineDelete,
        // ...
    }
}

// ✅ 正确：如果资源不支持更新且参数使用的NonUpdatable特性，应定义返回为空的UpdateContext并在Schema中引用
func resourceMicroserviceEngineUpdate(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
    return nil
}

func ResourceMicroserviceEngine() *schema.Resource {
    return &schema.Resource{
        CreateContext: resourceMicroserviceEngineCreate,
        ReadContext:   resourceMicroserviceEngineRead,
        UpdateContext: resourceMicroserviceEngineUpdate, // 正确引用
        DeleteContext: resourceMicroserviceEngineDelete,
        // ...
    }
}
```

3. **未被使用的测试辅助函数**

```go
// ❌ 错误：定义了测试辅助函数但未被使用
func getTestConfig() *config.Config {
    // ...
}

func TestResourceMicroserviceEngine(t *testing.T) {
    // getTestConfig 未被调用
    // ...
}

// ✅ 正确：测试辅助函数被正确使用
func getTestConfig() *config.Config {
    // ...
}

func TestResourceMicroserviceEngine(t *testing.T) {
    cfg := getTestConfig()
    // ...
}
```

4. **包外可见但未使用的函数**

```go
// ❌ 错误：包外可见的函数未被使用
func GetMicroserviceEngineById(client *golangsdk.ServiceClient, engineId, epsId string) (interface{}, error) {
    // ...
}

// 该函数在包内未被调用，在其他包中也未被使用

// ✅ 正确：如果需要在测试用例中使用，应保持包外可见
func GetMicroserviceEngineById(client *golangsdk.ServiceClient, engineId, epsId string) (interface{}, error) {
    // ...
}

// 在测试文件中使用
func getMicroserviceEngineFunc(conf *config.Config, state *terraform.ResourceState) (interface{}, error) {
    client, _ := conf.NewServiceClient("cse", state.Primary.Attributes["region"])
    return cse.GetMicroserviceEngineById(client, state.Primary.ID, "")
}

// ✅ 正确：如果不需要在其他包中使用，应改为包内可见
func getMicroserviceEngineById(client *golangsdk.ServiceClient, engineId, epsId string) (interface{}, error) {
    // ...
}
```

5. **需要保留但暂时未使用的函数**

```go
// ✅ 正确：如果函数确实需要保留，应添加注释说明原因
// TODO: This function will be used in the next version when the API supports batch operations.
// Keep it for future use.
func batchCreateMicroserviceEngines(client *golangsdk.ServiceClient, engines []interface{}) (interface{}, error) {
    // ...
}

// ✅ 正确：接口实现要求的函数（即使暂时未使用）
type EngineInterface interface {
    Create(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics
    Read(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics
}

// 实现接口必须定义所有方法，即使某些方法暂时未使用
func (r *EngineResource) Read(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // 接口实现要求，必须定义
    return nil
}
```

6. **检查方法**

**使用Go工具检查未使用的函数**：

```bash
# 使用 go vet 检查（需要配合静态分析工具）
go vet ./...

# 使用 golangci-lint 检查（配置 unused 检查器）
golangci-lint run --enable=unused

# 使用 deadcode 工具检查
deadcode ./...
```

**检查清单**：

- [ ] 资源文件中是否存在未被使用的辅助函数？
- [ ] 数据源文件中是否存在未被使用的辅助函数？
- [ ] 测试用例文件中是否存在未被使用的测试辅助函数？
- [ ] CRUD方法是否都在Schema中正确引用？
- [ ] 包外可见的函数是否在其他包中被使用？如果未使用，是否应改为包内可见？
- [ ] 如果函数需要保留但暂时未使用，是否添加了注释说明原因？
- [ ] 是否存在重复定义的函数？
- [ ] 是否使用工具（如golangci-lint、deadcode等）检查了未使用的函数？

### 23. 工具方法（utils）

**说明**：本规范适用于资源与数据源，建议在实现 CRUD/Read、请求体构建、响应解析、导入与数据转换等场景中优先使用 `huaweicloud/utils` 包提供的通用工具方法，以减少重复代码与边界问题，提升一致性与可维护性。

**设计原则**：

- **优先复用**：在同类场景中优先使用 `utils` 既有方法，而不是手写重复的字段提取、空值过滤、JSON 转换等逻辑。
- **组合使用**：请求体构建通常采用 `ValueIgnoreEmpty` + `RemoveNil`；响应解析通常采用 `FlattenResponse` + `PathSearch`；导入/查找常用 `IsUUID` 区分 ID/名称路径。
- **一致性优先**：同一类字段（如可选字段、集合字段、JSON 字符串承载对象字段）在各资源/数据源中应采用一致的处理方式。

**常用方法与适用场景**：

- **`utils.FlattenResponse()`**：将 `golangsdk` 响应体解析为可被 `PathSearch`/断言使用的数据结构。
  - **适用场景**：所有 `client.Request(...)` 的响应体读取与后续字段提取。
- **`utils.PathSearch()`**：从任意 `interface{}` 中按路径/表达式提取字段值（含嵌套结构、数组、过滤表达式等）。
  - **适用场景**：Read 回填、flatten 方法、build 子方法中提取子参数、分页响应提取 `items/next_marker` 等。
  - **注意**：若后续需要断言，默认值必须**类型匹配**（列表用 `make([]interface{}, 0)`，map 用 `make(map[string]interface{})` 等），避免 panic。
- **`utils.ValueIgnoreEmpty()`**：忽略可选字段的零值/空值输入，避免把空字符串、空切片、空 map 传给 API。
  - **适用场景**：Create/Update 的请求体构建，尤其是可选参数、可选列表/集合、可选 map。
- **`utils.RemoveNil()`**：移除请求体中的 `nil` 字段，减少服务端对空字段的歧义解析。
  - **适用场景**：将 build 方法返回的 `map[string]interface{}` 作为 `RequestOpts.JSONBody` 前统一清理。
- **`utils.ExpandToStringListBySet()`**：将 `*schema.Set` 转为 `[]string`。
  - **适用场景**：schema 为 `TypeSet` 且元素为字符串时（如 AZ、标签 key 列表等）。
- **`utils.StrSliceContains()`**：判断字符串是否在白名单/黑名单集合中。
  - **适用场景**：状态轮询（`StateRefreshFunc`）的目标状态/失败状态判定。
- **`utils.IsUUID()`**：识别输入是否为 UUID。
  - **适用场景**：ImportState 或查询时支持“名称/ID 双模式”，决定是否需要额外 List 查找。
- **`utils.JsonToString()` / `utils.StringToJson()`**：在 schema 以 JSON 字符串表达对象，而 API 真实字段为 object/map 时用于转换。
  - **适用场景**：Read 回填 JSON 字符串字段、Create/Update 将 JSON 字符串转换为 object 字段。
- **`utils.SchemaDesc()`**：生成 Internal/Deprecated 字段的描述模板，保持文案一致性。
  - **适用场景**：Internal 参数/属性、Deprecated 参数/属性。

**最佳实践**：

1. **请求体构建：ValueIgnoreEmpty + RemoveNil 组合**

```go
createOpts := golangsdk.RequestOpts{
	KeepResponseBody: true,
	JSONBody: utils.RemoveNil(map[string]interface{}{
		// Required parameters.
		"name": d.Get("name").(string),
		// Optional parameters.
		"description": utils.ValueIgnoreEmpty(d.Get("description").(string)),
		"az_list":      utils.ValueIgnoreEmpty(utils.ExpandToStringListBySet(d.Get("availability_zones").(*schema.Set))),
	}),
}
```

2. **响应解析：FlattenResponse + PathSearch 组合**

```go
resp, err := client.Request("GET", getPath, &getOpt)
if err != nil {
	return nil, err
}
body, err := utils.FlattenResponse(resp)
if err != nil {
	return nil, err
}
engineId := utils.PathSearch("id", body, "").(string)
```

3. **导入/查找：IsUUID 决定是否走 List**

```go
imported := d.Id()
if !utils.IsUUID(imported) {
	items, err := listEngines(client, d)
	if err != nil {
		return nil, err
	}
	id := utils.PathSearch(fmt.Sprintf("[?name=='%s']|[0].id", imported), items, "").(string)
	if id == "" {
		return nil, fmt.Errorf("unable to find engine by name (%s)", imported)
	}
	d.SetId(id)
}
```

**检查清单**：

- [ ] 请求体构建是否对可选字段使用了 `utils.ValueIgnoreEmpty()`？
- [ ] 请求体是否在发送前通过 `utils.RemoveNil()` 清理了 `nil` 字段？
- [ ] 查询响应是否通过 `utils.FlattenResponse()` 解析后再进行字段提取？
- [ ] 字段提取是否优先使用 `utils.PathSearch()`，并为断言场景提供了类型匹配的默认值？
- [ ] `TypeSet` 字段是否使用 `utils.ExpandToStringListBySet()` 做集合到列表的转换？
- [ ] 状态/轮询判断是否使用 `utils.StrSliceContains()` 统一处理白名单/黑名单？
- [ ] ImportState/按名称导入场景是否使用 `utils.IsUUID()` 判断并按需走 List 查找？

### 24. 企业项目参数

**说明**：本规范适用于接口支持企业项目并需要在资源与数据源中定义 `enterprise_project_id` 的场景

**设计原则**：

- 与企业项目强相关的资源/数据源应在 Schema 中显式定义 `enterprise_project_id`。
- 该参数在代码中统一通过 `cfg.GetEnterpriseProjectID(d)`（或同等封装）获取，不应多处手写解析逻辑。
- 数据源中如该参数需要在 `ReadContext` 回填，允许 `Optional: true` + `Computed: true`，并在成功读取后 `d.Set("enterprise_project_id", ...)`。
- 资源中如该参数参与导入、查询或删除路径，应在 `Read`/`ImportState` 中保证状态回填一致。

**最佳实践**：

**1. 资源中定义企业项目ID（企业项目ID通过请求头进行构造）**

```go

// @API CSE POST /v4/token
// @API CSE POST /v4/{project_id}/registry/microservices
// @API CSE GET /v4/{project_id}/registry/microservices/{service_id}
// @API CSE DELETE /v4/{project_id}/registry/microservices/{service_id}
func ResourceMicroservice() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceMicroserviceCreate,
		ReadContext:   resourceMicroserviceRead,
		UpdateContext: resourceMicroserviceUpdate,
		DeleteContext: resourceMicroserviceDelete,

		Importer: &schema.ResourceImporter{
			StateContext: resourceMicroserviceImportState,
		},

		CustomizeDiff: config.FlexibleForceNew(microserviceNonUpdatableParams),

		Schema: map[string]*schema.Schema{
			// Special parameters.
			// These parameters are used to specify the address that used to request the access token and access the
			// microservice engine.
			"auth_address": {
			
			// ...
			"enterprise_project_id": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				Description: `The enterprise project ID to which the microservice belongs.`,
			},
		},
	}
}

func createMicroservice(client *golangsdk.ServiceClient, d *schema.ResourceData, authInfo MicroserviceEngineAuthInfo) (interface{}, error) {
	var (
		httpUrl    = "v4/{project_id}/registry/microservices"
		createPath = client.Endpoint + httpUrl
	)

	createPath = strings.ReplaceAll(createPath, "{project_id}", microserviceDefaultProjectId)

	createOpts := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders:      buildRequestMoreHeaders(authInfo.EnterpriseProjectId),
		JSONBody:         buildMicroserviceCreateOpts(d),
	}

	// ...
}

func resourceMicroserviceCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	resp, err := createMicroservice(client, d, microserviceEngineAuthInfo)
	if err != nil {
		return diag.Errorf("error creating microservice: %s", err)
	}

	// ...
}

func resourceMicroserviceRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg                        = meta.(*config.Config)
		client                     = common.NewCustomClient(true, d.Get("connect_address").(string))
		microserviceId             = d.Id()
		microserviceEngineAuthInfo = MicroserviceEngineAuthInfo{
			AuthAddress:         getAuthAddress(d),
			AdminUser:           d.Get("admin_user").(string),
			AdminPass:           d.Get("admin_pass").(string),
			EnterpriseProjectId: cfg.GetEnterpriseProjectID(d),
		}
	)

	// ...

	mErr := multierror.Append(nil,
		// ...
		// 在ReadContext中对企业项目ID字段进行值的回填：
		// 1. 如果查询的响应中包含了企业项目信息，则从响应结果中获取对应的值并进行填写
		// 2. 如果查询的响应中不包含企业项目信息，则直接使用从cfg.GetEnterpriseProjectID(d)中获取的值进行填写
		// 此处样例为：响应体中不包含企业项目ID
		d.Set("enterprise_project_id", microserviceEngineAuthInfo.EnterpriseProjectId),
	)

	return diag.FromErr(mErr.ErrorOrNil())
}
```

**2. 资源中定义企业项目ID（企业项目ID通过请求体进行构造）**

```go
func ResourceFunction() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceFunctionCreate,
		
		// ...

		Schema: map[string]*schema.Schema{
			// ...
			"enterprise_project_id": {
				Type:        schema.TypeString,
				Optional:    true,
				Computed:    true,
				Description: `The ID of the enterprise project to which the function belongs.`,
			},
		},
	}
}

func buildCreateFunctionBodyParams(cfg *config.Config, d *schema.ResourceData) map[string]interface{} {
	// ...
	return map[string]interface{}{
		// ...
		"enterprise_project_id": cfg.GetEnterpriseProjectID(d),
	}
}

func createFunction(cfg *config.Config, client *golangsdk.ServiceClient, d *schema.ResourceData) (string, error) {
	httpUrl := "v2/{project_id}/fgs/functions"

	createPath := client.Endpoint + httpUrl
	createPath = strings.ReplaceAll(createPath, "{project_id}", client.ProjectID)
	createOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
		JSONBody: utils.RemoveNil(buildCreateFunctionBodyParams(cfg, d)),
	}

	// ...
}

func resourceFunctionRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ...
	
	mErr := multierror.Append(
		// ...
		// 在ReadContext中对企业项目ID字段进行值的回填：
		// 1. 如果查询的响应中包含了企业项目信息，则从响应结果中获取对应的值并进行填写
		// 2. 如果查询的响应中不包含企业项目信息，则直接使用从cfg.GetEnterpriseProjectID(d)中获取的值进行填写
		// 此处样例为：响应体中包含企业项目ID
		d.Set("enterprise_project_id", utils.PathSearch("enterprise_project_id", function, nil)),
	)

	return nil
}
```

**检查清单**：

- [ ] 是否在需要企业项目维度控制的资源/数据源中定义了 `enterprise_project_id`？
- [ ] 是否统一通过 `cfg.GetEnterpriseProjectID(d)` 获取企业项目参数？
- [ ] 资源和数据源中的参数使用具备回填逻辑，是否使用 `Optional: true` + `Computed: true` 并在 `ReadContext` 中 `d.Set`？
- [ ] 资源导入与读取流程中是否保持 `enterprise_project_id` 状态一致性（`Read`/`ImportState`）？
