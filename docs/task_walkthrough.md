# The linking task, line by line

This walks through the one task in `tasks/main.yml` that does all the
work of this role — how it connects to inventory, and how `item` turns
into an actual Zabbix API call. Traced and verified against a live
inventory run, not just read off the page.

```yaml
- name: Link each monitored host to its cluster templates
  community.zabbix.zabbix_host:
    host_name: "{{ item }}"
    host_groups:
      - "{{ zabbix_host_group }}"
    link_templates: "{{ hostvars[item].zabbix_agent_clusters | map('extract', zabbix_cluster_template_names) | list }}"
    status: enabled
    state: present
    interfaces:
      - type: 1
        main: 1
        useip: 1
        ip: "{{ hostvars[item].ansible_host }}"
        dns: ""
        port: "10050"
  loop: "{{ groups[zabbix_link_target_group] }}"
  loop_control:
    label: "{{ item }}"
```

## Line 18 — `loop: "{{ groups[zabbix_link_target_group] }}"`

Runs first, conceptually — it decides *how many times* the block above
executes, and what `item` equals each time. `groups` is Ansible's
built-in map of group name → list of inventory hostnames.
`zabbix_link_target_group` defaults to `zabbix_agents`. Checked against
the live inventory rather than assumed:

```
groups.zabbix_agents:
  - rhel9-ansible
  - rhel10-ansible
  - rhel9-hardened1   <- only declared under hypervisors: in hosts.yml
```

`rhel9-hardened1` isn't listed directly under `zabbix_agents:` in
`hosts.yml` — only under its child group `hypervisors:` — yet it still
shows up. A child group's hosts count as members of the parent
automatically, so this one group name sweeps in `hypervisors` and
`containers` too without listing them separately. So this task body
runs 3 times: once each with `item = "rhel9-ansible"`,
`"rhel10-ansible"`, `"rhel9-hardened1"`.

## Line 20 — `label: "{{ item }}"`

Cosmetic only — makes the Ansible output print `rhel9-hardened1` per
iteration instead of dumping the whole task dict. No effect on
behavior.

## The part that actually connects "inventory" to "linking": `hostvars[item]`

This whole play targets `zabbix_api_endpoint` (`deploy_zabbix.yml`
Phase 2) — Ansible is connected to *that one thing* over `httpapi`. It
is **never connected to** `rhel9-hardened1` or the others in this task.
`item` is just a plain string used as a **lookup key**. `hostvars` is
Ansible's dictionary of every host's variables, indexed by hostname,
built once at inventory-load time from `hosts.yml` + `group_vars`. So
`hostvars["rhel9-hardened1"]` pulls that host's variables purely as
*data* — no network call to it. That's how a task running against the
Zabbix server can still read what cluster a totally different,
SSH-only host belongs to.

## Line 5 — `host_name: "{{ item }}"`

The Zabbix host object gets named literally after the Ansible
inventory hostname. `"rhel9-hardened1"` in `hosts.yml` becomes the host
name shown in the Zabbix GUI too — one name, not two things to keep in
sync.

## Lines 6-7 — `host_groups: ["{{ zabbix_host_group }}"]`

Same static value every iteration (`"Linux servers"`). Every host lands
in the same Zabbix host group regardless of which iteration this is.

## Line 8 — the template-linking line, traced with real values

```
hostvars[item].zabbix_agent_clusters | map('extract', zabbix_cluster_template_names) | list
```

Take `item = "rhel9-hardened1"`. Trace its `zabbix_agent_clusters`
through `group_vars`:

- `inventory/group_vars/zabbix_agents.yml` sets the baseline:
  `zabbix_agent_clusters_baseline: [host_health, host_access,
  privileged_activity]`, and `zabbix_agent_clusters: "{{
  zabbix_agent_clusters_baseline }}"`.
