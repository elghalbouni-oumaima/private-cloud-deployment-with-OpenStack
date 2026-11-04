# Private Cloud with OpenStack: Multi-Node Deployment (Atelier 2)

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

## 5. Screenshots

| Evidence | File |
|---|---|
| DevStack completion summary | `screenshots/stack-complete.png` |
| `service list`, `network list`, `image list` | `screenshots/controller-check-*.png` |
| Compute nodes `up` and hypervisor list | `screenshots/compute-nodes.png` |
| Flavor, network, router, keypair, rules | `screenshots/part3-*.png` |
| Instance `ACTIVE` on compute1 | `screenshots/instance.png` |
| Ping, SSH, security-rule test | `screenshots/part4-*.png` |
| Horizon dashboard and hypervisors | `screenshots/horizon-*.png` |


## 6. Problems met and solutions

| Problem | Cause | Solution |
|---|---|---|
| RabbitMQ would not start | `controller` resolved to an IP the VM did not own | Aligned IP, `/etc/hosts` and `local.conf` |
| Empty `keystone` / `cinder` folders | Interrupted git clones | Deleted them, re-cloned with `--depth 1` and retry loop |
| Very slow downloads (5 to 10 KiB/s) | Weak connection | Shallow clones, longer git timeout |
| `local.conf` settings ignored | Lines placed after a `[[post-config]]` block | Moved them into `[[local\|localrc]]` |
| MySQL failed to restart | MariaDB leftovers | Purged MariaDB, reinstalled MySQL |
| etcd failed to start | Ubuntu `etcd` already used ports 2379/2380 | Stopped and removed it |
| IPv6 subnet MTU error | Tailscale MTU is 1280 | Disabled IPv6 (`IP_VERSION=4`) |
| `Missing value auth-url` | Credentials not loaded | `source ~/devstack/openrc admin admin` |
| Security rule command ambiguous | Two groups named `default` | Used the security group ID |
| Instance stuck in `BUILD` | compute1 was `down` (VPN or sleep) | Restored the node, recreated the instance |

## 7. Skills demonstrated

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


## 8. Limits and next steps

- Lab setup: QEMU emulation, VPN transport, one controller (no high availability), DevStack (not production-grade).
- Possible next steps: automate with Ansible/Terraform, add Cinder volumes, add a second compute scheduling test, move to Kolla-Ansible or a production-style install, learn Kubernetes on top of IaaS.

## 9. References

- OpenStack documentation: https://docs.openstack.org/
- DevStack documentation (multi-node lab): https://docs.openstack.org/devstack/latest/
- Tailscale documentation: https://tailscale.com/kb/
