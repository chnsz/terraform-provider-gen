# 资源核心规范

## 职责范围

本部分用于 **Terraform Provider Resource** 的代码质量检查，覆盖：资源主函数、CRUD 命名与签名、客户端创建、状态管理、变更检测、重试/轮询、全局变量、删除操作、导入状态、以及可更新/不可更新参数声明完整性检查等。

## 使用建议（检查顺序）

1. **资源主函数与 CRUD 骨架**：结构完整、方法签名/命名规范、必要配置项齐全。
2. **客户端创建模式**：单/多 client 命名与错误返回一致，创建流程与区域获取正确。
3. **状态管理**：`d.SetId`、`d.Set`、`multierror.Append` 使用正确，可选字段与容错策略符合规范。
4. **变更检测/更新逻辑**：`d.HasChange(s)` 分组合理，复杂差异计算抽象清晰，更新后回读策略正确。
5. **重试/轮询**：封装独立方法，超时来自 `d.Timeout`，404/删除场景处理符合约定。
6. **全局变量/删除/导入**：不可更新参数列表、错误码列表位置与命名规范；删除与导入处理清晰完整。
7. **参数完整性检查**：可更新参数全部参与更新逻辑；不可更新参数全部纳入 `FlexibleForceNew`（含嵌套绝对路径）。

### 1. 资源主函数结构

**设计原则**：

- 函数命名格式：`Resource{ResourceName}()` 或 `ResourceV{VersionNumber}{ResourceName}()`
- 在要求定义成特定版本的资源时，版本的V必须大写，版本号为要求的版本号，其中不能包含特殊字符
- 返回类型：`*schema.Resource`
- 必须声明为包外可见的函数（方法名称的首字母大写）

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
    return &schema.Resource{
        CreateContext: resourceV3AgencyCreate,
        ReadContext:   resourceV3AgencyRead,
        UpdateContext: resourceV3AgencyUpdate,
        DeleteContext: resourceV3AgencyDelete,

        CustomizeDiff: config.FlexibleForceNew(v3AgencyNonUpdatableParams),

        Timeouts: &schema.ResourceTimeout{
            Read: schema.DefaultTimeout(2 * time.Minute),
            Update: schema.DefaultTimeout(1 * time.Minute),
            Delete: schema.DefaultTimeout(1 * time.Minute),
        },

        Importer: &schema.ResourceImporter{
            StateContext: schema.ImportStatePassthroughContext,
        },

        Schema: map[string]*schema.Schema{
            // Schema 定义
        },
    }
}
```

**检查清单**：

- [ ] 资源函数是否声明为包外可见（首字母大写）？
- [ ] 函数命名是否符合规范（`Resource{ResourceName}()`或`ResourceV{VersionNumber}{ResourceName}()`）？
- [ ] 版本号V是否大写，版本号是否不包含特殊字符？
- [ ] 返回类型是否为`*schema.Resource`？

### 2. CRUD 函数命名和签名

**设计原则**：

- CRUD函数必须设置为包内可见（首字母小写）
- 函数命名格式：`resource{ResourceName}{Action}` 或 `resourceV{VersionNumber}{ResourceName}{Action}`
- 在要求定义成特定版本的资源时，版本的V必须大写，版本号为要求的版本号，其中不能包含特殊字符
- 函数签名必须依次包含：`ctx context.Context`, `d *schema.ResourceData`, `meta interface{}`, 如果Context等参数未被使用则定义为`_`, 如`_ ctx context.Context`
- 返回值必须为`diag.Diagnostics`
- 如果资源定义了`CustomizeDiff: config.FlexibleForceNew(...)`，则即便所有的参数都不支持更新，也需要定义一个空返回的`UpdateContext`

**最佳实践**：

```go
func resourceV3AgencyCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
}

func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
}

func resourceV3AgencyUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
}

func resourceV3AgencyDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
}
```

**检查清单**：

- [ ] CRUD函数是否设置为包内可见（首字母小写）？
- [ ] 函数命名是否符合规范（`resource{ResourceName}{Action}` 或 `resourceV{VersionNumber}{ResourceName}{Action}`）？
- [ ] 函数签名是否包含`ctx context.Context`作为第一个参数？
- [ ] 函数签名是否包含`d *schema.ResourceData`作为第二个参数？
- [ ] 函数签名是否包含`meta interface{}`作为第三个参数？
- [ ] 返回值是否为`diag.Diagnostics`？

### 3. 客户端创建模式

根据资源所使用API涉及的服务种类设计对应的客户端创建代码，若仅使用到当前服务自身的客户端，则根据标准模式进行设计

**设计原则**：

- 允许使用默认或自定义创建客户端的方法为API请求构造对应服务的客户端（`cfg.NewServiceClient()` 和 `common.NewCustomClient()` 方法）
- 当资源中使用默认的创建客户端方法时需保证资源代码中包含下列的步骤：
  1. 在变量定义的代码块中从 `meta` 获取配置（`cfg = meta.(*config.Config)`），再从配置中获取区域（`region = cfg.GetRegion(d)`）
  2. 根据服务产品名称使用默认的客户端创建方法（`NewServiceClient(serviceProductName, regionName string)`）
- 对于个别不使用IAM鉴权的资源允许其使用自定义的创建客户端方法（`common.NewCustomClient(insecure bool, endpoints ...string)`）
- **客户端变量命名规则**：
  + 当方法中**只使用一个服务的客户端**时，可以使用通用的`client`命名
  + 当方法中**使用多个服务的客户端**时，**必须**为每个客户端添加服务前缀以区分（如`cseClient`, `vpcClient`, `iamClient`等）
  + 同一资源的不同CRUD方法中，如果某个方法使用多个客户端而另一个方法只使用单个客户端，则命名可以不同（多客户端方法使用前缀，单客户端方法使用`client`）
- 错误消息格式：`"error creating {Service} client: %s"`，其中Service为服务产品的简称（如VPC、ModelArts等）
- 使用 `diag.Errorf` 返回错误

**最佳实践**：

```go
// **标准模式**：仅使用当前服务自身的客户端
func resourceV3AgencyDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg      = meta.(*config.Config)
		region   = cfg.GetRegion(d)
	)

    // 正常来说一个资源只需要用到本服务的client，符合这种情况时使用通用的`client`命名
	client, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

    // ...
}

