## Objective

In this section, we'll demonstrate templating a network configuration and pushing it a device. To do this, we'll employ a few new skills:

- Use and understand [host variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)
- Use simple [Jinja2 templating](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html)
- Demonstrate use of the network automation [junos_config module](https://docs.ansible.com/ansible/latest/modules/junos_config_module.html) for Juniper Junos and the [template module](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html) for Cumulus Linux.
- Use Ansible Network Automation to create an OSPF adjacency the Cumulus VX device vqfx1 and the Juniper Junos device vqfx1.

## Part 1 - Examining host_vars

For this exercise examine the folder named `host_vars`:

```
cd /antidote/stage3
ls host_vars/
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

You'll notice that this directory contains two files - `vqfx1`, and `cvx1`. Let's take a look at these files' contents:

```
cat host_vars/vqfx1
cat host_vars/cvx1
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

For each device there is one key-value pair.  The router-id for the Juniper device `vqfx1` is `10.10.10.1` and the router-id for `cvx1` is `10.10.10.2`.  We can store key-value pairs in `host_vars` and use them later in Jinja2 templates. If you need a refresher on what variables are, the previous section in this lesson should be helpful.

## Part 2 - Examining the frr.conf Jinja2 template

Cumulus devices use [Free-Range Routing]((https://frrouting.org/)) for their routing stack, which is configured using a file called `frr.conf`:

```
cat frr.conf
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

You'll notice the use of Jinja2 snippets in this file. If you're not familiar with Jinja2, there's a whole lesson focused on
this in another lesson. For now, lets examine the contents of this simple Jinja2 template in detail:

```
interface swp1
 ip ospf 1 area 0
 ip ospf network point-to-point
!
router ospf 1
 ospf router-id {{router_id}}
!
```

This is a configuration for the OSPF (Open Shortest Path First) routing protocol. There is one variable substitution taking place in this template for `{{router_id}}`.  The contents of `{{router_id}}` will be replaced with the value of `router_id` in `host_vars/cvx1` which is `10.10.10.2`.

## Part 3 - Template module for Linux

Variables and Jinja2 templates need to be combined by using a task in an Ansible Playbook. For Linux-based operating systems like Cumulus we can use the [template module](https://docs.ansible.com/ansible/latest/modules/template_module.html) to load and fill out the template and push it to a Linux device.  Look at this example:

```
- name: template out cumulus configuration for free range routing
  become: true
  template:
    src: frr.conf
    dest: /etc/frr/frr.conf
```

Here is a breakdown of each part of this task:
- `name:` - this is a description of what the task does
- `become: true` - this will make the task run at privileged mode (e.g. administrative or sudo) when it is required)
- `template:` - this task is using the template module
- `src: frr.conf` - this task will use the frr.conf Jinja2 template as the source
- `dest: /etc/frr/frr.conf` - this task will overwrite the file frr.conf in the relevant directory with the rendered template.

## Part 4 - Service module for Linux

Since Cumulus Network's VX is Linux, it can use the native Ansible [service module](https://docs.ansible.com/ansible/latest/modules/service_module.html) to start, restart, stop and reload services such as FRR.  This is the same module you would use on Fedora, Red Hat Enterprise Linux or any Linux operating system for controlling services.

Look at the following task:
```
- name: reload frr
  become: true
  service:
    name: frr
    state: reloaded
```

If we make a change to /etc/frr/frr.conf which is the file that holds the routing configurations for Cumulus Linux we need to reload frr for those changes to take affect.

## Part 5 - Examine the cumulus.yml Ansible Playbook

Examine the cumulus.yml Ansible Playbook that contains both the previous two tasks (using the template and service modules).

```
cat cumulus.yml
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

These tasks are not a holistic Ansible Playbook and can't be run by themselves.  Notice how there is not `hosts` or `tasks` parameter.  Another Ansible Playbook will load these tasks dynamically.

## Part 6 - Examine the junos.yml Ansible Playbook

Examine the junos.yml Ansible Playbook:

```
cat junos.yml
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

This file has one task using the [junos_config module](https://docs.ansible.com/ansible/latest/modules/junos_config_module.html).  The junos_config can work exactly like the template module above where it loads a separate Jinja2 template file, but since there is only two lines we will just template directly into the `lines` parameter for the task.

Just like with the Cumulus VX template the "{{router_id}}" will be replaced with the `router_id` from `host_vars/vqfx1` and place the id `10.10.10.1`.

## Part 7 - Examine the ospf.yml Ansible Playbook

In the previous sections there were two separate list of tasks created per network operating system.  One for Cumulus VX called `cumulus.yml` and one for Juniper Junos called `junos.yml`.  These two plays match identically to the `ansible_network_os` host variable found in inventory. This will allow us to easily write an agnostic Ansible Playbook to load the correct list of tasks based on the `ansible_network_os`.

Examine the ospf.yml Ansible Playbook:
```
cat ospf.yml
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

This Ansible Playbook has two important differences from previous Ansible Playbooks shown.  

- The first is that the `hosts` parameter is running on the group devices that includes both the Juniper Junos vqfx1 and the Cumulus VX cvx1.
- The second is that we use the [include module](https://docs.ansible.com/ansible/latest/modules/include_module.html) to dynamically load the correct list of tasks, keying off of the `ansible_network_os`.

```
    - name: include relevant network playbook
      include: "{{ansible_network_os}}.yml"
```

This task will load for `cumulus.yml` for cvx1 and `junos.yml` will load for vqfx1.  If you need a refresher where the `ansible_network_os` is set please look at the inventory again here:

```
cat /antidote/stage0/hosts
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

For a full list of `ansible_network_os` values and supported network operating systems please refer to the [Ansible documentation](https://docs.ansible.com/ansible/latest/network/user_guide/platform_index.html#settings-by-platform).

## Part 8 - Run the ospf.yml Ansible Playbook

Run the Ansible Playbook `ospf.yml` with `ansible-playbook`:

```
ansible-playbook ospf.yml
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('ansible', this)">Run this snippet</button>

When the Ansible Playbook runs successfully, OSPF will be configured between vqfx1 and cvx1.  Make sure you can ping between the devices or OSPF will not come up.

```
ping 10.10.10.2 count 5
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('vqfx1', this)">Run this snippet</button>

## Part 9 - Verify OSPF is up and there is an adjacency

Now that you have automated OSPF configuration for the two devices you can verify that OSPF is up and there is an adjacency.  The Juniper device will use the command `show ospf neighbor`

```
show ospf neighbor
```
<button type="button" class="btn btn-primary btn-sm" onclick="runSnippetInTab('vqfx1', this)">Run this snippet</button>


## Complete

You have completed Part 3!

## Takeaways

- [Jinja2](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html) templates allow us to easily template out a device configuration.
- The `os_config` (e.g. [junos_config](https://docs.ansible.com/ansible/latest/modules/junos_config_module.html)) and `cli_config` modules can source a jinja2 template file, and push directly to a device. If you want to just render a configuration locally on the control node, or template to a Linux device, use the [template module](https://docs.ansible.com/ansible/latest/modules/template_module.html).
- Variables are mostly commonly stored in `group_vars` and `host_vars`. This short example only used `host_vars`.

---

These exercises are made possible by [Juniper Networks](https://juniper.net) and the [Red Hat Ansible Automation Platform](https://www.ansible.com/products/automation-platform)

<img src="https://github.com/nre-learning/nrelabs-curriculum/blob/v1.2.0/lessons/ansible-network-automation/rh-ansible-platform.png?raw=true"></div>

Check out our free network automation e-books on https://ansible.com:
- [Part 1: Modernize Your Network with Red Hat](https://www.ansible.com/resources/ebooks/network-automation-for-everyone)
- [Part 2: Automate Your Network with Red Hat](https://www.ansible.com/resources/ebooks/automate-your-network)
