---
page_title: "panos: panos_panorama_pbf_rule_group"
subcategory: "Panorama Policy"
---

# panos_panorama_pbf_rule_group

This resource allows you to add/update/delete Panorama policy based forwarding rule groups.

This resource manages clusters of policy based forwarding rules in a single vsys,
enforcing both the contents of individual rules as well as their
ordering.  Rules are defined in a `rule` config block.

Although you cannot modify non-group PBF rules with this
resource, the `position_keyword` and `position_reference` parameters allow you
to reference some other PBF rule that already exists, using it as
a means to ensure some rough placement within the ruleset as a whole.


## Best Practices

As is to be expected, if you are separating your deployment across
multiple plan files, make sure that at most only one plan specifies any given
absolute positioning keyword such as "top" or "directly below", otherwise
they'll keep shoving each other out of the way indefinitely.


## Example Usage

```hcl
# NOTE:  The "deny everything else" rule will need to already exist or
#  this will fail.
resource "panos_panorama_pbf_rule_group" "example" {
    device_group = panos_panorama_device_group.grp.name
    position_keyword = "above"
    position_reference = "deny everything else"
    rule {
        name = "my-pbf"
        description = "deployed by terraform"
        source {
            zones = ["zone1"]
            addresses = ["10.50.50.50"]
            users = ["any"]
            negate = true
        }
        destination {
            addresses = ["10.80.80.80"]
            applications = ["any"]
            services = ["application-default"]
        }
        forwarding {
            action = "discard"
        }
    }
}

resource "panos_panorama_device_group" "grp" {
    name = "myDeviceGroup"
}
```

## Argument Reference

The following arguments are supported:

* `device_group` - (Optional) The device group to put the rules into
  (default: `shared`).
* `rulebase` - (Optional) The rulebase.  This can be `pre-rulebase` (default),
  `post-rulebase`, or `rulebase`.
* `position_keyword` - (Optional) A positioning keyword for this group.  This
  can be `before`, `directly before`, `after`, `directly after`, `top`,
  `bottom`, or left empty (the default) to have no particular placement.  This
  param works in combination with the `position_reference` param.
* `position_reference` - (Optional) Required if `position_keyword` is one of the
  "above" or "below" variants, this is the name of a non-group rule to use
  as a reference to place this group.
* `rule` - The rule definition (see below).  The rule
  ordering will match how they appear in the terraform plan file.

The following arguments are valid for each `rule` section:

* `name` - (Required) The rule name.
* `description` - (Optional) The rule description.
* `tags` - (Optional) List of tags for this rule.
* `active_active_device_binding` - (Optional) The active-active device binding.
* `schedule` - (Optional) The schedule.
* `disabled` - (Optional, bool) Set to `true` to disable this rule.
* `uuid` - (Optional, computed, PAN-OS 9.0+) The UUID for the rule.
* `source` - (Required) The source spec (defined below).
* `destination` - (Required) The destination spec (defined below).
* `forwarding` - (Required) The forwarding spec (defined below).
* `target` - (Optional) A target definition (see below).  If there are no
  target sections, then the rule will apply to every vsys of every device
  in the device group.
* `negate_target` - (Optional, bool) Instead of applying the rule for the
  given serial numbers, apply it to everything except them.

`rule.source` supports the following arguments:

* `zones` - (Optional) If you want a source type of "zone", then define this
  list with the desired source zones.  Mutually exclusive with `rule.interfaces`.
* `interfaces` - (Optional) If you want a source type of "interface", then define this
  list with the desired source interfaces.  Mutually exclusive with `rule.zones`.
* `addresses` - (Required) List of source IP addresses.
* `users` - (Required) List of source users.
* `negate` - (Optional, bool) Set to `true` to negate the source.

`rule.destination` supports the following arguments:

* `addresses` - (Required) The list of destination addresses.
* `application` - (Required) The list of applications.
* `services` - (Required) The list of services.
* `negate` - (Optional, bool) Set to `true` to negate the destination.

`rule.forwarding` supports the following arguments:

* `action` - (Optional) The action to take.  Valid values are `forward` (default),
  `forward-to-vsys`, `discard`, or `no-pbf`.
* `vsys` - (Optional) If `action=forward-to-vsys`, the vsys to forward to.
* `egress_interface` - (Optional) If `action=forward`, the egress interface.
* `next_hop_type` - (Optional) If `action=forward`, the next hop type.  Valid values
  are `ip-address`, `fqdn`, or leaving this empty for a next hop type of None.
* `next_hop_value` - (Optional) If `action=forward` and `next_hop_type` is defined, then
  the next hop address.
* `monitor` - (Optional) The monitor spec (defined below).  If you do not want to enable
  monitoring, then do not specify a `monitor` config block.
* `symmetric_return` - (Optional) The symmetric return spec (defined below).  If you do
  not want to enforce symmetric

`rule.forwarding.monitor` supports the following arguments:

* `profile` - (Optional) The montior profile to use.
* `ip_address` - (Optional) The monitor IP address.
* `disable_if_unreachable` - (Optional, bool) Set to `true` to disable this rule if
  nexthop/monitor IP is unreachable.

`rule.forwarding.symmetric_return` supports the following arguments:

* `enable` - (Optional, bool) Set to `true` to enforce symmetric return.
* `addresses` - (Optional) List of next hop addresses.

`rule.target` supports the following arguments:

* `serial` - (Required) The serial number of the firewall.
* `vsys_list` - (Optional) A subset of all available vsys on the firewall
  that should be in this device group.  If the firewall is a virtual firewall,
  then this parameter should just be omitted.
