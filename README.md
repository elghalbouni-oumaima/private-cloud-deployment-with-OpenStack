# Private Cloud with OpenStack: Multi-Node Deployment

> Academic lab project: ID3, ENSAH, Abdelmalek Essaadi University, 2026/2027
> Team: **Oumaima EL GHALBOUNI** (Controller), **Yousra KHALLOU** (Compute 1), **Firdawss EL HADDOUCHI** (Compute 2)

This repository summarizes how we built a small **private cloud (IaaS)** with **OpenStack**: one controller node and two compute nodes running on three different laptops, connected through a VPN. The detailed report (commands, screenshots, explanations) is provided separately and extends this README.

---

## 1. Goal of the project

Deploy a working OpenStack cloud and prove it works by creating and testing a virtual machine end to end:

- Understand the OpenStack architecture and the role of each service.
- Install a **multi-node** platform (control plane separated from compute plane).
- Create cloud resources: image, flavor, network, router, key pair, security rules, instance.
- Test connectivity, security rules and the instance lifecycle.
- Document the problems met and their solutions.

## 2. Architecture

```
                 Tailscale VPN (private 100.x.x.x network)
   ┌─────────────────────────┐            ┌─────────────────────────┐
   │  CONTROLLER (Oumaima)   │◄──────────►│  COMPUTE 1 (Yousra)     │
   │  Keystone  Glance       │  RabbitMQ  │  nova-compute (QEMU)    │
   │  Nova API/Scheduler     │  MySQL     │  OVN controller         │
   │  Placement  Neutron/OVN │  APIs      │  OVN metadata agent     │
   │  Horizon  MySQL         │            │  ┌──────────────────┐   │
   │  RabbitMQ  etcd         │            │  │ Instance (CirrOS)│   │
   └─────────────────────────┘            │  └──────────────────┘   │
              ▲                           └─────────────────────────┘
              │                           ┌─────────────────────────┐
              └──────────────────────────►│  COMPUTE 2 (Firdaws)    │
                                          │  same services as C1    │
                                          └─────────────────────────┘
```

| Service | Role |
|---|---|
| **Keystone** | Identity, authentication, service catalog |
| **Nova** | Creates and manages virtual machines |
| **Neutron (OVN)** | Virtual networks, routers, security groups |
| **Glance** | Image catalog |
| **Placement** | Tracks available resources for the scheduler |
| **Cinder** | Block storage (studied, not deployed) |
| **Horizon** | Web dashboard |

OpenStack is a **distributed system**: independent services communicate over the network through REST APIs and a message queue (RabbitMQ), with shared state in MySQL.

## 3. Environment and choices

| Item | Choice | Reason |
|---|---|---|
| OS | Ubuntu 24.04 LTS in VirtualBox VMs | Well supported, same on all nodes |
| Method | **DevStack**, branch `stable/2026.1` | Automated, repeatable, suited to labs |
| Hypervisor | **QEMU** (software emulation) | No hardware virtualization inside the VMs |
| Network between nodes | **Tailscale VPN** | Laptops were on different networks; stable IPs |
| Test image | **CirrOS** | Tiny, boots quickly even with QEMU |
| Cinder | Disabled | Not required, saves RAM and downloads |

Controller sizing followed the official guide (8 GB RAM recommended); lab minimums were used where the laptops required it.

## 4. Main steps

### Step 1: Prepare the machines
- Ubuntu 24.04 VM, bridged network adapter, dedicated `stack` user, hostname set (`controller`, `compute1`, `compute2`).
- Clock synchronized, `/etc/hosts` aligned with the Tailscale IPs, 4 GB swap added on the controller.

### Step 2: Connect the nodes (VPN)
- Joined all three VMs to one Tailscale network. Tailscale IPs stay constant when the Wi-Fi changes, so they are used for `HOST_IP` and `SERVICE_HOST`.

### Step 3: Install the controller with DevStack
```bash
git clone https://opendev.org/openstack/devstack -b stable/2026.1
# configure local.conf (HOST_IP, SERVICE_HOST, MULTI_HOST=1, passwords,
#   disable n-cpu and cinder, IP_VERSION=4)
./stack.sh
```
Because the connection was very slow, repositories were pre-cloned with shallow clones (`--depth 1`) and a retry loop.

### Step 4: Verify the controller
```bash
source ~/devstack/openrc admin admin
openstack service list
openstack network list
openstack image list
openstack compute service list
```
Horizon is available at `http://<controller-tailscale-ip>/dashboard`.

