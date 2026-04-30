# Terraform Provider 代码质量规范检查最佳实践（Skill 操作指引）

本 Skill 用于对 Terraform Provider（资源/数据源）代码进行**质量规范检查**。

本规范基于以下 [华为云 Provider](https://github.com/huaweicloud/terraform-provider-huaweicloud) 资源、数据源文件提供代码质量指导：
- `huaweicloud/services/iam/resource_huaweicloud_identity_agency.go` （资源参考文件）
- `huaweicloud/services/cse/resource_huaweicloud_cse_microservice.go` （资源参考文件）
- `huaweicloud/services/cse/resource_huaweicloud_cse_microservice_engine.go` （资源参考文件）
- `huaweicloud/services/cse/resource_huaweicloud_cse_microservice_instance.go` （资源参考文件）
- `huaweicloud/services/fgs/resource_huaweicloud_fgs_function.go` （资源参考文件）
- `huaweicloud/services/iam/data_source_huaweicloud_identity_roles.go` （数据源参考文件）

## 规范分类说明

本规范文档包含以下三类规范：

- **资源核心规范**：适用于 Terraform Provider 资源（Resource）的代码规范
- **数据源核心规范**：适用于 Terraform Provider 数据源（DataSource）的代码规范
- **通用规范**：适用于资源和数据源的通用代码规范

为避免上下文过长，本 Skill 仅保留**操作指引**；对应的**拆分正文**位于 `./references/best-practices/`。
建议在以下场景对照其检查入口与清单进行核对，决定后续参考正文的加载：

- **新建 Resource/Data Source 后**：完成主函数骨架、Schema 初稿、CRUD/Read 主流程与 `list/get`、`buildQueryParams/buildBody`、`flatten` 等关键
  方法后，进行一次**资源核心规范**和**通用规范**的自检，提前发现结构/顺序/错误处理问题，降低返工成本。
- **整改/重构已有代码后**：涉及 Schema 分类顺序调整、分页/轮询逻辑重写、错误处理（404/非标404转换）、更新变更检测（`d.HasChange(s)`）拆分、
  `CustomizeDiff/FlexibleForceNew` 参数补全等，优先对照最佳实践的**通用规范**清单进行自检。


## 使用方式（推荐流程）

1. 确认你要检查的是 **资源（Resource）** 还是 **数据源（Data Source）** ，根据类型从对应检查入口加载匹配的Skill。
2. 无论是 **资源（Resource）** 还是 **数据源（Data Source）** ，统一根据 **通用最佳实践** 对代码进行分析。
3. 打开对应的“检查索引文件”，按其“建议检查步骤”逐条核对，并在需要细节时再跳转到正文小节。

## 资源核心最佳实践（检查入口）

- **职责范围**：用于对 Terraform Provider **资源（Resource）** 代码进行质量规范检查与一致性约束，重点覆盖：
  - 资源主函数结构：`Resource{X}()` / `ResourceV{N}{X}()` 命名、`*schema.Resource` 返回、Schema 组织、Timeout/Importer/CustomizeDiff 等配置是否齐全且摆放合理
  - CRUD 函数命名与签名：`resource{X}{Action}` / `resourceV{N}{X}{Action}`、入参顺序与未使用变量 `_` 规范、返回 `diag.Diagnostics`、是否需要空 `UpdateContext`
  - 客户端创建模式：从 `meta` 获取 `cfg`/`region`、单/多 client 命名规则、错误消息格式、`diag.Errorf` 返回方式
  - 资源状态管理：`d.SetId()`、`d.Get`/`d.GetOk`、`d.Set` 回填约束、`multierror.Append()` 聚合回填、可选字段与容错日志策略
  - 变更检测与更新逻辑：`d.HasChange(s)` 的分组与差异计算抽象、更新后回读策略、404/NotFound 场景处理
  - 重试与轮询：`resource.RetryContext` / `resource.StateChangeConf` 的使用边界、超时来自 `d.Timeout()`、删除阶段 404 的处理约束
  - 全局变量：不可更新参数列表、错误码列表等的位置、命名、引用一致性
  - 删除与导入：删除前解关联、删除的错误处理与重试封装、导入 ID 解析策略（Split/SplitN）、自定义导入的错误提示与字段回填
  - 参数完整性检查：可更新参数是否都参与更新逻辑；不可更新参数是否完整纳入 `CustomizeDiff: config.FlexibleForceNew()`（含嵌套绝对路径）
- **作用**：提供资源检查的分步流程与细节正文（设计原则/最佳实践/检查清单），用于统一资源代码风格、减少缺陷与返工
- **什么时候参考**：
  - 新资源生成阶段：完成 Resource 主函数与 CRUD 骨架、Schema 初稿、导入/Timeout/CustomizeDiff 等关键结构后，用于快速自检是否符合规范
  - 资源整改阶段：对已有资源进行结构对齐与质量修复时（例如补充 Update/删除/导入、修复状态回填、重试/轮询逻辑整理、不可更新参数补全等）
  - 编码过程中的特殊自检场景：当你正在实现/修改以下任一类逻辑时应触发参考并对照检查清单逐项核对
    - Schema 分类与顺序、ForceNew/FlexibleForceNew 策略调整
    - Update 变更检测分组（`d.HasChange(s)`）、差异计算与更新后回读
    - 404/NotFound 与非标准 404 错误处理、删除阶段处理边界
    - 重试/轮询与超时（`d.Timeout()`）使用、全局变量（错误码/不可更新参数）补齐
- **检查索引文件**：`./references/best-practices/resources.md`

## 数据源核心最佳实践（检查入口）

- **职责范围**：用于对 Terraform Provider **数据源（DataSource）** 代码进行质量规范检查与一致性约束，重点覆盖：
  - 数据源主函数结构：`DataSource{Xs}()` / `DataSourceV{N}{Xs}()` 命名与返回 `*schema.Resource`，仅包含 `ReadContext`，不包含资源专有配置（CRUD/CustomizeDiff/Timeouts/Importer）
  - ReadContext 命名与签名：`dataSource{Xs}Read` / `dataSourceV{N}{Xs}Read`、入参顺序与 `_` 规范、返回 `diag.Diagnostics`、错误格式与上下文信息
  - 客户端创建模式：从 `meta` 获取 `cfg`/`region`、创建 client 的统一方式、错误消息格式与 `diag.Errorf` 返回
  - ID 设置规范：查询成功后生成随机 UUID 并 `d.SetId()`，失败时返回清晰错误
  - 查询参数构建：查询参数拼接函数的抽象与格式统一、空值处理、避免引入不必要的参数
  - 数据扁平化：`flatten` 函数职责单一、嵌套对象递归扁平化、空输入返回 `nil`、字段提取使用 `utils.PathSearch`
- **作用**：提供数据源检查的分步流程与细节正文，确保数据源“只读、可复现、回填正确、结构稳定”
- **什么时候参考**：
  - 新数据源生成阶段：完成 DataSource 主函数、Schema 初稿、ReadContext 查询与回填骨架后，用于自检“只读结构”和回填是否符合规范
  - 数据源整改阶段：对已有数据源进行规范对齐与缺陷修复时（例如 ID 设置补齐、查询参数构建重构、flatten 递归补全、回填字段遗漏修复等）
  - 编码过程中的特殊自检场景：当你正在实现/修改以下任一类逻辑时应触发参考并对照检查清单逐项核对
    - ReadContext 签名/错误返回与上下文信息、客户端创建统一性
    - 查询参数构建函数抽象与空值处理
    - flatten 结构递归、一致性与空输入返回约束
- **检查索引文件**：`./references/best-practices/datasources.md`

## 通用最佳实践（检查入口）

- **职责范围**：用于对资源与数据源共享的通用代码质量问题进行检查与统一约束，重点覆盖：
  - 方法抽象与可见性：主流程保持简洁、复杂逻辑抽象为单一职责方法、导出/不导出有依据（测试/跨包复用/数据源复用资源方法）
  - 辅助函数体系：请求体构建/查询参数构建/flatten/数据转换/Attach-Detach/查询辅助等函数的命名、入参/返回与职责边界
  - Schema 定义：参数/属性分类顺序与分隔、ForceNew 与 FlexibleForceNew 的使用边界、Optional/Computed 组合等约束
  - 错误处理：`diag.Errorf`/`diag.FromErr`/`multierror.Append`、资源不存在判断、非标准404转换、删除态（如 DELETED）处理等
  - HTTP/URL：请求封装模式、占位符替换完整性、OkCodes 规则、请求体清理（`utils.RemoveNil`）等
  - 轮询与分页：状态轮询封装、page/marker/offset 分页处理模式
  - 日志与变量组织：日志级别与信息密度、变量命名与组织方式、避免与包名冲突等
  - 并发/性能/类型转换/工程性检查：锁的粒度与范围、性能优化点、断言与类型转换安全、行长度、函数参数规范、unused/deadcode 等检查
  - 企业项目参数：EP 参数的获取、回填与使用约束
- **作用**：提供资源与数据源共享部分的统一检查基线，减少“各写各的”导致的可维护性问题
- **什么时候参考**：
  - 新资源/新数据源编码阶段：在抽象子方法、封装 HTTP 请求、实现分页/轮询、整理 Schema 与错误处理等共性模块时，用于统一实现方式
  - 资源/数据源整改阶段：对已有代码进行通用问题治理时（例如错误处理统一、HTTP/URL 构建规范化、分页/轮询封装补齐、Schema 分类与组织调整等）
  - 编码过程中的特殊自检场景：当你正在实现/修改以下任一类逻辑时应触发参考并对照检查清单逐项核对
    - 子方法抽象与导出可见性、辅助函数职责边界
    - 错误处理体系（diag/multierror/404 判断/非标准404转换/删除态）
    - HTTP/URL 请求封装、OkCodes、`utils.RemoveNil`、日志与变量组织
    - 分页/轮询、并发锁、性能与类型断言等工程性约束
- **检查索引文件**：`./references/best-practices/common.md`
