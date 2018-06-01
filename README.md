# Network Address Translation Manager (natm)
This playbook automatically manages 1:1 static NAT statements
within a specified VRF on Cisco IOS routers. This is a particularly
labor-intensive and error-prone task, and this playbook
ensures that the desired state of the NAT table is applied.
Non-VRF (global table) operations are also supported. 

## Hosts

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

```
---
vrf: "TEST"
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
and to a log file in the format `natlog_hostname_DTG.txt`. For example,
`natlog_csrnat_20180115T183845.txt`. Liberal use of the Linux `cat`
command can be useful to view the chronology for a given device or
specific point in time:

  * `cat logs/natlog_csrnat*`
  * `cat logs/*20180115T183845.txt`

The network device hostname and DTG are also written in the text of the
log to make it easy to bookmark certain events during concatenation.
The output below illustrates the use of `cat` and the value of
these text labels.

```
[some_user@localhost nat]$ cat logs/natlog_RHN5_PER*
! BEGIN RHN5_PER @ 20180117T023545
! Updates:
no ip nat name TEST_129
no ip nat name TEST_131
!
! END   RHN5_PER @ 20180117T023545
! BEGIN RHN5_PER @ 20180117T024315
! Updates:
ip nat name TEST_130 inside source static 192.168.0.130 100.64.0.130 vrf TEST
ip nat name TEST_131 inside source static 192.168.0.131 100.64.0.131 vrf TEST
!
! END   RHN5_PER @ 20180117T024315
! BEGIN RHN5_PER @ 20180117T024519
! Updates:
no ip nat name TEST_130
no ip nat name TEST_131
!
! END   RHN5_PER @ 20180117T024519
```
