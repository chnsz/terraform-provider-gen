# 单元测试开发技能

本文档介绍 terraform-provider-huaweicloud 项目的验收测试开发技能。

## 测试框架概述

项目使用 Terraform Plugin SDK v2 提供的测试框架进行验收测试，测试文件位于 `services/acceptance/` 目录下。

### 测试框架核心组件

```go
import (
    "testing"
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/resource"
    "github.com/hashicorp/terraform-plugin-sdk/v2/terraform"
)
```

## 测试文件结构

### 文件命名

测试文件命名规则：
- Resource 测试：`resource_huaweicloud_<name>_test.go`
- Data Source 测试：`data_source_huaweicloud_<name>_test.go`

### 目录结构

```
huaweicloud/services/acceptance/
├── acceptance.go              # 测试框架和公共方法
├── resource_check.go          # 资源检查工具
├── common/                    # 公共测试资源
│   └── resources.go
└── vpc/                       # VPC 服务测试
    ├── resource_huaweicloud_vpc_test.go
    └── data_source_huaweicloud_vpcs_test.go
```

## Resource 测试

### 基本测试结构

```go
func TestAccVpcV1_basic(t *testing.T) {
    var vpc vpcs.Vpc

    rName := acceptance.RandomAccResourceName()
    resourceName := "huaweicloud_vpc.test"

    resource.ParallelTest(t, resource.TestCase{
        PreCheck:          func() { acceptance.TestAccPreCheck(t) },
        ProviderFactories: acceptance.TestAccProviderFactories,
        CheckDestroy:      testAccCheckVpcV1Destroy,
        Steps: []resource.TestStep{
            {
                Config: testAccVpcV1_basic(rName),
                Check: resource.ComposeTestCheckFunc(
                    testAccCheckVpcV1Exists(resourceName, &vpc),
                    resource.TestCheckResourceAttr(resourceName, "name", rName),
                    resource.TestCheckResourceAttr(resourceName, "cidr", "192.168.0.0/16"),
                    resource.TestCheckResourceAttr(resourceName, "status", "OK"),
                ),
            },
            {
                ResourceName:      resourceName,
                ImportState:       true,
                ImportStateVerify: true,
            },
        },
    })
}
```

### 测试组件说明

#### 1. PreCheck

测试前置检查：

```go
PreCheck: func() { acceptance.TestAccPreCheck(t) },
```

对于需要特殊环境的测试：

```go
PreCheck: func() {
    acceptance.TestAccPreCheck(t)
    acceptance.TestAccPreCheckEpsID(t)
},
```

#### 2. ProviderFactories

Provider 工厂配置：

```go
ProviderFactories: acceptance.TestAccProviderFactories,
```

#### 3. CheckDestroy

资源销毁验证：

```go
func testAccCheckVpcV1Destroy(s *terraform.State) error {
    config := acceptance.TestAccProvider.Meta().(*config.Config)
    vpcClient, err := config.NetworkingV1Client(acceptance.HW_REGION_NAME)
    if err != nil {
        return fmtp.Errorf("Error creating huaweicloud vpc client: %s", err)
    }

    for _, rs := range s.RootModule().Resources {
        if rs.Type != "huaweicloud_vpc" {
            continue
        }

        _, err := vpcs.Get(vpcClient, rs.Primary.ID).Extract()
        if err == nil {
            return fmtp.Errorf("Vpc still exists")
        }
    }

    return nil
}
```

#### 4. TestStep

测试步骤定义：

```go
Steps: []resource.TestStep{
    {
        Config: testAccVpcV1_basic(rName),
        Check: resource.ComposeTestCheckFunc(
            testAccCheckVpcV1Exists(resourceName, &vpc),
            resource.TestCheckResourceAttr(resourceName, "name", rName),
        ),
    },
},
```

### 资源存在性检查

```go
func testAccCheckVpcV1Exists(n string, vpc *vpcs.Vpc) resource.TestCheckFunc {
    return func(s *terraform.State) error {
        rs, ok := s.RootModule().Resources[n]
        if !ok {
            return fmtp.Errorf("Not found: %s", n)
        }

        if rs.Primary.ID == "" {
            return fmtp.Errorf("No ID is set")
        }

        config := acceptance.TestAccProvider.Meta().(*config.Config)
        vpcClient, err := config.NetworkingV1Client(acceptance.HW_REGION_NAME)
        if err != nil {
            return fmtp.Errorf("Error creating huaweicloud vpc client: %s", err)
        }

        found, err := vpcs.Get(vpcClient, rs.Primary.ID).Extract()
        if err != nil {
            return err
        }

        if found.ID != rs.Primary.ID {
            return fmtp.Errorf("vpc not found")
        }

        *vpc = *found
        return nil
    }
}
```

