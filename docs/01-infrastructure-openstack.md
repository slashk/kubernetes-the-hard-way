# Cloud Infrastructure Provisioning - OpenStack

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 9 virtual machines will be created.

> Nota Bene: This assumes that you have rights and quotas to create nine virtual machines.

After completing this guide you should have the following compute instances:

```
$ openstack server list
+--------------------------------------+-------------+--------+------------------------+
| ID                                   | Name        | Status | Networks               |
+--------------------------------------+-------------+--------+------------------------+
| e97a7029-0bf4-4a0e-b4af-09184240a9e0 | worker2     | ACTIVE | kubernetes=10.240.0.32 |
| 28ab9a24-e6fd-4f96-944d-0b9599311e5e | worker1     | ACTIVE | kubernetes=10.240.0.31 |
| 2ac8c86f-3cdd-4e68-9b9c-87855c50667e | worker0     | ACTIVE | kubernetes=10.240.0.30 |
| 56cd5af6-26f4-4ef5-be17-2de30aa2a49d | controller2 | ACTIVE | kubernetes=10.240.0.22 |
| 969e0185-f61d-496c-86a2-5a52e071d547 | controller1 | ACTIVE | kubernetes=10.240.0.21 |
| f9d34a71-5ab7-485f-99f9-f195f507899a | controller0 | ACTIVE | kubernetes=10.240.0.20 |
| 2e326862-081d-4c1c-8c77-16ee32b41cf6 | etcd2       | ACTIVE | kubernetes=10.240.0.12 |
| 772ebeb5-fe4c-48a9-b40f-98a05af5062d | etcd1       | ACTIVE | kubernetes=10.240.0.11 |
| be5135bd-2674-4fc2-bf17-a5812e9a5de6 | etcd0       | ACTIVE | kubernetes=10.240.0.10 |
+--------------------------------------+-------------+--------+------------------------+
```

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## SSH Keypair

```
$ openstack keypair create k8s-the-hard-way > k8s-the-hard-way.rsa
```

## Networking

```
$ openstack network create kubernetes
```

```
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2016-09-24T21:06:37                  |
| description               |                                      |
| dns_domain                |                                      |
| headers                   |                                      |
| id                        | 986e3dfb-2ad4-451d-a05e-4350ef30d098 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| mtu                       | 1450                                 |
| name                      | kubernetes                           |
| project_id                | 98fe59987fb74194b865a05627e41618     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 28                                   |
| router:external           | Internal                             |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      | []                                   |
| updated_at                | 2016-09-24T21:06:37                  |
+---------------------------+--------------------------------------+
```

Create a subnet for the Kubernetes cluster on the previously created Kubernetes network (use that network UUID instead of `986e3dfb-2ad4-451d-a05e-4350ef30d098`):

```
$ openstack subnet create kubernetes \
--subnet-range 10.240.0.0/24 \
--network 986e3dfb-2ad4-451d-a05e-4350ef30d098
```

```
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.240.0.2-10.240.0.254              |
| cidr              | 10.240.0.0/24                        |
| created_at        | 2016-09-24T21:13:00                  |
| description       |                                      |
| dns_nameservers   |                                      |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.240.0.1                           |
| headers           |                                      |
| host_routes       |                                      |
| id                | 6c6936eb-76fb-491e-8d4b-dae921b2b618 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | kubernetes                           |
| network_id        | 986e3dfb-2ad4-451d-a05e-4350ef30d098 |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| subnetpool_id     | None                                 |
| updated_at        | 2016-09-24T21:13:00                  |
+-------------------+--------------------------------------+
```


### Firewall Rules

Create the security group for our rules:

```
$ openstack security group create kubernetes
+-------------+---------------------------------------------------------------------------------+
| Field       | Value                                                                           |
+-------------+---------------------------------------------------------------------------------+
| description | kubernetes                                                                      |
| headers     |                                                                                 |
| id          | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5                                            |
| name        | kubernetes                                                                      |
| project_id  | 98fe59987fb74194b865a05627e41618                                                |
| rules       | direction='egress', ethertype='IPv4', id='e892623c-71a3-412c-af62-dd78325dc3fb' |
|             | direction='egress', ethertype='IPv6', id='42bee8c4-427b-4d80-ac10-336ddd5c9556' |
+-------------+---------------------------------------------------------------------------------+
```

Allow pings:

```
$ openstack security group rule create \
--ingress --src-ip 0.0.0.0/0 --protocol icmp kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 44d0852b-ad4a-4798-a8ab-1edd94215583 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```


```
$ openstack security group rule create \
--src-ip 10.240.0.0/24 \
--protocol tcp \
--dst-port 1:65535 kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | a2e1b90b-9752-4fc4-96bc-812bbd5113b0 |
| port_range_max    | 65535                                |
| port_range_min    | 1                                    |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 10.240.0.0/24                        |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```

```
$ openstack security group rule create --ingress \
--src-ip 10.240.0.0/24 \
--protocol udp \
--dst-port 1:65535 kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 46aeb4fe-8512-490a-8f85-2d4fbbd79e33 |
| port_range_max    | 65535                                |
| port_range_min    | 1                                    |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | udp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 10.240.0.0/24                        |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```


