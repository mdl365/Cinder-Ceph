Requirements to be a Node: 

Python 3

Systemd

Podman or Docker for running containers

Time synchronization (such as Chrony or the legacy ntpd)

LVM2 for provisioning storage devices

All should be installed except Podman:

All devices need an address in Public and Storage Network

```yaml
sudo apt install podman
```

Deploy Admin Node

1.	Connect compute/controller node to switch (optionally a VyOS object) 

2.	Connect switch to Admin Node, configure that node with Public Network IP and storage network IP (2 networks)
    a. if connecting to management network, this object will need 3 NICs

Installing Cephadm (on admin node)

3. 
```yaml
apt install -y cephadm
```

    if not able to use that:
```yaml
CEPH_RELEASE=replace this with the active release (https://docs.ceph.com/en/latest/releases/#active-releases)
```

```yaml
curl --silent --remote-name --location https://download.ceph.com/rpm-$<CEPH_RELEASE>/el9/noarch/cephadm
```

4.  Check that the file is executable and can be run from current directory

```yaml 
chmod +x cephadm
```

5.  Install the Cephadm command 

```yaml
./cephadm add-repo --release reef
```
```yaml
./cephadm install
```

6.  Confirm Install with 

```yaml
which cephadm
```

should output "/usr/sbin/cephadm"

Boostrap a New Cluster

*this command creates an MON/MGR on admin node*
*Note: mon-ip= public (client) IP

7.  
```yaml
cephadm bootstrap --mon-ip <mon-ip> --cluster-network <private(storage) address>
```

At this point: Cephadm takes over alot
    a. Creates monitor and manager daemon on node
    b. Adds SSH key for Ceph cluster to root user's file
    c. CONTINUE EXPLAINATION

*Optional: Install Ceph Containers*

```yaml
cephadm add-repo --release quincy
```

```yaml
cephadm install ceph-common
```

8.  Confirm Install with: 

```yaml
ceph -v
```

9.  Check Cluster Connection and Status

```yaml
ceph status
```

Deploy Additional Nodes

*We deployed monitor and manager on the admin node, all remaining nodes will need MON and OSD*

10. Run these commands on Admin node for all hosts we want to add: 

```yaml
ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host-ip>*
```

11. Create a Hosts.yaml file in the current directory

Example Structure:

```yaml 
service_type: host
hostname: node-00
addr: 192.168.0.10
labels:
- example1
- example2
---
service_type: host
hostname: node-01
addr: 192.168.0.11
labels:
- grafana
---
```

12. Run the YAML file

```yaml
ceph orch apply -i hosts.yaml
```

13. Deploy 2 more MON nodes

```yaml
ceph orch daemon add mon hostnames_from_YAML
```

14. Tell Ceph to consume all unused device

```yaml
ceph orch apply osd --all-available-devices
```


https://docs.ceph.com/en/latest/rbd/rbd-openstack/