### Terraform 配置模板

```go
func testAccVpcV1_basic(rName string) string {
    return fmt.Sprintf(`
resource "huaweicloud_vpc" "test" {
  name        = "%s"
  cidr        = "192.168.0.0/16"
  description = "created by acc test"

  tags = {
    foo = "bar"
    key = "value"
  }
}
`, rName)
}
```

### 更新测试

```go
func TestAccVpcV1_basic(t *testing.T) {
    var vpc vpcs.Vpc

    rName := acceptance.RandomAccResourceName()
    rNameUpdate := rName + "_updated"
    resourceName := "huaweicloud_vpc.test"

    resource.ParallelTest(t, resource.TestCase{
        // ...
        Steps: []resource.TestStep{
            {
                Config: testAccVpcV1_basic(rName),
                Check: resource.ComposeTestCheckFunc(
                    testAccCheckVpcV1Exists(resourceName, &vpc),
                    resource.TestCheckResourceAttr(resourceName, "name", rName),
                ),
            },
            {
                Config: testAccVpcV1_update(rNameUpdate),
                Check: resource.ComposeTestCheckFunc(
                    testAccCheckVpcV1Exists(resourceName, &vpc),
                    resource.TestCheckResourceAttr(resourceName, "name", rNameUpdate),
                ),
            },
        },
    })
}

func testAccVpcV1_update(rName string) string {
    return fmt.Sprintf(`
resource "huaweicloud_vpc" "test" {
  name        = "%s"
  cidr        = "192.168.0.0/16"
  description = "updated by acc test"

  tags = {
    foo1 = "bar"
    key  = "value_updated"
  }
}
`, rName)
}
```

### 导入测试

```go
{
    ResourceName:      resourceName,
    ImportState:       true,
    ImportStateVerify: true,
},
```

忽略某些字段的导入验证：

```go
{
    ResourceName:            resourceName,
    ImportState:             true,
    ImportStateVerify:       true,
    ImportStateVerifyIgnore: []string{"secondary_cidr"},
},
```

## Data Source 测试

### 基本测试结构

```go
func TestAccVpcsDataSource_basic(t *testing.T) {
    randName := acceptance.RandomAccResourceName()
    randCidr := acceptance.RandomCidr()
    dataSourceName := "data.huaweicloud_vpcs.test"

    dc := acceptance.InitDataSourceCheck(dataSourceName)

    resource.ParallelTest(t, resource.TestCase{
        PreCheck:          func() { acceptance.TestAccPreCheck(t) },
        ProviderFactories: acceptance.TestAccProviderFactories,
        Steps: []resource.TestStep{
            {
                Config: testAccDataSourceVpcs_basic(randName, randCidr),
                Check: resource.ComposeTestCheckFunc(
                    dc.CheckResourceExists(),
                    resource.TestCheckResourceAttr(dataSourceName, "vpcs.0.cidr", randCidr),
                    resource.TestCheckResourceAttr(dataSourceName, "vpcs.0.name", randName),
                    acceptance.TestCheckResourceAttrWithVariable(dataSourceName, "vpcs.0.id",
                        "${huaweicloud_vpc.test.id}"),
                ),
            },
        },
    })
}
```

### Data Source 配置模板

```go
func testAccDataSourceVpcs_base(rName, cidr string) string {
    return fmt.Sprintf(`
resource "huaweicloud_vpc" "test" {
  name = "%s"
  cidr = "%s"
}
`, rName, cidr)
}

func testAccDataSourceVpcs_basic(rName, cidr string) string {
    return fmt.Sprintf(`
%s

data "huaweicloud_vpcs" "test" {
  id = huaweicloud_vpc.test.id
}
`, testAccDataSourceVpcs_base(rName, cidr))
}
```

### 标签过滤测试

```go
func TestAccVpcsDataSource_tags(t *testing.T) {
    randName1 := acceptance.RandomAccResourceName()
    randName2 := acceptance.RandomAccResourceName()
    randCidr := acceptance.RandomCidr()
    dataSourceName := "data.huaweicloud_vpcs.test"

    dc := acceptance.InitDataSourceCheck(dataSourceName)

    resource.ParallelTest(t, resource.TestCase{
        PreCheck:          func() { acceptance.TestAccPreCheck(t) },
        ProviderFactories: acceptance.TestAccProviderFactories,
        Steps: []resource.TestStep{
            {
                Config: testAccDataSourceVpcs_tags(randName1, randName2, randCidr),
                Check: resource.ComposeTestCheckFunc(
                    dc.CheckResourceExists(),
                    resource.TestCheckResourceAttr(dataSourceName, "tags.foo", randName1),
                    resource.TestCheckResourceAttr(dataSourceName, "vpcs.0.name", randName1),
                ),
            },
        },
    })
}

func testAccDataSourceVpcs_tags(rName1, rName2, randCidr string) string {
    return fmt.Sprintf(`