// **多客户端模式**：需要使用多个服务的客户端
func resourceMicroserviceEngineCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    var (
        cfg    = meta.(*config.Config)
        region = cfg.GetRegion(d)
    )

    // 当方法中使用多个服务的客户端时，必须为每个客户端添加服务前缀以区分
    // 例如：需要CSE服务客户端时
    cseClient, err := cfg.NewServiceClient("cse", region)
    if err != nil {
        return diag.Errorf("error creating CSE client: %s", err)
    }
    
    // 例如：需要VPC服务客户端时
    vpcClient, err := cfg.NewServiceClient("vpc", region)
    if err != nil {
        return diag.Errorf("error creating VPC client: %s", err)
    }
    
    // ...
}

// **同一资源的不同方法可以有不同的命名**：
// Create方法使用多个客户端，使用前缀命名
func resourceMicroserviceEngineCreate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
    cseClient, err := cfg.NewServiceClient("cse", region)
    vpcClient, err := cfg.NewServiceClient("vpc", region)
    // ...
}

// Delete方法只使用一个客户端，使用通用命名
func resourceMicroserviceEngineDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    // ...
    client, err := cfg.NewServiceClient("cse", region)
    // ...
}
```

**检查清单**：

- [ ] 创建client前是否从`meta`获取配置（`cfg = meta.(*config.Config)`）？
- [ ] 创建client前是否获取区域（`region = cfg.GetRegion(d)`）？
- [ ] 是否根据服务产品名称使用`NewServiceClient()`创建客户端？
- [ ] 错误消息格式是否符合规范（`"error creating {Service} client: %s"`）？
- [ ] 是否使用`diag.Errorf`返回错误？
- [ ] 多客户端模式下是否为每个client添加了服务前缀？
- [ ] 单客户端模式下是否使用通用的`client`命名？
- [ ] 同一资源的不同CRUD方法中，如果某个方法使用多个客户端而另一个方法只使用单个客户端，命名是否合理（多客户端方法使用前缀，单客户端方法使用`client`）？


### 4. 资源状态管理

**设计原则**：

- 关于字段存储：
  - 使用`d.SetId()`进行设置，确保资源ID唯一且有效；无论资源创建请求后是否存在其他操作，都在该步骤进行一次资源设置，即使此时只有一个临时的任务ID，后续步骤如果ID会发生变化则再次进行设置
  - ReadContext方法以及Import方法中使用`multierror.Append()`处理多个状态设置操作
  - 对于可选的状态设置，使用条件判断避免设置空值
  - 如果资源某个功能对应的参数（通过其他API进行获取的）在获取时遇到了报错，则需要使用日志记录，而不是直接抛出错误，不能影响主流程
  - 参数、属性列表中没有的字段不使用`d.Set()`方法进行设置
- 关于字段获取：
  - 使用`d.Get()`和`d.GetOk()`正确获取状态值
  - 可选字段在获取后应通过`utils.ValueIgnoreEmpty()`方法忽略其默认值的输入（除非列表中要求输入空默认值）

**最佳实践**：

1. **设置资源 ID**

```go
respBody, err := createV3Agency(iamClient, d, domainId)
if err != nil {
    return diag.Errorf("error creating agency: %s", err)
}

agencyId := utils.PathSearch("agency.id", respBody, "").(string)
if agencyId == "" {
    return diag.Errorf("unable to find the agency ID from the API response")
}
d.SetId(agencyId)
```

2. **读取资源 ID**

```go
agencyId := d.Id()
```

3. **设置状态值**

```go
// 单个设置
d.Set("name", utils.PathSearch("agency.name", agency, ""))

// 多个设置（使用 multierror）
mErr := multierror.Append(nil,
    d.Set("name", utils.PathSearch("agency.name", agency, "")),
    d.Set("description", utils.PathSearch("agency.description", agency, "")),
    d.Set("expire_time", utils.PathSearch("agency.expire_time", agency, "")),
    d.Set("create_time", utils.PathSearch("agency.create_time", agency, "")),
    d.Set("duration", normalizeAgencyDuration(utils.PathSearch("agency.duration", agency, ""))),
)
if err = mErr.ErrorOrNil(); err != nil {
    return diag.Errorf("error setting identity agency fields: %s", err)
}
```

4. **条件设置状态值**

当需要根据查询结果的条件设置不同的字段时，使用条件判断：

```go
delegatedDomainName := utils.PathSearch("agency.trust_domain_name", agency, "").(string)
match, _ := regexp.MatchString("^op_svc_[A-Za-z]+$", delegatedDomainName)
if match {
    mErr = multierror.Append(mErr, d.Set("delegated_service_name", delegatedDomainName))
} else {
    mErr = multierror.Append(mErr, d.Set("delegated_domain_name", delegatedDomainName))
}
```

5. **错误处理时的状态设置**

当某些查询操作失败但不影响主流程时，使用日志记录错误但继续设置其他状态：

```go
func resourceV3AgencyRead(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
    domainId := utils.PathSearch("agency.domain_id", agency, "").(string)
    projectRoles, err := listAttachedProjectRolesForV3Agency(client, domainId, agencyId)
    if err != nil {
        log.Printf("[ERROR] error querying the roles attached on project for agency (%s): %s", agencyId, err)
    } else {
        mErr = multierror.Append(mErr, d.Set("project_role", projectRoles))
    }

    domainRoles, err := listAttachedDomainRolesForV3AgencyByDomainId(client, domainId, agencyId)
    if err != nil {
        log.Printf("[ERROR] error querying the roles attached on domain for agency (%s): %s", agencyId, err)
    } else {
        mErr = multierror.Append(mErr, d.Set("domain_roles", utils.PathSearch("[*].display_name", domainRoles,  make([]interface{}, 0)).([]interface{})))
    }

    // ...
}
```

6. **获取状态值**

```go
// 基本获取
name := d.Get("name").(string)

