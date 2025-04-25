Requirements to be a Node: 

Python 3

Systemd

Podman or Docker for running containers

Time synchronization (such as Chrony or the legacy ntpd)

LVM2 for provisioning storage devices

All devices need an address in Public and Storage Network

All should be installed except Podman:

MAKE SURE PODMAN IS INSTALLED, PREFERRED BY CEPH

```yaml
sudo apt install podman
```

Deploy Ceph-Admin Node

1.	Connect compute/controller node to switch (optionally a VyOS object) 

2.	Connect switch to Admin Node, configure that node with Public Network IP and storage network IP (2 networks)
    a. if connecting to management network, this object will need 3 NICs

Installing Cephadm (on admin node)

3. 
```yaml
apt install -y cephadm
```

Note: If installing manually, use version 18.2.2 (last version that has support for our CPU instruction sets (x86/64 v1))
    if not able to use apt install, can also use curl:
```yaml
CEPH_RELEASE=replace this with the active release (https://docs.ceph.com/en/latest/releases/#active-releases)
```

```yaml
curl --silent --remote-name --location https://download.ceph.com/rpm-$<CEPH_RELEASE>/el9/noarch/cephadm
```

4.  Make sure that the file is executable and can be run from current directory

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

*this command creates an MON/MGR/dashboard on admin node*
*Note: mon-ip= public IP
--cluster-network option used if deploying seperate storage network
    --cluster-network <private(storage) address>*

7.  
```yaml
CEPHADM_IMAGE=quay.io/ceph/ceph:v18.2.2 cephadm bootstrap --mon-ip 192.168.122.xxx 
```
*IMPORTANT*: If configuring dashboard, note default username and password are given during bootstrap, keep these for later

At this point: Cephadm takes over alot
    a. Creates monitor and manager daemon on node
    b. Adds SSH key for Ceph cluster to root user's file
    c. CONTINUE EXPLAINATION

*Optional: Install Ceph Containers*

There are two ways to run commands with Ceph, inside a container with the Ceph packages or with the Ceph-common repo to run from command line directly

For Containerized Run: 
```yaml
cephadm shell -- ceph *whatever command*
```
To install Ceph-common: 
```yaml
cephadm add-repo --release quincy
```
```yaml
cephadm install ceph-common
```
8.  Confirm Install with: (Run from wherever in command line, no need to run cephadm shell container first)

```yaml
ceph -v
```

9.  Check Cluster Connection and Status

```yaml
ceph status
```
Preparing Ubuntu for being an OSD HOst:

Host OSDs must not have partitions, a filesystem, LVM state, and cannot be mounted. There must also be more than 5 GB available on the device. 

Before Starting Device: 

a. In GNS3, right click and hit configure object, once the screen opens, click HDD in the tabs at the top. 

b. Under HDB, hit create
    i.) select Qcow2 on first screen, hit next
    ii.) Hit next on second screen, no changes
    iii.) Configure disk size to your liking (must be bigger than 5 GB) and hit finish

Boot/Reboot the Vm at this time, make sure you see the disk you intended to use 
```yaml
sudo fdisk -l
```
Then, If you used HDB like me, follow these commands: If not, replace dev/vdb with the name of the disk you use
```yaml
sudo wipefs --af /dev/vdb
sudo pvremove /dev/vdb || true
sudo dd if=/dev/zero of=/dev/vdb bs=1M count=10
```
This wipes the filesystem, clears out LVM, and clears out the first 10 MBs of data to be safe. The storage device is now a raw block device that can be used. Repeat as neccesary for other OSD nodes. (Minimum= 3 OSD nodes) 

Deploy Additional Nodes

*We deployed monitor and manager on the admin node, all remaining nodes will need MON and OSD*

First, Ceph needs passwordless sudo access on the Ceph Cluster hosts, to enable: 

On node to be OSD, allow root login, run: 
```yaml
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
sudo passwd root
```
When prompted: set the root password to the standard itsclass's password

On Ceph-Admin run: 
```yaml
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<target OSD IP>
```
Back on node to be OSD: 
```yaml 
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
This is just for security, only allows key-based authentication to root user after initial 

Back to Ceph-Admin, run the following to add host:
```yaml
ceph orch host add <hostname> <host IP address>
```
Repeat as Neccesary for all OSD nodes to be deployed.

If need to delete host: 
```yaml
ceph orch host drain <hostname>
```

With Ubuntu-OSD disk cleared and host added, the node can now be configured as an OSD

To Configure Multiple: 
```yaml
ceph orch apply osd --all-available-devices
```
All available devices on hosts in the cluster will become OSDs

To Configure individually: 
```yaml
ceph orch daemon add osd <host>:<device-path>
```
    ex.) ceph orch daemon add osd Ceph-1:/dev/vdb

Run 
```yaml
ceph -s
```
to verify OSDs are available and check cluster health

From here, you will need to integrate Ceph and Cinder, deploy an openstack server, and mount storage from Ceph/Cinder






11. Create a Hosts.yaml file in the current directory:
Was unable to fully flush this out but can allow for whole cluster to be deployed during one bootstrap command

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