resource "huaweicloud_vpc" "test1" {
  name = "%[1]s"
  cidr = "%[3]s"
  tags = {
    foo = "%[1]s"
  }
}

resource "huaweicloud_vpc" "test2" {
  name = "%[2]s"
  cidr = "%[3]s"
  tags = {
    foo = "%[2]s"
  }
}

data "huaweicloud_vpcs" "test" {
  cidr = "%[3]s"
  tags = {
    foo = "%[1]s"
  }
  depends_on = [
    huaweicloud_vpc.test1,
    huaweicloud_vpc.test2,
  ]
}
`, rName1, rName2, randCidr)
}
```

## 测试工具函数

### 随机资源名称

```go
rName := acceptance.RandomAccResourceName()
// 输出: tf-acc-test-xxxxx
```

### 随机 CIDR

```go
randCidr := acceptance.RandomCidr()
// 输出: 192.168.x.0/24
```

### Data Source 检查器

```go
dc := acceptance.InitDataSourceCheck(dataSourceName)

// 使用检查器
dc.CheckResourceExists()
```

### 变量属性检查

```go
acceptance.TestCheckResourceAttrWithVariable(dataSourceName, "vpcs.0.id",
    "${huaweicloud_vpc.test.id}")
```

## 测试环境变量

### 必需环境变量

```bash
export HW_REGION_NAME="cn-north-4"
export HW_ACCESS_KEY="your-access-key"
export HW_SECRET_KEY="your-secret-key"
```

### 可选环境变量

```bash
export HW_DOMAIN_NAME="your-domain-name"
export HW_ENTERPRISE_PROJECT_ID_TEST="your-eps-id"
export HW_ENTERPRISE_MIGRATE_PROJECT_ID_TEST="your-migrate-eps-id"
```

### 特定测试环境变量

```go
func TestAccVpcV1_WithEpsId(t *testing.T) {
    // ...
    PreCheck: func() {
        acceptance.TestAccPreCheckEpsID(t)
        acceptance.TestAccPreCheckMigrateEpsID(t)
    },
    // ...
}
```

## 运行测试

### 运行单个测试

```bash
go test -v -run TestAccVpcV1_basic ./huaweicloud/services/acceptance/vpc/
```

### 运行所有测试

```bash
go test -v ./huaweicloud/services/acceptance/vpc/
```

### 运行并行测试

```bash
go test -v -parallel 4 ./huaweicloud/services/acceptance/vpc/
```

### 生成覆盖率报告

```bash
go test -cover ./huaweicloud/services/acceptance/vpc/
```

## 测试最佳实践

### 1. 使用并行测试

```go
resource.ParallelTest(t, resource.TestCase{
    // ...
})
```

### 2. 随机命名避免冲突

```go
rName := acceptance.RandomAccResourceName()
```

### 3. 完整的生命周期测试

```go
Steps: []resource.TestStep{
    // 创建测试
    {Config: testAccVpcV1_basic(rName), Check: ...},
    // 更新测试
    {Config: testAccVpcV1_update(rNameUpdate), Check: ...},
    // 导入测试
    {ResourceName: resourceName, ImportState: true, ImportStateVerify: true},
}
```

### 4. 验证所有属性

```go
Check: resource.ComposeTestCheckFunc(
    testAccCheckVpcV1Exists(resourceName, &vpc),
    resource.TestCheckResourceAttr(resourceName, "name", rName),
    resource.TestCheckResourceAttr(resourceName, "cidr", "192.168.0.0/16"),
    resource.TestCheckResourceAttr(resourceName, "description", "created by acc test"),
    resource.TestCheckResourceAttr(resourceName, "status", "OK"),
    resource.TestCheckResourceAttr(resourceName, "tags.foo", "bar"),
),
```

### 5. 测试错误场景

```go
{
    Config:      testAccVpcV1_invalidCidr(rName),
    ExpectError: regexp.MustCompile(`invalid CIDR`),
},
```

## 测试检查清单

### Resource 测试
- [ ] 基本创建测试
- [ ] 更新测试
- [ ] 导入测试
- [ ] 销毁验证
- [ ] 标签测试（如适用）
- [ ] 企业项目测试（如适用）

### Data Source 测试
- [ ] 基本查询测试
- [ ] 按名称查询
- [ ] 按 ID 查询
- [ ] 按标签查询
- [ ] 多条件组合查询

### 通用检查
- [ ] 使用随机资源名称
- [ ] 验证所有重要属性
- [ ] 清理测试资源
- [ ] 记录测试日志
