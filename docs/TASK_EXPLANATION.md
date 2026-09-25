# zabbix_host_link — the task, block by block

Plain-language walkthrough of this role's single task: what each line
does, why it is there, and whether it is really needed. Paths are
relative to this role's folder. For a deeper trace of one host through
the loop with real values, see `docs/task_walkthrough.md`.

## What this role is for (one paragraph)

It runs **against the Zabbix server's API** (over `httpapi`) and, for
every monitored host, creates (or updates) that host's **Host object**
in Zabbix and **links it to the right templates**. The other two roles
each do half a job: `zabbix_agent_deploy` prepares the machine, and
`zabbix_template_deploy` puts the template recipes on the server.
Neither says "poll *this* machine using *that* recipe." This role is
the only place that does — without it you'd have a working agent and a
working template producing zero data.

## The variables it uses, and where they come from

| Variable | Set in | Meaning |
| --- | --- | --- |
| `zabbix_link_target_group` | `defaults/main.yml` = `zabbix_agents` | **Ansible** inventory group to loop over |
| `zabbix_host_group` | `defaults/main.yml` = `Linux servers` | **Zabbix** host group each host is filed under |
| `zabbix_cluster_template_names` | `vars/main.yml` | Fixed map: cluster ID → Zabbix template name |
| `zabbix_agent_clusters` | the inventory (`group_vars/`), per host | Which clusters this host belongs to |
| `ansible_host` | the inventory (`hosts.yml`), per host | The host's IP |

Why the split between `defaults/` and `vars/`: the first two are real
per-deployment choices, so they stay easy to override. The mapping is
a fixed fact tied to the template files' internal names, so it sits in
`vars/` where inventory can't accidentally shadow it.

---

## `tasks/main.yml` — the whole role

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

### `loop: "{{ groups[zabbix_link_target_group] }}"`
- `groups` is Ansible's built-in map of group name → list of hostnames.
  With the default `zabbix_agents` it resolves to every monitored host,
  **including hosts only declared in child groups** such as
  `hypervisors` (a child's members count as members of the parent).
- The task body then runs once per host; `item` is that host's name.
- **Why:** there's no hard-coded host list anywhere in the role. Add a
  host to the inventory and the next run picks it up.

### `loop_control: label: "{{ item }}"`
Cosmetic: prints the host name per iteration instead of the whole task
dictionary.

### `community.zabbix.zabbix_host:`
The module that creates/updates a Host object through the Zabbix API.
The play is connected to the Zabbix server only — it never connects to
the monitored hosts here.

### `host_name: "{{ item }}"`
Names the Zabbix host exactly like the inventory hostname.
**This is load-bearing:** `zabbix_agent_deploy` writes the same
`inventory_hostname` into the agent's `Hostname=` line. Server and
agent agree on the host's identity because both come from one source.

### `host_groups: ["{{ zabbix_host_group }}"]`
Zabbix requires every host to belong to at least one *Zabbix* host
group (unrelated to Ansible's inventory groups). Every host goes to
`Linux servers`. It doesn't change what gets monitored — only how the
host is organized and permissioned in the Zabbix GUI. Check it under
*Data collection → Host groups*.

### `link_templates: "{{ hostvars[item].zabbix_agent_clusters | map('extract', zabbix_cluster_template_names) | list }}"`
The core line. Read it left to right:

1. `hostvars[item]` — Ansible's table of every host's variables, looked
   up by name. This lets a task running against the Zabbix server read
   another host's data straight from the inventory, with no connection
   to that host.
2. `.zabbix_agent_clusters` — that host's cluster list, e.g.
   `[host_health, host_access, privileged_activity]`. The same list
   `zabbix_agent_deploy` used to decide which access to grant.
3. `map('extract', zabbix_cluster_template_names)` — for each cluster
   ID, look it up in the mapping and return the Zabbix template name.
4. `| list` — turn the lazy result into a real list the module accepts.

Result for a plain agent host:

```yaml
link_templates:
  - "Scope Host Health and Availability"
  - "Scope Host Access"
  - "Scope Privileged Activity"
```

- **Why this design:** a host's whole monitoring scope is defined once,
  in the inventory. Changing `zabbix_agent_clusters` changes both the
  access granted on the host **and** the templates linked, with no
  second list to keep in sync.
- **Sharp edge:** a cluster ID with no entry in
  `zabbix_cluster_template_names` makes the task **fail loudly** (tested:
  `object of type 'dict' has no attribute '<id>'`). It does not quietly
  skip.
- **Sharp edge:** `link_templates` **replaces** the host's entire
  template list every run. Anything linked by hand in the GUI is
  removed on the next run — intentional, so this role stays the single
  source of truth.

### `status: enabled`
The host is actively monitored (not created in a disabled state).

### `state: present`
"Create it if missing, update it if it exists." This is what rebuilds a
host you deleted in the GUI, and what makes re-runs safe.

### `interfaces:` block
Tells Zabbix *how to reach the agent* (separate from Ansible's SSH
info):

| Line | Meaning |
| --- | --- |
| `type: 1` | Interface type "Zabbix agent" (2=SNMP, 3=IPMI, 4=JMX) |
| `main: 1` | The primary interface of that type |
| `useip: 1` | Connect by the IP field, not DNS |
| `ip: "{{ hostvars[item].ansible_host }}"` | The IP straight from the inventory — the same value Ansible uses to SSH in, so the two can't drift |
| `dns: ""` | Blank on purpose; ignored when `useip: 1` |
| `port: "10050"` | The agent's standard port, the same one `zabbix_agent_deploy` opens in firewalld |

---

## Order requirement

This role must run **after** `zabbix_template_deploy`, in the same
execution. It links by template *name*; a name that isn't on the server
yet can't be linked. (And because the template import unlinks
everything, this role also restores the links straight after.)
`playbooks/deploy_zabbix.yml` enforces that order.

## Requirements

- `community.zabbix` collection on the control node.
- An `httpapi` connection to the Zabbix server with credentials that
  may create/update hosts.
- Every host in the target group must already have `ansible_host` and
  `zabbix_agent_clusters` available from the inventory — this role
  reads them, it doesn't define them.

## Quick "is it really needed?" summary

| Piece | Needed? | Why |
| --- | --- | --- |
| The loop over `groups[...]` | Yes | Decides which hosts get processed, with no hard-coded list |
| `host_name` | Yes | Must equal the agent's `Hostname=` |
| `host_groups` | Yes (Zabbix requires one) | But has no effect on what is collected |
| `link_templates` expression | Yes | The actual link between a host and its templates |
| `interfaces` | Yes | Zabbix needs an address and port to poll |
| `state: present` / `status: enabled` | Yes | Create-or-update, and actually monitor it |
| `loop_control` label | No (cosmetic) | Readable output |
