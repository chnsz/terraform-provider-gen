# 数据源核心规范

## 职责范围

本部分用于 **Terraform Provider Data Source** 的代码质量检查，覆盖：数据源主函数结构、ReadContext 命名/签名、客户端创建模式、ID 设置规范、查询参数构建与数据扁平化函数等。

## 使用建议（检查顺序）

1. **数据源主函数结构**：只包含 `ReadContext`，不包含 CRUD、`CustomizeDiff`、`Timeouts`、`Importer` 等资源专有配置。
2. **ReadContext 签名与返回**：命名/签名规范、`diag.Diagnostics` 返回、错误上下文信息完整。
3. **客户端创建**：`cfg/region` 获取与 client 创建方式统一、错误消息格式一致。
4. **ID 设置**：查询成功后设置随机 UUID，失败时返回错误。
5. **查询参数/扁平化**：参数构建抽象合理；`flatten` 递归清晰，空输入返回 `nil`。

### 1. 数据源主函数结构

**设计原则**：

- 函数命名格式：`DataSource{ResourceName}()`或 `DataSourceV{VersionNumber}{ResourceName}()` (ResourceName采用复数名称) 
- 在要求定义成特定版本的数据源时，版本的V必须大写，版本号为要求的版本号，其中不能包含特殊字符
- 返回类型：`*schema.Resource`
- 必须声明为包外可见的函数（方法名称的首字母大写）
- 数据源只包含 `ReadContext`，不包含 `CreateContext`、`UpdateContext`、`DeleteContext`
- 数据源不包含 `CustomizeDiff`、`Timeouts`、`Importer` 等配置

**最佳实践**：

```go
func DataSourceV3Roles() *schema.Resource {
	return &schema.Resource{
		ReadContext: dataSourceV3RolesRead,

		Schema: map[string]*schema.Schema{
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
						"name": {
							Type:        schema.TypeString,
							Computed:    true,
							Description: `The name of the role.`,
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

- [ ] 数据源函数是否声明为包外可见（首字母大写）？
- [ ] 函数命名是否符合规范（`DataSource{ResourceName}()`或`DataSourceV{VersionNumber}{ResourceName}()`, ResourceName必须采用复数名称）？
- [ ] 版本号V是否大写，版本号是否不包含特殊字符？
- [ ] 返回类型是否为`*schema.Resource`？
- [ ] 是否只包含`ReadContext`，不包含其他CRUD方法？
- [ ] 是否不包含`CustomizeDiff`、`Timeouts`、`Importer`等配置？

### 2. ReadContext 函数命名和签名

**设计原则**：

- ReadContext函数必须设置为包内可见（首字母小写）
- 函数命名格式：`dataSource{ResourceName}Read` 或 `dataSourceV{VersionNumber}{ResourceName}Read` (ResourceName采用复数名称) 
- 在要求定义成特定版本的数据源时，版本的V必须大写，版本号为要求的版本号，其中不能包含特殊字符
- 函数签名必须依次包含：`ctx context.Context`, `d *schema.ResourceData`, `meta interface{}`，方法内未使用的需标记成 `_`, 如Context未被使用则定义为`_ context.Context`
- 返回值必须为`diag.Diagnostics`

**最佳实践**：

```go
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

**检查清单**：

- [ ] ReadContext函数是否设置为包内可见（首字母小写）？
- [ ] 函数命名是否符合规范（`dataSource{ResourceName}Read`或`dataSourceV{VersionNumber}{ResourceName}Read`，ResourceName必须采用复数名称）？
- [ ] 函数签名是否包含`ctx context.Context`作为第一个参数（未使用时使用`_`）？
- [ ] 函数签名是否包含`d *schema.ResourceData`作为第二个参数？
- [ ] 函数签名是否包含`meta interface{}`作为第三个参数？
- [ ] 返回值是否为`diag.Diagnostics`？

### 3. 客户端创建模式

数据源的客户端创建模式与资源相同，遵循相同的设计原则。

**设计原则**：

- 创建客户端前需要在变量定义的代码块中从 `meta` 获取配置（`cfg = meta.(*config.Config)`），再从配置中获取区域（`region = cfg.GetRegion(d)`）
- 根据服务产品名称使用默认的客户端创建方法（默认方法 `NewServiceClient()` 或自定义方法 `common.NewCustomClient()`）
- 错误消息格式：`"error creating {Service} client: %s"`，其中Service为服务产品的简称
- 使用 `diag.Errorf` 返回错误

**最佳实践**：

```go
func dataSourceV3RolesRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	var (
		cfg    = meta.(*config.Config)
		region = cfg.GetRegion(d)
	)

	client, err := cfg.NewServiceClient("iam", region)
	if err != nil {
		return diag.Errorf("error creating IAM client: %s", err)
	}

	// ...
}
```

**检查清单**：

- [ ] 创建client前是否从`meta`获取配置（`cfg = meta.(*config.Config)`）？
- [ ] 创建client前是否获取区域（`region = cfg.GetRegion(d)`）？
- [ ] 是否根据服务产品名称使用`NewServiceClient()`创建客户端？
- [ ] 错误消息格式是否符合规范（`"error creating {Service} client: %s"`）？
- [ ] 是否使用`diag.Errorf`返回错误？

### 4. ID 设置规范

数据源必须设置ID，使用随机UUID生成。

**设计原则**：