// 可选字段检查
if domainName, ok := d.GetOk("delegated_domain_name"); ok {
    return domainName.(string)
}

// Set 类型
if rawRoles := d.Get("project_role").(*schema.Set); rawRoles.Len() > 0 {
    // ...
}
```

**检查清单**：

- [ ] 资源ID是否从API响应中正确提取并设置？
- [ ] 资源ID提取失败时是否返回了错误？
- [ ] 多个状态设置是否使用`multierror.Append()`处理？
- [ ] 多个状态设置是否使用`diag.FromErr(mErr.ErrorOrNil())`转换错误？
- [ ] 条件设置状态值时是否正确处理了各种情况？
- [ ] 错误处理时的状态设置是否使用日志记录而不中断流程？
- [ ] 状态值的获取是否正确使用了`d.Get()`或`d.GetOk()`？
- [ ] Set类型的状态值获取是否正确进行了类型断言？

### 5. 变更检测

如果资源的更新涉及到多个API接口，则**必须**通过d.HasChange或d.HasChanges方法识别参数的变更，将不同API的变更场景进行区分。

**设计原则**：

- 使用`d.GetChange()`获取变更前后的值
- 对于复杂的Set类型变更，定义专门的差异计算函数
- 差异计算函数应该使用命名返回值（如`removeProjectRoles, addProjectRoles []interface{}`），提高代码可读性
- 差异计算函数应该返回需要删除和需要添加的列表
- 在更新操作中，使用`utils.IsResourceNotFound()`检查404错误，避免因资源已不存在而中断流程
- 先执行删除操作，再执行添加操作，确保变更的正确性

**最佳实践**：

1. **检测单个字段变更**

```go
if d.HasChange("description") {
    // 处理变更
}
```

2. **检测多个字段变更**

```go
if d.HasChanges("delegated_domain_name", "delegated_service_name", "description", "duration") {
    // 处理变更
}
```

3. **在 Update 函数中的完整示例**

```go
func resourceV3AgencyUpdate(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg      = meta.(*config.Config)
		region   = cfg.GetRegion(d)
		agencyId = d.Id()
		domainId = cfg.DomainID
	)

	iamClient, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	if domainId == "" {
		return diag.Errorf("the parameter 'domain_id' in provider-level configuration must be specified")
	}

	// 检测基础字段变更
	if d.HasChanges("delegated_domain_name", "delegated_service_name", "description", "duration") {
		if err = updateV3Agency(ctx, iamClient, agencyId, d, d.Timeout(schema.TimeoutUpdate)); err != nil {
			return diag.Errorf("error updating agency (%s): %s", agencyId, err)
		}
	}

	// 只在需要时查询角色列表
	var parsedRolePairs map[string]string
	if d.HasChanges("project_role", "domain_roles", "all_resources_roles", "enterprise_project_roles") {
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

	if d.HasChange("domain_roles") {
		if err = updateDomainRolesForV3Agency(iamClient, d, parsedRolePairs, domainId, agencyId); err != nil {
			return diag.FromErr(err)
		}
	}

    // ...

	return resourceV3AgencyRead(ctx, d, meta)
}
```

4. **获取变更值**

```go
// 为了避免使用名称new（保留关键字），这里推荐保持以下统一的命名
oldRaw, newRaw = d.GetChange("project_role")
```

5. **变更差异计算函数**

对于复杂的变更检测，可以定义专门的差异计算函数：

```go
func diffChangesOfProjectRolesForV3Agency(oldVal, newVal *schema.Set) (removeProjectRoles, addProjectRoles []interface{}) {
	removeProjectRoles = make([]interface{}, 0)
	addProjectRoles = make([]interface{}, 0)

	oldProjectRolePairs := parseProjectRolesToPairs(oldVal)
	newProjectRolePairs := parseProjectRolesToPairs(newVal)

	// 找出需要删除的角色（在旧值中存在但在新值中不存在）
	for k := range oldProjectRolePairs {
		if _, ok := newProjectRolePairs[k]; !ok {
			removeProjectRoles = append(removeProjectRoles, k)
		}
	}

	// 找出需要添加的角色（在新值中存在但在旧值中不存在）
	for k := range newProjectRolePairs {
		if _, ok := oldProjectRolePairs[k]; !ok {
			addProjectRoles = append(addProjectRoles, k)
		}
	}

	return removeProjectRoles, addProjectRoles
}