### Step 5: Add the compute nodes
- Each compute node runs DevStack with `SERVICE_HOST` pointing to the controller and `ENABLED_SERVICES=n-cpu,placement-client,ovn-controller,ovs-vswitchd,ovsdb-server,q-ovn-metadata-agent`, `LIBVIRT_TYPE=qemu`.
- On the controller, register them:
```bash
./tools/discover_hosts.sh
openstack compute service list --service nova-compute
openstack hypervisor list
```
- **compute1** registered and appeared `up` as a QEMU hypervisor. **compute2**: _update this line with its final status_.

### Step 6: Create the cloud resources 
```bash
openstack flavor create --vcpus 1 --ram 256 --disk 1 m1.test
openstack keypair create mykey > ~/mykey.pem && chmod 600 ~/mykey.pem
openstack network create tp-net
openstack subnet create --network tp-net --subnet-range 192.168.50.0/24 tp-subnet
openstack router create tp-router
openstack router add subnet tp-router tp-subnet
openstack router set --external-gateway public tp-router
openstack security group rule create --proto icmp <default-sg-id>
openstack security group rule create --proto tcp --dst-port 22 <default-sg-id>
openstack server create --flavor m1.test --image cirros-0.6.3-x86_64-disk \
  --network tp-net --key-name mykey --security-group <default-sg-id> test1
```
Result: instance `ACTIVE` on `tp-net` (192.168.50.0/24), scheduled on **compute1**.

### Step 7: Tests and validation
| Test | Command / action | Expected |
|---|---|---|
| Instance started | `openstack server show` | `ACTIVE`, host = compute1 |
| Connectivity | floating IP + `ping` | replies |
| SSH access | `ssh -i mykey.pem cirros@<floating-ip>` | login succeeds |
| Security rules | delete ICMP rule, ping fails; re-add, ping works | rule enforcement proven |
| Lifecycle | `server stop`, `start`, `delete` | `SHUTOFF`, `ACTIVE`, removed |
| Monitoring | Horizon: Admin → Hypervisors / Instances | resources visible |


## 5. Problems met and solutions

### 1. Network and VPN

| Problem | Cause | Solution |
|---|---|---|
| VMs on different subnets (`192.168.1.x`, `192.168.0.x`, `100.93.x.x`) | Each laptop was on a different network | Joined all nodes to a **Tailscale VPN** and used the Tailscale IPs |
| `ping` blocked even to the gateway | Network policy | Tested with `curl -sI https://opendev.org` instead |
| `nmcli con up` failed, then "Network is unreachable" | The connection was left half-configured after switching static to DHCP | Re-activated the connection, restarted NetworkManager, checked the default route |
| Clock off by 1 hour on a node | Time not synchronized | `sudo systemctl restart systemd-timesyncd` |
| `tailscale up` refused to run, `NeedsLogin` | Existing flags and an unauthenticated session | `sudo tailscale up --hostname=controller` and logged in through the link |
| `tailscale ping compute1`: `no matching peer` | compute1 had not joined the tailnet yet | Waited until the teammates joined, then used their `100.x` IPs |

### 2. Installation (DevStack)

| Problem | Cause | Solution |
|---|---|---|
| RabbitMQ would not start | `controller` resolved to an IP the VM did not own | Matched the real IP, `/etc/hosts` and `local.conf` |
| Empty `keystone` and `cinder` folders (`no commits yet`, `setup.cfg` missing) | Interrupted git clones | Deleted them and cloned again with `--depth 1` |
| Git at 5 to 10 KiB/s, `curl 56` TLS errors | Weak connection | Shallow clones with a retry loop, `http.version HTTP/1.1` |
| `Maximum of 3 git retries reached` | DevStack's 600 s clone limit was too short | Pre-cloned the repos, `GIT_TIMEOUT=3600` |
| `local.conf` settings ignored (Cinder still enabled) | Lines appended after `[[post-config]]`, and `c-bak` was not disabled | Moved them above that block, `disable_service c-api c-sch c-vol c-bak cinder` |
| MySQL failed to restart | MariaDB leftovers (I had advised pre-installing `mariadb-server`, but DevStack uses MySQL 8) | Purged MariaDB, reinstalled MySQL |
| etcd failed: `address already in use` | The Ubuntu `etcd` service held ports 2379 and 2380 | Stopped, disabled and removed it |
| `Device or resource busy` on `/opt/stack/data/etcd` | It is a tmpfs mount point used by DevStack | Left it alone |
| `409 ... MTU ... minimum of 1280` (IPv6 subnet) | Tailscale MTU is 1280 | `IP_VERSION=4` |
| `pip install`: `externally-managed-environment` | Ubuntu 24.04 protection | Used DevStack's venv pip, never `--break-system-packages` |
| compute2 stuck cloning `requirements` | DevStack forces a full clone with no timeout | Pre-cloned it with `--depth 1` |
| `c-vol` in a compute node's `ENABLED_SERVICES` | Cinder is disabled on the controller | Removed `c-vol` |

