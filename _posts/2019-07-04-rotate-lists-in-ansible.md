---
title: "Rotate Lists in Ansible"
excerpt_separator: <!--more-->
tags:
  - ansible
  - jinja2
  - etcd
  - kubernetes
  - k8s
---

Overview
---

This is quick showcase for rotating lists in Ansible.

The technique described below was used to implement [an elegant work-around](https://github.com/kubernetes/kubernetes/issues/72102#issuecomment-478834647) to [kubernetess issue #72102](https://github.com/kubernetes/kubernetes/issues/72102), which comprised generating lists of etcd endpoints in alternating order for each k8s master.
<!--more-->

As we manage k8s configuration with [Ansible playbooks](https://docs.ansible.com/ansible/2.7/user_guide/playbooks.html), the solution was implemented in Ansible. This added a requirement of producing the same order for each instance across multiple executions.

Rotating the list fits nicely for the case: master hosts form a group in Ansible inventory, so do etcd peers, both can be considered ordered collections as such. We want to rotate list of etcd peers and we define offset based on the position of a given master in the list of master hosts.

Starting point
---

We start with the code where three masters receive identical sequences of endpoints.

We have inventory defined as follows:
```ini
[master_nodes]
master01
master02
master03

[etcd_nodes]
etcd01
etcd02
etcd03
etcd04
etcd05
```

The playbook applied to all master nodes:
```yaml
- name: Roll out k8s master
  hosts: master_nodes

  tasks:
  - name: Install k8s master configuration
    template:
      src: templates/master_config.j2
      dest: /etc/kubernetes/master_config
    notify: restart master
```

And the jinja2 template with k8s master configuration:
```text
{% raw %}{% set endpoints = groups['etcd_nodes'] %} }}{% endraw %}
--endpoints=
{% raw %}{% for endpoint in endpoints %}{% endraw %}
{% raw %}{{ endpoint }}{% endraw %}
{% raw %}{{- "," if not loop.last else "" -}}{% endraw %}
{% raw %}{% endfor %}{% endraw %}
```

When executed this produces the same configuration for each master:
```ini
--endpoints=etcd01,etcd02,etcd03,etcd04,etcd05
```

Deliver a differently ordered list to each master
---

As Ansible uses jinja2 templating we employ jinja2 [filters](https://docs.ansible.com/ansible/2.7/user_guide/playbooks_filters.html) functionality. As the desired filter is not shipped with Ansible, we'll write a custom one and use it in our Ansible code.

Python code that does the job of rotating a list.

```python
def rotate_right(l, shifts_nr):
    return l[shifts_nr:] + l[:shifts_nr]
```

In Ansible environment this code [should be placed](https://docs.ansible.com/ansible/2.7/user_guide/playbooks_best_practices.html?highlight=layout#content-organization) into `/filter_plugins` directory. Some boilerplate code to make Ansible aware of the filter should be provided as well.

This is how the complete filter code looks like:
```python
class FilterModule(object):
    """
    A jijna2 filter to right-rotate a list of values by a given number of positions.
    """

    def filters(self):
        return {
            'rotate_right': rotate_right
        }


def rotate_right(l, shifts_nr):
    return l[shifts_nr:] + l[:shifts_nr]
```

This is how the code is now organised:
```
├── filter_plugins
│   └── rotate_list.py
├── inventory
├── playbook.yaml
└── templates
    └── master_config.j2
```

With the filter in place we can now use it in our configuration template:
```text
{% raw %}{% set endpoints = groups['etcd_nodes'] | rotate_right(groups['master_nodes'].index(inventory_hostname)) %}{% endraw %}
--endpoints=
{% raw %}{%- for endpoint in endpoints %}{% endraw %}
{% raw %}{{ endpoint }}{% endraw %}
{% raw %}{{- "," if not loop.last else "" -}}{% endraw %}
{% raw %}{% endfor %}{% endraw %}
```

The only change to the initial template code is that now the list of etcd endpoints is fed to our filter for rotation: `groups['etcd_nodes'] | rotate_right(n)` where `n` is dynamic and function to the master host currently being served. `groups['master_nodes'].index(inventory_hostname)` is the position of the current host in the group, suitable to serve as an offset.

When this is executed each master gets different configuration, just as desired:
```ini
config for master01: --endpoints=etcd01,etcd02,etcd03,etcd04,etcd05

config for master02: --endpoints=etcd02,etcd03,etcd04,etcd05,etcd01

config for master03: --endpoints=etcd03,etcd04,etcd05,etcd01,etcd02
```

A complete and executable example can be found here: https://github.com/patrungel/supportingcode/tree/master/ansible-rotate-list

EPLS.

In Closing
---

Ansible version: 2.7

Article that helped me to get started with the above: [Enhance your Ansible playbooks with custom Jinja2 filters by Jordan Bach](https://opensolitude.com/2016/05/21/ansible-jinja2-filter-plugins.html).

Remark: in fact the order of endpoints could be of any kind, the only requirement being to have a different first element. This means that swapping first and n-th elements of the list would also work. Implementing this filter is left as an exercise for the reader :).