// 在Update方法中使用
func updateProjectRolesForV3Agency(client *golangsdk.ServiceClient, d *schema.ResourceData, parsedRolePairs map[string]string,
	domainId, agencyId string) error {
	var (
		oldRaw, newRaw                      = d.GetChange("project_role")
		removeProjectRoles, addProjectRoles = diffChangesOfProjectRolesForV3Agency(oldRaw.(*schema.Set), newRaw.(*schema.Set))
	)

	if len(removeProjectRoles) > 0 {
		if err := detachProjectRolesFromV3Agency(client, parsedRolePairs, removeProjectRoles, domainId, agencyId); err != nil {
			return err
		}
	}

	if len(addProjectRoles) > 0 {
		if err := attachProjectRolesToV3Agency(client, parsedRolePairs, addProjectRoles, domainId, agencyId); err != nil {
			return err
		}
	}

	return nil
}
```

6. **处理404错误（资源已不存在）**：

在更新操作中，如果某个资源已经不存在（返回404），应该忽略该错误而不是中断流程：

```go
err = detachProjectRoleFromV3Agency(client, agencyId, projectId, roleId)
if err != nil && !utils.IsResourceNotFound(err) {
    return fmt.Errorf("error detaching role (%s) by project (%s) from agency (%s): %s",
        roleId, projectId, agencyId, err)
}
```

**检查清单**：

- [ ] 是否使用`d.HasChange()`或`d.HasChanges()`正确检测字段变更？
- [ ] 复杂的Set类型变更是否定义了专门的差异计算函数？
- [ ] 差异计算函数是否使用命名返回值提高可读性？
- [ ] 更新操作中是否使用`utils.IsResourceNotFound()`检查404错误？
- [ ] 是否先执行删除操作，再执行添加操作？

### 6. 重试机制

对于需要重试、轮询的逻辑代码，**必须**将重试逻辑封装为独立的方法，使用`resource.RetryContext`或`resource.StateChangeConf`实现重试和状态轮询，并通过`d.Timeout()`获取超时时间而不是硬编码。

**设计原则**：

- 使用 `d.Timeout(schema.Timeout{Read|Create|Update|Delete})` 获取超时时间，而不是硬编码
- 404 错误的重试应该只在资源是新创建的情况下进行：调用方通过仅在新资源时传入 timeout（且大于 0），未传或为 0 时重试方法内部只调用一次查询接口
- 删除操作中的404错误应该返回`NonRetryableError`，因为资源已不存在
- 其他错误通常不需要重试，除非是临时性错误（如网络超时、服务暂时不可用等）
- 使用 `common.CheckForRetryableError(err)` 检查可重试错误
- 添加 `// lintignore:R006` 注释忽略 linter 警告（resource.RetryContext）
- 添加 `// lintignore:R018` 注释忽略 linter 警告（time.Sleep）
- 将重试逻辑封装为独立方法，便于复用和维护
- 在重试方法中添加注释说明重试的原因和超时时间的用途
- 更新操作成功后，如果需要等待状态稳定，可以在重试成功后添加短暂延迟（通常为10秒）
- 使用 `utils.RemoveNil()` 清理请求体中的 nil 值
- 如果一个方法需要使用到timeout信息，且该方法适用于多个调用阶段（Create、Update、Delete），则应将timeout定义于方法的入参中，根据不同的调用时机传入不同的超时时间

**最佳实践**：

1. **使用 resource.RetryContext**

```go
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

// 在主函数中使用：仅新资源时传入 timeout
var timeout time.Duration
if d.IsNewResource() {
	timeout = d.Timeout(schema.TimeoutRead)
}
agency, err := GetV3AgencyByIdWithRetry(ctx, client, agencyId, timeout)
if err != nil {
    return common.CheckDeletedDiag(d, err, "error retrieving agency")
}
```

2. **更新操作中的重试机制**

更新操作中也可能需要重试机制，特别是在更新后需要等待资源状态稳定时：

```go
func updateV3Agency(ctx context.Context, client *golangsdk.ServiceClient, agencyId string, d *schema.ResourceData,
	timeout time.Duration) error {
	httpUrl := "v3.0/OS-AGENCY/agencies/{agency_id}"

	updatePath := client.Endpoint + httpUrl
	updatePath = strings.ReplaceAll(updatePath, "{agency_id}", agencyId)

	updateOpt := golangsdk.RequestOpts{
		KeepResponseBody: true,
		MoreHeaders: map[string]string{
			"Content-Type": "application/json",
		},
		JSONBody: utils.RemoveNil(buildUpdateV3AgencyBodyParams(d)),
	}

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
	if err != nil {
		return err
	}

	return nil
}
```

**检查清单**：

- [ ] 是否使用`d.Timeout()`获取超时时间而不是硬编码？
- [ ] 404错误的重试是否只在资源是新创建时进行（通过可变参数控制）？
- [ ] 删除操作中的404错误是否返回`NonRetryableError`？
- [ ] 是否使用`common.CheckForRetryableError()`检查可重试错误？
- [ ] 重试逻辑是否封装为独立方法？
- [ ] 重试方法中是否添加了注释说明重试原因和超时时间用途？
- [ ] 是否添加了`// lintignore:R006`注释忽略linter警告（resource.RetryContext）？
- [ ] 是否添加了`// lintignore:R018`注释忽略linter警告（time.Sleep）？
- [ ] 更新操作成功后是否添加了适当的延迟等待状态稳定？
- [ ] 重试方法是否使用可变参数`timeout ...time.Duration`支持可选重试？


**设计原则**：

- Schema 的参数、属性的定义顺序**必须**严格按照以下顺序从上到下依次定义：
  1. **region**（如果存在）
  2. **必选参数**（Required parameters）
  3. **可选参数**（Optional parameters）
  4. **属性**（Attributes，Computed）
  5. **内部参数**（Internal parameters）
  6. **内部属性**（Internal attributes）
  7. **废弃参数**（Deprecated parameters）
  8. **废弃属性**（Deprecated attributes）

**最佳实践**：

```go
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ... CRUD方法声明、CustomizeDiff声明等

		Schema: map[string]*schema.Schema{
			// region 参数（如果存在）
			"region": {
				// ...
			},

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

**检查清单**：

- [ ] Schema参数和属性的定义顺序是否严格按照以下顺序：region（如果存在）→ 必选参数 → 可选参数 → 属性 → 内部参数 → 内部属性 → 废弃参数 → 废弃属性？
- [ ] 各分类之间是否使用注释分隔（如`// Required parameters.`、`// Optional parameters.`等）？
- [ ] 各分类之间是否保持一个空行？
- [ ] 是否只有`region`参数设置了`ForceNew: true`，其他参数都没有设置`ForceNew`？
- [ ] 如果需要不可更新参数，是否通过`CustomizeDiff`（`config.FlexibleForceNew()`）实现，而不是使用`ForceNew`？
- [ ] 如果定义了`CustomizeDiff`，是否在Schema中提供了对应的内部参数`enable_force_new`？

### 7. 全局变量定义

**设计原则**：

- **位置要求**：全局变量必须定义在package块的下方，资源主函数（Schema定义）的上面
- **命名规范**：使用有意义的名称，遵循驼峰命名法
- **类型选择**：根据用途选择合适的类型（字符串列表、整数列表等）
- **注释说明**：对于复杂的全局变量，应该添加注释说明其用途
- **使用场景**：只有在多个地方使用或需要统一管理时才定义为全局变量

#### 7.1 不可更新参数列表

**设计原则**：

