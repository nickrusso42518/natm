[![Build Status](
https://travis-ci.org/nickrusso42518/natm.svg?branch=master)](
https://travis-ci.org/nickrusso42518/natm)

[![published](
http://cs.co/codeex-badge)](
https://developer.cisco.com/codeexchange/github/repo/nickrusso42518/natm)

# Network Address Translation Manager (natm)
This playbook automatically manages 1:1 static NAT statements
within a specified VRF on Cisco IOS routers. This is a particularly
labor-intensive and error-prone task, and this playbook
ensures that the desired state of the NAT table is applied.
Non-VRF (global table) operations are also supported.

> Contact information:\
> Email:    njrusmc@gmail.com\
> Twitter:  @nickrusso42518

  * [Supported platforms](#supported-platforms)
  * [Variables](#variables)
  * [Templates](#templates)
  * [Handlers](#handlers)

## Supported platforms
Cisco IOS routers are supported today. Any router that is running NAT and
could benefit by having automated management of the NAT statements is a good
candidate.

Testing was conducted on the following platforms and versions:
  * Cisco CSR1000v, version 16.07.01a, running in AWS

Control machine information:
```
$ cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.4 (Maipo)

$ uname -a
Linux ip-10-125-0-100.ec2.internal 3.10.0-693.el7.x86_64 #1 SMP
  Thu Jul 6 19:56:57 EDT 2017 x86_64 x86_64 x86_64 GNU/Linux

$ ansible --version
ansible 2.6.2
  config file = /home/ec2-user/natm/ansible.cfg
  configured module search path = [u'/home/ec2-user/.ansible/plugins/modules',
    u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, May  3 2017, 07:55:04)
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-14)]
```

## Variables
These playbooks rely only on `host_vars` which are defined for each router
within a given NAT line. Each NAT line must identify a unique name,
the inside private (inside local in Cisco speak), and outside public
(inside global in Cisco speak). All three fields are required. The
playbook checks the names and IPv4 addresses for uniqueness as a single
typo could cause significant issues on the production network. If invalid
input (e.g., malformed IPv4 addresses) or duplicate input (e.g., same
inside local in two places), the task fails for a given host.

Additionally, each NAT line must identify the target state. There are
two choices for this field, `"present"` and `"absent"`, which are should
be self-evident. A sample `host_vars` file is shown below. The `vrf`
option can be set to `false` (without the quotes) or a string. When
false is used, the global table is assumed. When any non-empty string is
used, that string is the VRF name. An empty string yields an error.

For simplicity, the VRF is defined globally and applies to all NAT entries.
If there are multiple VRFs with multiple NATs per VRF, simply create
additional variables files and run the playbook again. Below is an example of
a `host_vars` file.

Two other minor variables are also defined. `save_when` is just a wrapper for
the `ios_command` option that determines when to save configuration changes.
`ci_test` is a true/false variable that specifies when a host should actually
log into routers and manage NAT statements (`false`) or should use mocked
data for local testing only (`true`). Mock data files can be found in
`tasks/mock_<hostname>.yml`.

```
---
vrf: "TEST"
ci_test: false
vrf: false
static_nats:
  - name: "TEST_1"
    state: "present"
    inside_private: "192.168.0.1"
    outside_public: "100.64.0.1"
  - name: "TEST_2"
    state: "absent"
    inside_private: "192.168.0.2"
    outside_public: "100.64.0.2"
  - name: "TEST_3"
    state: "present"
    inside_private: "192.168.0.3"
    outside_public: "100.64.0.3"
...
```

## Templates
The Jinja2 templates are heavily commented for readability and are used to
develop host specific configurations based on the previously defined variables.
 __The templates should not be changed at the operator level.__

One locally-scoped variable is used in the `nat.j2` template. `nat_cmd_prefix`
contains the `ip nat name (name)` string which simplifies addition and removal.
Ultimately, this keeps the template clean and readable.

The template contains a loop which iterates over the `static_nats` list. One
of two actions will occur:

  * If present and the NAT line does not already exist in the config, add it.
  * If absent and the NAT line already exists in the config, remove it.

Removal is slightly simpler since NAT rules are referenced by name only
for deletion. Relying on the build-in idempotent functionality of the
`ios_config` module could have reduced the playbook complexity a little,
however it is preferable to actually check the NAT state table rather than
only rely on the presence or absence of a configuration line. As such, this
playbook adds complexity but provides a more comprehensive and accurate
assessment of the current NAT state.

## Handlers
A second template is for logging. Since this playbook is idempotent, logging
changes is useful to identify what changed, when, and on which device. When
changes occur, the task that manages the NAT configurations shows "changed"
and thus can notify handlers. These log messages are printed to stdout
and to a log file in the format `hostname_DTG.txt` in the `LOG_PATH`
variable computed by the control machine earlier in the playbook.
For example: `csrnat_20180115T183845.txt`.

The network device hostname and DTG are also written in the text of the
log to make it easy to bookmark certain events during concatenation.
The output below illustrates the use of `cat` and the value of
these text labels.

```
[ec2-user@ansible natm]$ tree logs/
logs/
├── natm_20180601T115455
│   └── csrnat.txt
└── natm_20180601T115632
    └── csrnat.txt

[ec2-user@ansible natm]$ cat \
> logs/natm_20180601T115455/csrnat.txt \
> logs/natm_20180601T115632/csrnat.txt
! BEGIN csrnat @ 20180601T115455
! Updates:
ip nat name TEST_1 inside source static 192.168.0.1 100.64.0.1
ip nat name TEST_3 inside source static 192.168.0.3 100.64.0.3
!
! END   csrnat @ 20180601T115455
! BEGIN csrnat @ 20180601T115632
! Updates:
no ip nat name TEST_1
!
! END   csrnat @ 20180601T115632
```