```
$ openstack security group rule create \
--src-ip 0.0.0.0/0 --protocol tcp \
--dst-port 3389 kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | faff13a4-c1e1-43a5-9c01-20ea09cbdda9 |
| port_range_max    | 3389                                 |
| port_range_min    | 3389                                 |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```

```
$ openstack security group rule create \
--src-ip 0.0.0.0/0 --protocol tcp \
--dst-port 22 kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 9aff9908-9cca-410f-a531-2aa7f99f8833 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```

TODO: I DON"T THINK WE NEED THIS
```
gcloud compute firewall-rules create kubernetes-allow-healthz \
  --allow tcp:8080 \
  --network kubernetes \
  --source-ranges 130.211.0.0/22
```

```
$ openstack security group rule create \
--src-ip 0.0.0.0/0 --protocol tcp \
--dst-port 6443 kubernetes
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 84c7bbec-d728-41d8-82ac-d7f9addfa098 |
| port_range_max    | 6443                                 |
| port_range_min    | 6443                                 |
| project_id        | 98fe59987fb74194b865a05627e41618     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| security_group_id | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5 |
+-------------------+--------------------------------------+
```


```
$ openstack security group show kubernetes
+-------------+-----------------------------------------------------------------------------------------------------------------------------+
| Field       | Value                                                                                                                       |
+-------------+-----------------------------------------------------------------------------------------------------------------------------+
| description | kubernetes                                                                                                                  |
| id          | 6cc59a3b-b1a4-4c45-b6b6-7a3f8cd3c2f5                                                                                        |
| name        | kubernetes                                                                                                                  |
| project_id  | 98fe59987fb74194b865a05627e41618                                                                                            |
| rules       | direction='egress', ethertype='IPv6', id='42bee8c4-427b-4d80-ac10-336ddd5c9556'                                             |
|             | direction='ingress', ethertype='IPv4', id='44d0852b-ad4a-4798-a8ab-1edd94215583', protocol='icmp',                          |
|             | remote_ip_prefix='0.0.0.0/0'                                                                                                |
|             | direction='ingress', ethertype='IPv4', id='46aeb4fe-8512-490a-8f85-2d4fbbd79e33', port_range_max='65535',                   |
|             | port_range_min='1', protocol='udp', remote_ip_prefix='10.240.0.0/24'                                                        |
|             | direction='ingress', ethertype='IPv4', id='84c7bbec-d728-41d8-82ac-d7f9addfa098', port_range_max='6443',                    |
|             | port_range_min='6443', protocol='tcp', remote_ip_prefix='0.0.0.0/0'                                                         |
|             | direction='ingress', ethertype='IPv4', id='9aff9908-9cca-410f-a531-2aa7f99f8833', port_range_max='22', port_range_min='22', |
|             | protocol='tcp', remote_ip_prefix='0.0.0.0/0'                                                                                |
|             | direction='ingress', ethertype='IPv4', id='a2e1b90b-9752-4fc4-96bc-812bbd5113b0', port_range_max='65535',                   |
|             | port_range_min='1', protocol='tcp', remote_ip_prefix='10.240.0.0/24'                                                        |
|             | direction='egress', ethertype='IPv4', id='e892623c-71a3-412c-af62-dd78325dc3fb'                                             |
|             | direction='ingress', ethertype='IPv4', id='faff13a4-c1e1-43a5-9c01-20ea09cbdda9', port_range_max='3389',                    |
|             | port_range_min='3389', protocol='tcp', remote_ip_prefix='0.0.0.0/0'                                                         |
+-------------+-----------------------------------------------------------------------------------------------------------------------------+
```

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux Kernel that has good support for Docker.

### Virtual Machines

#### etcd Servers

``` sh
for x in 0 1 2; do
  # reserve port with IP address
  openstack port create etcd${x}  \
  --fixed-ip subnet=kubernetes,ip-address=10.240.0.1${x} \
  --network=kubernetes
  # create server with reserved port
  openstack server create \
  --nic port-id=$(openstack port show etcd${x} -f json | jq -r .id) \
  --flavor 2 \
  --key-name k8s-the-hard-way \
  --image e21e7b10-047f-426d-8073-e5230cf0be19  etcd${x}
done
```

#### Kubernetes Controllers

``` sh
for x in 0 1 2; do
  # reserve port with IP address
  openstack port create controller${x}  \
  --fixed-ip subnet=kubernetes,ip-address=10.240.0.2${x} \
  --network=kubernetes
  # create server with reserved port
  openstack server create \
  --nic port-id=$(openstack port show controller${x} -f json | jq -r .id) \
  --flavor 2 \
  --key-name k8s-the-hard-way \
  --image e21e7b10-047f-426d-8073-e5230cf0be19  controller${x}
done
```

#### Kubernetes Workers

``` sh
for x in 0 1 2; do
  # reserve port with IP address
  openstack port create worker${x}  \
  --fixed-ip subnet=kubernetes,ip-address=10.240.0.3${x} \
  --network=kubernetes
  # create server with reserved port
  openstack server create \
  --nic port-id=$(openstack port show worker${x} -f json | jq -r .id) \
  --flavor 2 \
  --key-name k8s-the-hard-way \
  --image e21e7b10-047f-426d-8073-e5230cf0be19  worker${x}
done
```