- 变量命名格式：`{resourceName}NonUpdatableParams` 或 `v{VersionNumber}{ResourceName}NonUpdatableParams`
- 变量类型：`[]string`（字符串列表）
- 变量位置：位于资源主函数上方，package块下方
- 使用场景：用于`CustomizeDiff`中的`config.FlexibleForceNew()`方法
- 参数来源：URL中的参数及Query Parameters中的参数都是不可更新参数

**最佳实践**：

```go
// 不可更新参数列表（位于资源主函数上方）
var v3AgencyNonUpdatableParams = []string{"name"}

var v3UserInfoNonUpdatableParams = []string{
	"email",
	"mobile",
}

var v5PolicyNonUpdatableParams = []string{
	"name",
	"path",
	"description",
}

// @API IAM POST /v3.0/OS-AGENCY/agencies
func ResourceV3Agency() *schema.Resource {
	return &schema.Resource{
		// ...
		CustomizeDiff: config.FlexibleForceNew(v3AgencyNonUpdatableParams),
		// ...
	}
}
```

**检查清单**：

- [ ] 不可更新参数列表是否定义在资源主函数上方？
- [ ] 变量命名是否符合规范（`{resourceName}NonUpdatableParams`或`v{VersionNumber}{ResourceName}NonUpdatableParams`）？
- [ ] 变量类型是否为`[]string`？
- [ ] 是否在`CustomizeDiff`中使用`config.FlexibleForceNew()`引用该变量？
- [ ] 是否包含了所有URL中的参数及Query Parameters中的参数？

#### 7.2 错误码列表

**设计原则**：

- 变量命名格式：`{resourceName}NotFoundCodes` 或 `v{VersionNumber}{ResourceName}NotFoundCodes`
- 变量类型：`[]string`（字符串列表），即使只有一个错误码也定义为列表
- 变量位置：位于资源主函数上方，package块下方
- 使用场景：用于`ConvertExpectedXXXErrInto404Err()`方法转换非标准404错误
- 错误码来源：根据API实际返回的错误结构确定错误码
- 多个错误码：如果存在多种资源不存在错误（资源本身不存在、父资源不存在等），应定义多个变量

**最佳实践**：

