# Reordered Documentation: Sample Outputs from Key Deployment Stages

## 📃 Node Summary

| Node       | IP Address      | Hostname   | Roles/Services                                                     |
| ---------- | --------------- | ---------- | ------------------------------------------------------------------ |
| Controller | 192.168.122.100 | controller | Keystone, Glance, Nova API, Neutron, Cinder API/Scheduler, Horizon |
| Compute    | 192.168.122.101 | compute    | Nova Compute, Ceph-integrated                                      |
| Ceph-Admin | 192.168.122.120 | ceph1-mgr  | MON, MGR, Dashboard, Ceph bootstrap/admin                          |
| Ceph-OSD1  | 192.168.122.121 | ceph-osd1  | Ceph OSD Daemon (OSD.0)                                            |
| Ceph-OSD2  | 192.168.122.122 | ceph-osd2  | Ceph OSD Daemon (OSD.1)                                            |
| Ceph-OSD3  | 192.168.122.123 | ceph-osd3  | Ceph OSD Daemon (OSD.2)                                            |

## 🛠️ Ceph Setup and Cluster Preparation

### Hostnames and Static IP Assignment:
Performed on Ceph nodes using `nmcli`.

### Bootstrap Ceph Cluster:
```bash
sudo cephadm bootstrap --mon-ip 192.168.122.201   --initial-dashboard-user admin   --initial-dashboard-password <redacted>
```

### Dashboard Access:
```
https://192.168.122.201:8443/
```

### Add Ceph Hosts to Cluster:
```bash
ceph orch host add ceph-osd1 192.168.122.202
ceph orch host add ceph-osd2 192.168.122.203
ceph orch host add ceph-osd3 192.168.122.204
```

### Verify Available Disks:
```bash
ceph orch device ls ceph-osd3
```

### Create OSDs and Volumes Pool:
```bash
ceph orch daemon add osd ceph-osd3:/dev/vdb
ceph osd pool create volumes 64
```

## 📁 Cinder Integration with Ceph (Controller Node)

### Install and Enable Cinder Services:
```bash
sudo apt install cinder-api cinder-scheduler cinder-volume ceph-common -y
sudo systemctl enable --now cinder-volume
```

### Register Cinder with Keystone:
```bash
openstack user create --domain default --password-prompt cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
```

### Configure `/etc/cinder/cinder.conf`:
```ini
[DEFAULT]
auth_strategy = keystone
enabled_backends = ceph
glance_api_version = 2
transport_url = rabbit://openstack:<password>@controller

[keystone_authtoken]
auth_url = http://controller:5000
www_authenticate_uri = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = <password>

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = <generated-uuid>
volume_backend_name = cep
```

## 🏠 Compute Node Configuration (Nova + Ceph)

### Install Ceph Utilities:
```bash
sudo apt install ceph-common -y
```

### Copy Ceph Config and Keyring:
```bash
scp ceph.conf ceph.client.cinder.keyring /etc/ceph/
chown root: /etc/ceph/ceph.*
chmod 644 /etc/ceph/ceph.*
```

### Update `/etc/nova/nova.conf`:
```ini
[DEFAULT]
my_ip = 192.168.122.111
transport_url = rabbit://openstack:<password>@controller

[libvirt]
images_type = rbd
images_rbd_pool = volumes
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = <same-uuid-as-cinder>
disk_cachemodes="network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
```

## 🔒 Libvirt Secret Configuration

On each compute node:
```bash
uuidgen  # Save the UUID
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid><insert-uuid></uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file secret.xml
sudo virsh secret-set-value --secret <uuid> --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
```

## 📄 Volume Creation and Verification

### Create Volume from Glance Image:
```bash
openstack volume create --image cirros --size 1 boot-from-volume-cirros
```

### List Volumes:
```bash
openstack volume list
```

### Sample Output:
```text
| ID                                   | Name                  | Status    | Size | Attached to |
|--------------------------------------|------------------------|-----------|------|--------------|
| 5bc94aaa-874a-46a3-9201-abf9632f55e1 | boot-from-volume-cirros | available | 1    |              |
```

This reordering retains all the original content but structures it around logical deployment milestones.