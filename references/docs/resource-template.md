---
subcategory: "Virtual Private Cloud (VPC)"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_vpc_subnet"
description: |-
  Manages a VPC subnet resource within HuaweiCloud.
---

# huaweicloud_vpc_subnet

Manages a VPC subnet resource within HuaweiCloud.

## Example Usage

### Basic Usage
```hcl
variable "vpc_id" {}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id = var.vpc_id
}
```

### Filter Subnet by status
```hcl
variable "vpc_id" {}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id      = var.vpc_id
  number      = 10
  ipv6_enable = false
}
```

## Argument Reference

The following arguments are supported:

* `region` - (Optional, String, ForceNew) Specifies the region in which to create the subnet. If omitted, the
  provider-level region will be used. Changing this parameter will create a new resource.

* `vpc_id` - (Required, String, ForceNew) Specifies the ID of the VPC to create the subnet in.  
  Changing this parameter will create a new resource.

* `path` - (Optional, String, ForceNew) Specifies the path of the probe.  
  Changing this parameter will create a new resource.

* `number` - (Optional, Int) Specifies the number of the subnet.

* `password` - (Optional, String) Specifies the password of the subnet.

* `ipv6_enable` - (Optional, Bool) Whether enable IPv6 for the subnet. Defaults to **false**.
  The valid values are as follows:
  + **true**
  + **false**

* `source` - (Required, String) Specifies the source configuration of the component, in JSON format.  
  For the keys, please refer to the [documentation](https://support.huaweicloud.com/intl/en-us/api-servicestage/servicestage_06_0076.html#servicestage_06_0076__en-us_topic_0220056058_ref28944532).

* `ip_address` - (Optional, List) Specifies the IP address of the subnet.

* `ports` - (Optional, List) Specifies the ports of the subnet.  
  The [ports](#subnet_ports_arg) structure is documented below.

* `route` - (Optional, List) Specifies the routes of the subnet.  
  The [route](#subnet_route) structure is documented below.

* `tags` - (Optional, Map) Specifies the tags of the subnet.

<a name="subnet_ports_arg"></a>
The `ports` block supports:

* `name` - The name of the port.

<a name="subnet_route"></a>
The `route` block supports:

* `id` - The ID of the route.

* `destination` - The destination of the route.

## Attribute Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The resource ID.

* `created_at` - The time when the subnet was created.

* `updated_at` - The time when the subnet was updated.

* `english_name` - The english name of the subnet.

  ->**Note:** The value of the `english_name` maybe empty in some regions.

* `chinese_name` - The chinese name of the subnet.

* `other_name` - The other name of the subnet.

~>**WARNING:** The `chinese_name` and `other_name` only returns in **630** and later version.

* `ports` - The ports of the subnet.  
  The [ports](#subnet_ports_attr) structure is documented below.

<a name="subnet_ports_attr"></a>
The `ports` block supports:

* `id` - The ID of the port.

* `description` - The description of the port.

## Import

Subnet can be imported using the `id`, e.g.

```bash
$ terraform import huaweicloud_vpc_subnet.test <id>
```

Note that the imported state may not be identical to your resource definition, due to some attributes missing from the
API response, security or some other reason. The missing attributes include: `password`.
It is generally recommended running `terraform plan` after importing the resource.
You can then decide if changes should be applied to the subnet, or the resource definition should be updated to
align with the subnet. Also you can ignore changes as below.

```hcl
resource "huaweicloud_vpc_subnet" "test" {
  ...

  lifecycle {
    ignore_changes = [
      password,
    ]
  }
}
```