```go
// 错误码列表（位于资源主函数上方）
// 单个错误码也定义为列表
var instanceNotFoundCodes = []string{
	"DDS.0010", // Instance does not exist
}

// 多个错误码
var instanceNotFoundCodes = []string{
	"DDS.0010", // Instance does not exist
	"DDS.0011", // Instance is deleted
}

// 多种资源不存在错误（资源本身和父资源）
var trustedServiceNotFoundErrCodes = []string{
	"ORG.0001", // Trusted service does not exist
	"ORG.0002", // Trusted service is deleted
}

var organizationNotFoundErrCodes = []string{
	"ORG.1001", // Organization does not exist
	"ORG.1002", // Organization is deleted
}

// @API IAM GET /v3.0/OS-AGENCY/agencies/{agency_id}
func ResourceDdsInstance() *schema.Resource {
	return &schema.Resource{
		// ...
	}
}

// ReadContext方法中使用
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

**检查清单**：

- [ ] 错误码列表是否定义在资源主函数上方？
- [ ] 变量命名是否符合规范（`{resourceName}NotFoundErrCodes`或`v{VersionNumber}{ResourceName}NotFoundErrCodes`）？
- [ ] 变量类型是否为`[]string`（即使只有一个错误码也定义为列表）？
- [ ] 是否在`ConvertExpectedXXXErrInto404Err()`方法中使用该变量？
- [ ] 错误码是否根据API实际返回的错误结构确定？
- [ ] 如果存在多种资源不存在错误，是否定义了多个变量？

**总体检查清单**：

- [ ] 全局变量是否定义在package块的下方，资源主函数（Schema定义）的上面？
- [ ] 变量命名是否有意义且清晰，遵循驼峰命名法？
- [ ] 是否根据用途选择了合适的类型（字符串列表、整数列表等）？
- [ ] 对于复杂的全局变量，是否添加了注释说明其用途？
- [ ] 是否只有在多个地方使用或需要统一管理时才定义为全局变量？

### 8. 删除操作处理

**说明**：删除操作的基本要求（方法抽象、重试机制、错误处理等）已在以下规范中说明：
- **方法抽象**：参见 [规范4 - 将一至多个API抽象为一个逻辑方法](#4-资源部分将一至多个api抽象为一个逻辑方法并根据使用需求设置包内可见或包外可见)
- **重试机制**：参见 [规范8 - 重试机制](#8-资源部分重试机制)（删除操作中的404错误应返回`NonRetryableError`）
- **错误处理**：参见 [规范5 - 错误处理规范](#5-资源部分错误处理规范)（使用`CheckDeletedDiag()`处理404错误）

本规范仅说明删除操作特有的处理要求。

**设计原则**：

- **删除前资源解关联**：删除主资源前，如果存在关联资源，应该先检查是否需要强制解关联相关资源
- **关联资源删除的错误处理**：在删除关联资源时，如果资源已不存在（404错误），应该记录警告并继续处理其他资源，禁止直接中断整个处理流程
- **可选操作的错误处理**：对于可选的相关资源删除操作，使用WARN级别日志记录错误但不中断流程

**最佳实践**：

1. **删除前解关联相关资源**

```go
func resourceV3AgencyDelete(ctx context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg      = meta.(*config.Config)
		region   = cfg.GetRegion(d)
		agencyId = d.Id()
		timeout  = d.Timeout(schema.TimeoutDelete)
	)

	client, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	// 可选：强制解关联相关资源
	if d.Get("force_dissociate_v5_policies").(bool) {
		policies, err := listV5AgencyAssociatedPolicies(client, agencyId)
		if err != nil {
			log.Printf("[WARN] error listing associated v5 policies with the agency (%s): %s", agencyId, err)
		} else {
			err = deleteV5PoliciesFromAgency(client, agencyId, policies)
			if err != nil {
				return diag.Errorf("error dissociating v5 policies from the agency (%s): %s", agencyId, err)
			}
		}
	}

	// 调用抽象后的重试方法（参见规范4和规范8）
	err = deleteV3AgencyWithRetry(ctx, client, agencyId, timeout)
	if err != nil {
		return common.CheckDeletedDiag(d, err, fmt.Sprintf("error deleting agency (%s)", agencyId))
	}

	return nil
}
```

2. **删除关联资源时的错误处理**

在删除主资源前，如果需要删除关联资源，应该处理404错误（资源已不存在）：

```go
func deleteV5PoliciesFromAgency(client *golangsdk.ServiceClient, agencyId string, policies []interface{}) error {
	httpUrl := "v5/policies/{policy_id}/detach-agency"

	for _, policy := range policies {
		policyId := utils.PathSearch("id", policy, "").(string)

		detachPath := client.Endpoint + httpUrl
		detachPath = strings.ReplaceAll(detachPath, "{policy_id}", policyId)

		detachOpt := golangsdk.RequestOpts{
			KeepResponseBody: true,
			MoreHeaders: map[string]string{
				"Content-Type": "application/json",
			},
			JSONBody: map[string]interface{}{
				"agency_id": agencyId,
			},
		}

		_, err := client.Request("POST", detachPath, &detachOpt)
		if err != nil {
			if _, ok := err.(golangsdk.ErrDefault404); ok {
				// 资源已不存在，记录警告并继续处理其他资源
				log.Printf("[WARN] the policy (%s) was already detached from the agency (%s)", policyId, agencyId)
				continue
			}
			// Note: here we cannot format the error, otherwise the original status code will be lost
			return err
		}
	}
	return nil
}
```

**检查清单**：

- [ ] 删除前是否检查了是否需要强制解关联相关资源？
- [ ] 删除关联资源时，如果资源已不存在（404错误），是否记录警告并继续处理其他资源（使用`continue`）？
- [ ] 可选的相关资源删除操作是否使用WARN级别日志记录错误但不中断流程？
- [ ] 删除操作的重试逻辑是否遵循规范8的要求（抽象为独立方法，404错误返回`NonRetryableError`）？
- [ ] DeleteContext方法是否遵循规范4的要求（调用抽象后的重试方法，不直接使用`resource.RetryContext`）？
- [ ] 是否使用`common.CheckDeletedDiag`处理资源已删除的情况（遵循规范5的要求）？

### 9. 导入状态处理

**说明**：本规范适用于需要自定义导入状态处理的资源。如果资源的ID可以直接用于查询（不需要额外的父资源信息），则使用`schema.ImportStatePassthroughContext`即可，无需自定义导入方法。

**设计原则**：

- **字符串分割方法选择**：根据实际需求选择合适的字符串分割方法
  + 当只需要分割成固定数量的部分（如2个部分）时，可以使用`strings.SplitN(importedId, "/", 2)`限制分割次数
  + 当需要根据分割后的部分数量进行不同处理（如1个、2个或3个部分）时，可以使用`strings.Split(importedId, "/")`，然后通过`len(parts)`或`switch len(parts)`进行判断
  + **两种方法都是允许的**，应根据实际业务需求选择合适的方法
- **错误处理**：导入ID格式错误时应返回清晰的错误信息，说明期望的格式和实际接收到的格式
- **ID设置**：根据导入ID的格式正确设置资源ID和必要的参数

**最佳实践**：

1. **使用 SplitN（固定数量部分）**

当导入ID只需要分割成固定数量的部分时，使用`SplitN`限制分割次数：

```go
func resourceComponentImportState(_ context.Context, d *schema.ResourceData, _ interface{}) ([]*schema.ResourceData, error) {
	importedId := d.Id()
	parts := strings.SplitN(importedId, "/", 2)
	if len(parts) != 2 {
		return nil, fmt.Errorf("invalid format specified for import ID, want '<application_id>/<id>', but got '%s'", importedId)
	}

	d.SetId(parts[1])
	return []*schema.ResourceData{d}, d.Set("application_id", parts[0])
}
```

2. **使用 Split（可变数量部分）**

当导入ID可能需要分割成不同数量的部分时，使用`Split`并根据部分数量进行不同处理：

```go
func resourceEngineImportState(_ context.Context, d *schema.ResourceData, _ interface{}) ([]*schema.ResourceData, error) {
	importedId := d.Id()
	parts := strings.Split(importedId, "/")
	switch len(parts) {
	case 1:
		// 只有资源ID，使用默认企业项目
		d.SetId(parts[0])
		return []*schema.ResourceData{d}, nil
	case 2:
		// 资源ID和企业项目ID
		d.SetId(parts[0])
		return []*schema.ResourceData{d}, d.Set("enterprise_project_id", parts[1])
	default:
		return nil, fmt.Errorf("invalid format specified for import ID, want '<id>' or '<id>/<enterprise_project_id>', but got '%s'", importedId)
	}
}
```

3. **使用 Split（多个格式支持）**

当导入ID支持多种格式时，使用`Split`并根据部分数量判断格式：

```go
func resourceDataServiceApiImportState(_ context.Context, d *schema.ResourceData, _ interface{}) ([]*schema.ResourceData, error) {
	importedId := d.Id()
	parts := strings.Split(importedId, "/")
	if len(parts) != 2 && len(parts) != 3 {
		return nil, fmt.Errorf("invalid format specified for import ID, must be '<workspace_id>/<dlm_type>/<id>' or "+
			"'<workspace_id>/<id>', but got '%s'", importedId)
	}

	mErr := multierror.Append(nil, d.Set("workspace_id", parts[0]))
	if len(parts) == 2 {
		d.SetId(parts[1])
	}
	if len(parts) == 3 {
		mErr = multierror.Append(mErr, d.Set("dlm_type", parts[1]))
		d.SetId(parts[2])
	}

	return []*schema.ResourceData{d}, mErr.ErrorOrNil()
}
```

**检查清单**：

- [ ] 是否根据实际需求选择了合适的字符串分割方法（`Split`或`SplitN`）？
- [ ] 导入ID格式错误时是否返回了清晰的错误信息（说明期望格式和实际格式）？
- [ ] 是否根据导入ID的格式正确设置了资源ID和必要的参数？
- [ ] 是否处理了所有可能的导入ID格式情况？
- [ ] 多个参数设置是否使用了`multierror.Append()`统一处理？

**重要说明**：
- **`strings.Split`和`strings.SplitN`都是允许的**：应根据实际业务需求选择合适的方法
- **`SplitN`适用场景**：当只需要分割成固定数量的部分时，使用`SplitN`可以限制分割次数，避免不必要的分割
- **`Split`适用场景**：当需要根据分割后的部分数量进行不同处理（如1个、2个或3个部分）时，使用`Split`更灵活

### 10. 检查更新参数是否声明齐全

**设计原则**：

检查资源的可更新参数是否完整参与在更新方法的逻辑中，确保没有遗漏可更新参数的更新处理。

**1. 如何判断一个特殊参数、必选参数、可选参数是否为可更新的参数？**

- 该参数的 schema 定义中**没有**设置 `ForceNew`
- 该参数名称**没有**被定义在 `CustomizeDiff: config.FlexibleForceNew()` 对应引用的字符串数组中
   - **注意**：对于嵌套的子参数，其在数组中需声明**绝对参数路径**，如 `data_center.*.name` 表示 `data_center` 结构体数组下的 `name` 子参数
- 对于一个可更新参数，其**必须**参与在更新方法的逻辑中：`UpdateContext` 或其调用的子方法。
  - **注意**：不是所有的可更新参数都会被定义在 `d.HasChange` 或 `d.HasChanges` 方法中。某些参数的更新可能通过其他方式参与（如作为更新请求体的组成部分、作为条件判断的输入等），但可更新参数必须在 `UpdateContext` 或其调用的子方法中有对应的处理逻辑。
- `UpdateContext` 及其调用的所有子方法中必须包含每个可更新参数的更新处理逻辑
- 在 `CustomizeDiff` 引用的不可更新参数数组中，嵌套子参数**必须**使用绝对路径格式，如 `health_check.*.port`
- 当资源所调用的 API 不支持更新（无更新接口）时，所有输入参数均为不可更新参数（用于其他阶段控制特殊逻辑的可更新参数除外），此时 `UpdateContext` 应直接返回 `nil`，无需实现任何更新逻辑。
  - **注意**：当存在用于删除阶段控制资源逻辑的参数也会被设计成可更新参数，因此这种情况下更新参数会仅对这类参数进行更新（在ReadContext阶段实现值覆盖，仍保证不在UpdateContext中设计更新逻辑）

**最佳实践**

**1. 资源仅部分参数支持更新**

```go
// 仅参数type不支持更新，其余参数均可更新
var v3AclNonUpdatableParams = []string{"type"}