### 3. Command and CLI errors

| Problem | Cause | Solution |
|---|---|---|
| `Missing value auth-url` | Credentials not loaded | `source ~/devstack/openrc admin admin` as `stack` |
| `No recognized column names` on `hypervisor show` | Newer client dropped those columns | Used Placement (`resource provider ...`) or skipped it |
| Syntax error near `newline` | Typed `<UUID>` literally | Replaced placeholders with real values, without `<>` |
| `More than one SecurityGroup ... 'default'` | Several projects have a group named `default` | Used the security group ID |
| `More than one Server exists with the name 'test1'` | Two instances with the same name | Used the instance ID, deleted the duplicate |
| `Guest does not have a console available` | The instance was `SHUTOFF` | Started it first |
| Test 4: rule not deleted, ping still worked | I gave `ICMP_RULE_ID` as a placeholder, and `-c Protocol` should be `-c "IP Protocol"` | Used the real rule ID |

### 4. Instance and network

| Problem | Cause | Solution |
|---|---|---|
| Instance stuck in `BUILD`, host `None` | compute1 was `down`, so the scheduler had no host | Restored compute1, recreated the instance (or used `--availability-zone nova:compute2`) |
| compute1 `down`, OVN agents `XXX` | Laptop asleep, VPN lost or services stopped (cause not confirmed) | Restarted `devstack@*` services, kept the laptop awake |
| Floating IP unreachable, `br-ex` DOWN | `br-ex` lost `172.24.4.1/24`, probably after a reboot | `ip addr add 172.24.4.1/24 dev br-ex` and `ip link set br-ex up` |
| `Destination Host Unreachable` after that | `tp-router`'s gateway was bound to compute1, which was down | Raised the controller's gateway priority in OVN, or brought compute1 back |

### 5. Observations worth citing

- Network MTU is **1222** (Tailscale 1280 minus Geneve overhead).
- Hardware virtualization was unavailable (`egrep -c '(vmx|svm)'` returned 0), so **QEMU** was used, which made VM boots slow.
- The control plane stayed healthy while a worker node dropped out, a typical distributed-systems failure.


## 6. Skills demonstrated

| Skill | Where we used it | Typical use in industry |
|---|---|---|
| **Cloud computing / IaaS** | Deployed and operated an OpenStack cloud | Private and hybrid clouds, AWS/Azure/GCP equivalents |
| **Linux administration** | Ubuntu, systemd, apt, users, logs | Server and DevOps roles everywhere |
| **Virtualization** | QEMU/KVM concepts, VirtualBox, nested virtualization limits | Data centers, VMware/KVM/Proxmox environments |
| **Virtual networking** | Subnets, routers, floating IPs, security groups, OVN/Geneve, MTU | SDN, cloud networking, NFV |
| **VPN and secure connectivity** | Tailscale mesh between nodes | Remote access, site-to-site, zero-trust networking |
| **Distributed systems** | Message queue, DB, service discovery, scheduler | Microservices and large-scale platforms |
| **Identity and access management** | Keystone projects, roles, tokens | IAM in any cloud |
| **Troubleshooting** | Logs, `journalctl`, `ss`, `systemctl` | Operations, SRE, support |
| **Automation basics** | DevStack scripts, shell loops, `local.conf` | Infrastructure as Code, scripting |
| **CLI and API usage** | OpenStack CLI | Cloud automation |
| **Documentation** | Report, error log, architecture | Professional engineering practice |


## 7. Limits and next steps

- Lab setup: QEMU emulation, VPN transport, one controller (no high availability), DevStack (not production-grade).
- Possible next steps: automate with Ansible/Terraform, add Cinder volumes, add a second compute scheduling test, move to Kolla-Ansible or a production-style install, learn Kubernetes on top of IaaS.

## 8. References

- OpenStack documentation: https://docs.openstack.org/
- DevStack documentation (multi-node lab): https://docs.openstack.org/devstack/latest/
- Tailscale documentation: https://tailscale.com/kb/
