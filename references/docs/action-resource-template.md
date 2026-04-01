---
subcategory: "Workspace"
layout: "huaweicloud"
page_title: "HuaweiCloud: huaweicloud_workspace_desktop_user_batch_create"
description: |-
  Use this resource to batch create users for desktops within HuaweiCloud.
---

# huaweicloud_workspace_desktop_user_batch_create

Use this resource to batch create users for desktops within HuaweiCloud.

-> This resource is a one-time action resource for batch creating users for a desktop. Deleting this resource will
   not clear the corresponding request record, but will only remove the resource information from the tfstate file.

## Example Usage

### Create one user for desktop

```hcl
variable "desktop_id" {}
variable "user_name" {}

resource "huaweicloud_workspace_desktop_user_batch_create" "test" {
  desktop_id = var.desktop_id
  user_name  = var.user_name
}
```

### Batch create users for desktop

```hcl
variable "desktop_id" {}
variable "create_user_information" {
  type = list(object({
    user_name  = string
    user_group = string
  }))
}

resource "huaweicloud_workspace_desktop_user_batch_create" "test" {
  desktop_id = var.desktop_id
    
  dynamic "create_user_infos" {
      for_each = var.create_user_information
    
      content {
        user_name  = create_user_infos.value.user_name
        user_group = create_user_infos.value.user_group 
      }
  }
}
```

## Argument

The following arguments are supported:

* `region` - (Optional, String, ForceNew) Specifies the region where the desktops are located.  
  If omitted, the provider-level region will be used. This parameter is non-updatable.

* `desktop_id` - (Optional, String, NonUpdatable) Specifies the ID of the desktop to be assigned.

* `user_name` - (Optional, String, NonUpdatable) Specifies the user to whom the desktop belongs.  

* `number` - (Optional, Int) Specifies the number of the subnet.

* `ipv6_enable` - (Optional, Bool) Whether the subnet is IPv6 enabled. Defaults to **false**.
  The valid values are as follows:
  + **true**
  + **false**

* `source` - (Required, String) Specifies the source configuration of the component, in JSON format.  
  For the keys, please refer to the [documentation](https://support.huaweicloud.com/intl/en-us/api-servicestage/servicestage_06_0076.html#servicestage_06_0076__en-us_topic_0220056058_ref28944532).

* `ip_address` - (Optional, List) Specifies the IP address of the subnet.

* `create_user_infos` - (Required, List, NonUpdatable) Specifies the list of user information to be created.  
  The [create_user_infos](#desktop_user_batch_attach_create_user_infos) structure is documented below.

<a name="desktop_user_batch_attach_create_user_infos"></a>
The `create_user_infos` block supports:

* `user_name` - (Optional, String, NonUpdatable) Specifies the name of the desktop assignment object.  

* `user_group` - (Optional, String, NonUpdatable) Specifies the user group to which the desktop user belongs.  
  The valid values are as follows:
  + **sudo**: Linux administrator group.
  + **default**: Linux default user group.
  + **administrators**: Windows administrator group. Administrators have full access to the desktop and can make any
    changes needed (except disable operations).
  + **users**: Windows standard user group. Standard users can use most software and can change system settings that
    do not affect other users.

## Attributes

In addition to all arguments above, the following attributes are exported:

* `id` - The resource ID.