// @API IAM GET /v3.0/OS-SECURITYPOLICY/domains/{domain_id}/api-acl-policy
// @API IAM PUT /v3.0/OS-SECURITYPOLICY/domains/{domain_id}/api-acl-policy
// @API IAM GET /v3.0/OS-SECURITYPOLICY/domains/{domain_id}/console-acl-policy
// @API IAM PUT /v3.0/OS-SECURITYPOLICY/domains/{domain_id}/console-acl-policy
func ResourceV3Acl() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceV3AclCreate,
		ReadContext:   resourceV3AclRead,
		UpdateContext: resourceV3AclUpdate,
		DeleteContext: resourceV3AclDelete,

		CustomizeDiff: config.FlexibleForceNew(v3AclNonUpdatableParams),

		Schema: map[string]*schema.Schema{
			// Required parameters.
			"type": {
				Type:        schema.TypeString,
				Required:    true,
				Description: `The type of the ACL policy.`,
			},

			// Optional parameters.
			"ip_cidrs": {
				Type:         schema.TypeList,
				Optional:     true,
				AtLeastOneOf: []string{"ip_ranges"},
				Elem: &schema.Resource{
					Schema: map[string]*schema.Schema{
						"cidr": {
							Type:         schema.TypeString,
							Required:     true,
							ValidateFunc: utils.ValidateCIDR,
							Description:  `The IPv4 CIDR block which allow access through console or API.`,
						},
						"description": {
							Type:        schema.TypeString,
							Optional:    true,
							Description: `The description of the IPv4 CIDR block.`,
						},
					},
				},
				Description: `The list of IPv4 CIDR blocks from which console access or API access is allowed.`,
			},
			// ...

			// Internal parameters.
			"enable_force_new": {
				Type:         schema.TypeString,
				Optional:     true,
				ValidateFunc: validation.StringInSlice([]string{"true", "false"}, false),
				Description: utils.SchemaDesc(
					`Whether to allow parameters that do not support changes to have their change-triggered behavior set to 'ForceNew'.`,
					utils.SchemaDescInput{
						Internal: true,
					},
				),
			},

			// Internal attributes.
			"ip_ciders_order": {
				Type:             schema.TypeList,
				Optional:         true,
				Computed:         true,
				DiffSuppressFunc: utils.SuppressDiffAll,
				Elem: &schema.Resource{
					Schema: map[string]*schema.Schema{
						"cidr": {
							Type:        schema.TypeString,
							Computed:    true,
							Description: `The origin IPv4 CIDR block.`,
						},
					},
				},
				Description: utils.SchemaDesc(
					`The origin list of IPv4 CIDR blocks that used to reorder the 'ip_cidrs' parameter.`,
					utils.SchemaDescInput{
						Internal: true,
						Computed: true,
					},
				),
			},
			// ...
		}
	}
}

