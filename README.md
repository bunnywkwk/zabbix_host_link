Zabbix Host Link
==================

Ensures each monitored host exists as a Host object on the Zabbix server,
and links it to the Zabbix template(s) matching its assigned requirement
clusters — over the Zabbix HTTP API.

This is the third and final piece of the pipeline, and it's deliberately
narrow: `zabbix_agent_deploy` prepares a host to *be* monitored (over
ssh), `zabbix_template_deploy` puts a template's *definition* on the
server (over httpapi), and this role does the one thing neither of those
does — tells the server "poll *this specific host* using *this specific
template*." Without it, a perfectly configured agent and a perfectly
imported template still produce zero data, because nothing on the server
associates the two.

Purpose & Approach
-------------------

For every host in `zabbix_link_target_group`, this role calls
`community.zabbix.zabbix_host` to create-or-update that host's Zabbix
object and set its linked templates. It reads everything it needs
straight from that inventory's `hostvars` — the host's IP
(`ansible_host`) and its cluster list (`zabbix_agent_clusters`) — rather
than keeping a second, separate copy of either. The same
`zabbix_agent_clusters` list that already decided which sudoers/ACL/
UserParameter access `zabbix_agent_deploy` granted on the host is what
decides which templates get linked here, translated from cluster IDs
(`host_health`, `storage`, ...) to Zabbix's actual template names via
`zabbix_cluster_template_names`. A host's full monitoring scope is
therefore defined in exactly one place: its inventory group.

Because it links to templates by name, **this role must run after
`zabbix_template_deploy`** in any playbook that uses both — a host can't
be linked to a template that doesn't exist on the server yet.

Requirements
------------

- The `community.zabbix` collection installed on the control node (see
  `meta/main.yml`'s `dependencies:` comment — tracked at the playbook
  level, not bundled with this role).
- An `httpapi` connection to the target Zabbix server, and valid Zabbix
  API credentials — same requirements as `zabbix_template_deploy`.
- Every host in `zabbix_link_target_group` must already have
  `ansible_host` and `zabbix_agent_clusters` set in inventory — this role
  reads both directly out of `hostvars`, it doesn't define either.

Role Variables
---------------

Defined in `defaults/main.yml`:

| Variable                      | Default          | Purpose                                                                                   |
| ------------------------------- | ------------------ | --------------------------------------------------------------------------------------------- |
| `zabbix_link_target_group`     | `zabbix_agents`   | Inventory group to loop over. Its `hypervisors`/`container_hosts` children are included automatically (Ansible counts a child group's hosts as members of the parent too). |
| `zabbix_host_group`            | `Linux servers`   | The Zabbix host group every linked host is placed into. Unrelated to each template's own `template_groups` field, which varies per template (Linux servers / Scope Templates / Hypervisors) — that's UI categorization for templates, not for hosts. |
| `zabbix_cluster_template_names` | (6-entry mapping) | Translates a cluster ID (as used in `zabbix_agent_clusters`) to the exact `template:` name declared in its `scope_*.yaml`. |

Dependencies
------------

None (no other Ansible roles). The `community.zabbix` collection is a
hard requirement but isn't a role dependency — see Requirements above.

Example Playbook
------------------

    - hosts: zabbix_api_endpoint
      gather_facts: false
      roles:
        - zabbix_template_deploy   # must come first - templates need to exist
        - zabbix_host_link

Known Limitations
-------------------

- `link_templates` **replaces** a host's full template list on every run,
  it isn't additive. Any template linked by hand later through the
  Zabbix GUI gets removed on the next run — intentional for keeping this
  role as the single source of truth, but worth knowing before it
  surprises you.
- A host present in `zabbix_link_target_group` with no
  `zabbix_agent_clusters` set (or an empty list) is still created on the
  server with `state: present` and `host_groups` set — it's just linked
  to zero templates, not skipped entirely.
- `zabbix_host_group` is a single fixed value for every host regardless
  of cluster. Fine while every monitored host belongs to one Zabbix host
  group; would need to become per-host if that ever needs to vary.

License
-------

MIT-0

Author Information
--------------------

Part of the `zabbix-roles` project — see `zabbix_agent_deploy/README.md`
for the project-wide purpose and the source documentation this role's
template names are drawn from.