- `rhel9-hardened1` sits in the `hypervisors` child group, so
  `inventory/group_vars/hypervisors.yml` overrides it:
  `zabbix_agent_clusters: "{{ zabbix_agent_clusters_baseline +
  ['virtualisation'] }}"`.

So `hostvars["rhel9-hardened1"].zabbix_agent_clusters` resolves to:

```
[host_health, host_access, privileged_activity, virtualisation]
```

`map('extract', zabbix_cluster_template_names)` runs next — `extract`
is a Jinja2 filter Ansible provides that treats its argument as a
lookup table: for every item in the list on the left, it does
`zabbix_cluster_template_names[item]` and returns the result. Using the
mapping from `vars/main.yml`:

```
host_health         -> "Scope Host Health and Availability"
host_access         -> "Scope Host Access"
privileged_activity  -> "Scope Privileged Activity"
virtualisation       -> "Scope Virtualisation KVM"
```

`map()` in Jinja2 is *lazy* — it produces a generator, not a real list
— so `| list` forces it to actually evaluate into a concrete list the
module can use. Final value sent to Zabbix for this host:

```yaml
link_templates:
  - "Scope Host Health and Availability"
  - "Scope Host Access"
  - "Scope Privileged Activity"
  - "Scope Virtualisation KVM"
```

**On a missing key — verified directly, not assumed:** if a cluster ID
in `zabbix_agent_clusters` has no matching entry in
`zabbix_cluster_template_names`, this does **not** fail silently. Ran a
minimal reproduction of `map('extract', dict)` with a key missing from
the dict, and it fails the task outright:

```
Error while resolving value for 'msg': object of type 'dict' has no attribute 'typo_key'
```

So this dict has to stay exactly in sync with whatever cluster IDs
`zabbix_agent_clusters` can produce — a mismatch is a hard failure on
the next run, not a quietly-missing template.

## Lines 9-10 — `status: enabled`, `state: present`

`state: present` is the create-or-update instruction — this is what
recreates a host object if it was ever deleted in the Zabbix GUI.
`status: enabled` means the host is actively polled, not sitting
disabled.

## Lines 11-17 — the `interfaces` block

Zabbix needs to know *how to reach the agent* to poll it — separate
from Ansible's own SSH connection info, even though it reuses the same
underlying value:

- `type: 1` — interface type 1 means "Zabbix agent" (Zabbix numbers its
  interface types: 1=agent, 2=SNMP, 3=IPMI, 4=JMX).
- `main: 1` — marks this the primary interface of that type.
- `useip: 1` — tells Zabbix to connect using the `ip` field below, not
  the `dns` field.
- `ip: "{{ hostvars[item].ansible_host }}"` — same trick as
  `host_name`: pulled straight from that host's own `ansible_host` in
  `hosts.yml` (e.g. `192.168.20.100` for `rhel9-hardened1`). One IP,
  defined once in inventory, reused for both "how Ansible reaches it
  over SSH" and "how Zabbix reaches it to poll" — not duplicated.
- `dns: ""` — empty on purpose, since `useip: 1` means this field is
  ignored anyway.
- `port: "10050"` — Zabbix agent's default listening port for passive
  checks.

## Putting the whole flow together, end to end, for one host

```
hosts.yml: rhel9-hardened1 under hypervisors, ansible_host=192.168.20.100
   |
group_vars/hypervisors.yml: zabbix_agent_clusters = baseline + [virtualisation]
   |
hostvars["rhel9-hardened1"].zabbix_agent_clusters =
   [host_health, host_access, privileged_activity, virtualisation]
   |
map('extract', zabbix_cluster_template_names) -> 4 real Zabbix template names
   |
community.zabbix.zabbix_host called against zabbix_api_endpoint:
   create/update host "rhel9-hardened1", IP 192.168.20.100, group "Linux servers",
   linked to those 4 templates
```

Nothing here connects to `rhel9-hardened1` itself — the whole task is:
read that host's *inventory data*, then tell the Zabbix API (which the
play *is* connected to) what to do with it.
