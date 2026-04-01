---
subcategory: "Virtual Private Cloud (VPC)"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_vpc_subnets"
description: |-
  Use this data source to query the VPC subnets under a specified region within HuaweiCloud.
---

# huaweicloud_vpc_subnets

Use this data source to query the VPC subnets under a specified region within HuaweiCloud.

## Example Usage

### Basic Usage
```hcl
variable "vpc_id" {}

data "huaweicloud_vpc_subnets" "test" {
  vpc_id = var.vpc_id
}
```

### Filter Subnets by status
```hcl
variable "vpc_id" {}

data "huaweicloud_vpc_subnets" "test" {
  vpc_id = var.vpc_id
  status = "PENDING"
}
```

## Argument Reference

The following arguments are supported:

* `region` - (Optional, String) Specifies the region in which to query the subnets. If omitted, the provider-level region will be used.

* `vpc_id` - (Required, String) Specifies the ID of the VPC to filter subnets by.

* `number` - (Optional, Int) Specifies the number of the subnet.

* `status` - (Optional, String) Specifies the status of the subnet.
  The valid values are as follows:
  + **ACTIVE**: The subnet is active.
  + **FAILED**: The subnet is failed.
  + **PENDING**: The subnet is pending.

* `path` - (Optional, Int) Specifies the path of the probe.  
  This parameter is only available when the `status` is set to **valid**.

* `ipv6_enable` - (Optional, Bool) Whether the subnet is IPv6 enabled. Defaults to **false**.
  The valid values are as follows:
  + **true**
  + **false**

* `source` - (Required, String) Specifies the source configuration of the component, in JSON format.  
  For the keys, please refer to the [documentation](https://support.huaweicloud.com/intl/en-us/api-servicestage/servicestage_06_0076.html#servicestage_06_0076__en-us_topic_0220056058_ref28944532).

* `ip_address` - (Optional, List) Specifies the IP address of the subnet.

* `ports` - (Optional, List) Specifies the ports of the subnet.  
  The [ports](#subnets_ports_arg) structure is documented below.

* `route` - (Optional, List) Specifies the routes of the subnet.  
  The [route](#subnets_route) structure is documented below.

* `tags` - (Optional, Map) Specifies the tags of the subnet.

<a name="subnets_ports_arg"></a>
The `ports` block supports:

* `name` - The name of the port.

<a name="subnets_route"></a>
The `route` block supports:

* `id` - The ID of the route.

* `destination` - The destination of the route.

## Attribute Reference

In addition to all arguments above, the following attributes are exported:

* `id` - The data source ID.

* `created_at` - The time when the subnet was created.

* `updated_at` - The time when the subnet was updated.

* `english_name` - The english name of the subnet.

  ->**Note:** The value of the `english_name` maybe empty in some regions.

* `chinese_name` - The chinese name of the subnet.

* `other_name` - The other name of the subnet.

~>**WARNING:** The `chinese_name` and `other_name` only returns in **630** and later version.

* `ports` - The ports of the subnet.  
  The [ports](#subnets_ports_attr) structure is documented below.

<a name="subnets_ports_attr"></a>
The `ports` block supports:

* `id` - The ID of the port.

* `description` - The description of the port.