- 数据源必须调用 `d.SetId()` 设置ID
- 使用 `uuid.GenerateUUID()` 生成随机UUID作为ID
- ID设置应在查询成功后、设置属性之前进行
- 如果UUID生成失败，应返回错误

**最佳实践**：

```go
func dataSourceV3RolesRead(_ context.Context, d *schema.ResourceData, meta interface{}) diag.Diagnostics {
	// ... 查询逻辑

	roles, err := listV3Roles(client, d)
	if err != nil {
		return diag.Errorf("error querying roles: %s", err)
	}

	randomUUID, err := uuid.GenerateUUID()
	if err != nil {
		return diag.Errorf("unable to generate ID: %s", err)
	}
	d.SetId(randomUUID)

	// ... 设置属性
}
```

**检查清单**：

- [ ] 是否调用`d.SetId()`设置ID？
- [ ] 是否使用`uuid.GenerateUUID()`生成UUID？
- [ ] ID设置是否在查询成功后进行？
- [ ] 是否处理了UUID生成失败的情况？

### 5. 查询参数构建

数据源通常需要构建查询参数，应抽象为独立方法。

**设计原则**：

- 查询参数构建应抽象为独立方法，命名格式：`build{ObjectName}QueryParams`
- 方法返回类型为 `string`
- 使用 `fmt.Sprintf` 构建查询参数字符串
- 参数之间使用 `&` 连接
- 如果参数为空，返回空字符串

**最佳实践**：

```go
func buildV3RolesQueryParams(d *schema.ResourceData) string {
	res := ""

	// 必填参数拼接

	// 可选参数的拼接
	if v, ok := d.GetOk("display_name"); ok {
		res = fmt.Sprintf("%s&display_name=%v", res, v)
	}
	if v, ok := d.GetOk("name"); ok {
		res = fmt.Sprintf("%s&name=%v", res, v)
	}
	if v, ok := d.GetOk("catalog"); ok {
		res = fmt.Sprintf("%s&catalog=%v", res, v)
	}
	// ... 其他参数

	return res
}
```

**检查清单**：

- [ ] 查询参数构建是否抽象为独立方法？
- [ ] 方法命名是否符合规范（`build{ObjectName}QueryParams`）？
- [ ] 返回类型是否为`string`？
- [ ] 是否使用`fmt.Sprintf`构建查询参数？
- [ ] 参数之间是否使用`&`连接？

### 6. 数据扁平化函数

数据源需要将API返回的数据转换为Terraform Schema格式，应抽象为扁平化函数。

**设计原则**：

- 数据扁平化应抽象为独立方法，命名格式：`flatten{ObjectName}`
- 方法入参：对象类型为 `map[string]interface{}`，列表类型为 `[]interface{}`
- 方法返回：`[]map[string]interface{}`（Terraform Schema格式）
- 使用 `utils.PathSearch` 提取字段值
- 如果输入为空，返回 `nil`
- 嵌套对象需要定义对应的扁平化方法，命名格式：`flatten{ParentObjectName}{ChildObjectName}`（递归）

**最佳实践**：

```go
// 嵌套对象的扁平化
func flattenV3RoleLinks(links map[string]interface{}) []map[string]interface{} {
	if len(links) < 1 {
		return nil
	}

	return []map[string]interface{}{
		{
			"self":     utils.PathSearch("self", links, nil),
			"previous": utils.PathSearch("previous", links, nil),
			"next":     utils.PathSearch("next", links, nil),
		},
	}
}

// 列表对象的扁平化
func flattenV3Roles(roles []interface{}) []map[string]interface{} {
	if len(roles) < 1 {
		return nil
	}

	result := make([]map[string]interface{}, 0, len(roles))
	for _, role := range roles {
		createdAt, _ := strconv.ParseInt(utils.PathSearch("created_time", role, "").(string), 10, 64)
		updatedAt, _ := strconv.ParseInt(utils.PathSearch("updated_time", role, "").(string), 10, 64)

		result = append(result, map[string]interface{}{
			"id":             utils.PathSearch("id", role, nil),
			"name":           utils.PathSearch("name", role, nil),
			"display_name":   utils.PathSearch("display_name", role, nil),
			"description":    utils.PathSearch("description", role, nil),
			"description_cn": utils.PathSearch("description_cn", role, nil),
			"catalog":        utils.PathSearch("catalog", role, nil),
			"type":           utils.PathSearch("type", role, nil),
			"flag":           utils.PathSearch("flag", role, nil),
			"domain_id":      utils.PathSearch("domain_id", role, nil),
			"policy":         utils.JsonToString(utils.PathSearch("policy", role, nil)),
			"created_at":     utils.FormatTimeStampRFC3339(createdAt/1000, false),
			"updated_at":     utils.FormatTimeStampRFC3339(updatedAt/1000, false),
			"links": flattenV3RoleLinks(
				utils.PathSearch("links", role, make(map[string]interface{})).(map[string]interface{})),
		})
	}

	return result
}
```

**检查清单**：

- [ ] 数据扁平化是否抽象为独立方法？
- [ ] 方法命名是否符合规范（`flatten{ObjectName}`）？
- [ ] 方法入参类型是否正确（对象为`map[string]interface{}`，列表为`[]interface{}`）？
- [ ] 方法返回类型是否为`[]map[string]interface{}`？
- [ ] 是否使用`utils.PathSearch`提取字段值？
- [ ] 输入为空时是否返回`nil`？
- [ ] 嵌套对象是否定义了对应的扁平化方法？