// 在该子方法中，根据ip_cidrs的本地值，请求了相关API，故能确定ip_cidrs是可更新参数
func updateV3AclPolicy(client *golangsdk.ServiceClient, d *schema.ResourceData, domainId string) error {
	var (
		updateOpts = &acl.ACLPolicy{}
		err        error
	)

	if addressNetmasks, ok := d.Get("ip_cidrs").([]interface{}); ok && len(addressNetmasks) > 0 {
		netmasksList := make([]acl.AllowAddressNetmasks, 0, len(addressNetmasks))
		for _, v := range addressNetmasks {
			netmasksList = append(netmasksList, acl.AllowAddressNetmasks{
				AddressNetmask: v.(map[string]interface{})["cidr"].(string),
				Description:    v.(map[string]interface{})["description"].(string),
			})
		}
		updateOpts.AllowAddressNetmasks = netmasksList
	}

	if ipRanges, ok := d.Get("ip_ranges").([]interface{}); ok && len(ipRanges) > 0 {
		ipRangesList := make([]acl.AllowIPRanges, 0, len(ipRanges))
		for _, v := range ipRanges {
			ipRangesList = append(ipRangesList, acl.AllowIPRanges{
				IPRange:     v.(map[string]interface{})["range"].(string),
				Description: v.(map[string]interface{})["description"].(string),
			})
		}
		updateOpts.AllowIPRanges = ipRangesList
	}

	switch d.Get("type").(string) {
	case "console":
		_, err = acl.ConsoleACLPolicyUpdate(client, updateOpts, domainId).ConsoleExtract()
	case "api":
		_, err = acl.APIACLPolicyUpdate(client, updateOpts, domainId).APIExtract()
	}

	return err
}

// ...

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

	if err = d.Set("ip_ciders_order", buildV3AclIpCidersOrder(d)); err != nil {
		log.Printf("[ERROR] error setting the ip_ciders_order field after updating ACL: %s", err)
	}

	// ...
	return resourceV3AclRead(ctx, d, meta)
}
```

**检查清单**：

- [ ] 每个可更新参数（满足：无 ForceNew 且不在 FlexibleForceNew 数组中）是否都参与（通过 `d.Get` 方法或 `utils.PathSearch` 方法调用）在 `UpdateContext` 或其调用的子方法中？

### 11. 检查不可更新参数是否声明齐全

**设计原则**：

在完成可更新参数检查后，需进一步检查资源的不可更新参数是否已完整补充到 `CustomizeDiff: config.FlexibleForceNew()` 对应的字符串数组中，确保没有遗漏应声明为不可更新的参数。

**1. 如何判断一个特殊参数、必选参数、可选参数是否为不可更新的参数？**

依据（满足以下**任一条件**即为不可更新参数）：

- 该参数出现在 API 的 **URI 路径**中（URL path parameters，如 `{project_id}`、`{resource_id}` 等）
- 该参数出现在 API 的 **Query Parameters** 中
- 该资源所调用的 **更新接口**不支持对该参数进行修改（即 API 层面不支持更新）

**2. 不可更新参数的处理要求**：

对于所有不可更新参数，其**必须**被定义在 `CustomizeDiff: config.FlexibleForceNew()` 对应引用的字符串数组中。

- **注意**：根据规范，除 `region` 参数外，不应在 schema 中直接设置 `ForceNew: true`，应通过 `CustomizeDiff` 统一管理不可更新参数
- **嵌套子参数**：对于嵌套的子参数，需在数组中声明**绝对参数路径**，如 `health_check.*.mode` 表示 `health_check` 结构体下的 `mode` 子参数

**3. 检查顺序**：

- 先执行 [规范 10 - 检查更新参数是否齐全](#10-检查更新参数是否齐全)，识别出所有可更新参数
- 再执行本规范：将 Schema 中除可更新参数、属性（Computed）、内部参数、废弃参数外的所有参数，与不可更新参数数组进行比对，确认无遗漏

**最佳实践**

**1. 所有参数均不支持更新**：

```go
var (
	// 资源中所有的没标记ForceNew的参数都被定义在该数组中
	microserviceInstanceNonUpdatableParams = []string{
		"auth_address",
		"connect_address",
		"admin_user",
		"admin_pass",
		"microservice_id",
		"host_name",
		"endpoints",
		"version",
		"properties",
		"health_check",
		// 对于 `TypeList` 且 `MaxItems: 1` 的对象类型参数，父参数与子参数均需声明。父参数声明参数名（如 `health_check`），子参数使用 `parent.*.child` 格式（如 `health_check.*.mode`），其中 `*` 表示该列表下的任意元素。
		"health_check.*.mode",
		"health_check.*.interval",
		"health_check.*.max_retries",
		"health_check.*.port",
		"data_center",
		"data_center.*.name",
		"data_center.*.region",
		"data_center.*.availability_zone",
	}
)

// @API CSE GET /v4/token
// @API CSE POST /v4/{project_id}/registry/microservices/{service_id}/instances
// @API CSE GET /v4/{project_id}/registry/microservices/{service_id}/instances/{instance_id}
// @API CSE DELETE /v4/{project_id}/registry/microservices/{service_id}/instances/{instance_id}
func ResourceMicroserviceInstance() *schema.Resource {
	return &schema.Resource{
		CreateContext: resourceMicroserviceInstanceCreate,
		ReadContext:   resourceMicroserviceInstanceRead,
		UpdateContext: resourceMicroserviceInstanceUpdate,
		DeleteContext: resourceMicroserviceInstanceDelete,

		CustomizeDiff: config.FlexibleForceNew(microserviceInstanceNonUpdatableParams),
		// ...
	}
}

// CSE 微服务实例 API 无更新接口，所有参数均为不可更新，UpdateContext 空实现
func resourceMicroserviceInstanceUpdate(_ context.Context, _ *schema.ResourceData, _ interface{}) diag.Diagnostics {
	return nil
}
```

**检查清单**：

- [ ] 是否已识别出所有不可更新参数（URI 路径参数、Query 参数、API 不支持更新的参数）？
- [ ] 所有不可更新参数是否均已补充到 `FlexibleForceNew` 引用的字符串数组中？
- [ ] 嵌套子参数是否使用了绝对路径格式（如 `parent.*.child`）？
- [ ] 是否避免了在 schema 中直接设置 `ForceNew: true`（除 `region` 外），统一通过 `CustomizeDiff` 管理？
