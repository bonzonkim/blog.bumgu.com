---
title: "(1) Replacing VMs by Building an In-House OpenStack - Keystone, Glance, Placement"
author: Bumgu
draft: false
date: 2025-06-29T12:13:11+09:00
categories: 
    - Openstack
    - Cloud
tags: 
    - Openstack
slug: "building_private_cloud_using_openstack-1"
---
[OpenStack Setup, Part 2](https://blog.bumgu.com/en/post/2025/07/06/building_private_cloud_using_openstack-2/)

---

# Background
Our VMs were no longer maintained and the license had expired, so I decided to replace them and started building OpenStack.  
I had previously tried to ~~take the easy way out~~ install everything with [OpenStack Helm](https://docs.openstack.org/openstack-helm/latest/), but it didn't go well and the effort petered out...  
At the Cloud bro seminar on June 4, I asked Seongsu Cho from NHN Cloud about it. He advised me not to try installing everything at once with Helm, but to follow the guide, install components one by one, and troubleshoot along the way. So I started from scratch, one component at a time.

# Environment
* OS: Ubuntu 24.04, 3 servers  
* Server 1: Controller, Network, Compute  
* Server 2: Compute  
* Server 3: Compute  

I built this using spare servers I found around the office, due to resource constraints... Sorry, server 1.

# Prerequisites
Apart from OpenStack components (Keystone, Nova, and so on), some software is needed beforehand.  
* Mysql (MariaDB)  
    >MySQL would also work, but the official documentation seemed to use MariaDB, so I installed MariaDB.

    `apt-get update -y && apt install mariadb-server -y`
* OpenStack client
    >This is the package used to control OpenStack via the `openstack` command (think of it as `kubectl` for Kubernetes).

    `apt install python3-openstackclient`
* RabbitMQ
    >OpenStack is a complex distributed system in which multiple services interact.  
    >RabbitMQ is used to exchange and coordinate tasks and state information between these services.  
    >The message queue lets each service exchange data asynchronously, improving system responsiveness.

`apt install rabbitmq-server`  
`systemctl start rabbitmq-server`

The official documentation uses hostnames such as `controller` in configuration rather than IP addresses, so it is convenient to configure `/etc/hosts` on every server to allow hostname-based communication, for example `192.168.64.22(controller IP) controller`.  
Also create an `admin-openrc` file containing the credentials required for the `openstack client` to communicate with OpenStack, and save it somewhere convenient.
```
export OS_USERNAME=admin
export OS_PASSWORD=your_admin_password
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
Before running OpenStack commands, be sure to apply this with `source admin-openrc`.  

Create a RabbitMQ user for OpenStack and grant it permissions:
```bash
sudo rabbitmqctl add_user openstack RABBIT_PASS
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

---

# Keystone
[Official installation guide](https://docs.openstack.org/keystone/2025.1/install/)  
Keystone provides OpenStack's centralized authentication and authorization service.  
It manages user and service access permissions and is a component that maintains system security and authentication.  
In OpenStack, all services (Nova, Neutron, Cinder, etc.) authenticate users through Keystone and obtain authorization to access resources.

#### 1. Create a Keystone database in MariaDB  
`create database keystone;`
#### 2. Create a user and grant permissions

```
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_PASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_PASS';
```

#### 3. Install and configure Keystone
`apt install keystone`

After installation, edit `/etc/keystone/keystone.conf`:
```
[database]
connection = mysql+pymysql://keystone:KEYSTONE_PASS@controller/keystone
[token]
provider = fernet
```

#### 4. Populate the Keystone database
`su -s /bin/sh -c "keystone-manage db_sync" keystone`

#### 5. Initialize the Fernet key repository

`keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`  
`keystone-manage credential_setup --keystone-user keystone --keystone-group keystone`  

#### 6. Bootstrap the identity service

```
keystone-manage bootstrap --bootstrap-password KEYSTONE_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne # Choose any region name you want.
```
`/etc/apache2/apache2.conf`
```
ServerName controller
```

---

# Glance
[Official installation guide](https://docs.openstack.org/glance/2025.1/install/)
Glance is OpenStack's image management service. It is used to store, search, register, and manage VM images; OS images required when creating instances are registered through Glance.

#### 1. Create a Glance database in MariaDB  
`create database glance;`
#### 2. Create a user and grant permissions

```
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_PASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_PASS';
```

#### 3. Create the Glance user
	`openstack user create --domain default --password-prompt glance`
#### 4. Create the service project
	`openstack project create --domain default --description "Service Project" service`
#### 5. Grant the admin role to the Glance user and service project
	`openstack role add --project service --user glance admin`
#### 6. Create the Glance service entity
	`openstack service create --name glance --description "OpenStack Image" image`

#### 7. Create image service API endpoints

Here, `RegionOne` should be the region name configured when bootstrapping Keystone.  
Run the same command for public, internal, and admin.
```
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8e1501dcc40943d69cb238bca4f6ec74 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8f2e3853a8db42169e2db576864383a7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e554673c32b94d439b88830a28cd2090 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8f2e3853a8db42169e2db576864383a7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \ 
  image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9d81f1fe8c454ecc869085610d0cb230 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8f2e3853a8db42169e2db576864383a7 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

#### 8. Install and configure Glance
`apt install glance`

After installation, edit `/etc/glance/glance-api.conf`:
```
[database]
connection = mysql+pymysql://glance:GLANCE_PASS@controller/glance
```

```
[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone
```

```
[DEFAULT]
enabled_backends=fs:file

[glance_store]
default_backend = fs

[fs]
filesystem_store_datadir = /var/lib/glance/images/
```

```
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = glance
system_scope = all
password = GLANCE_PASS
endpoint_id = ENDPOINT_ID
region_name = RegionOne # Region name used
```

#### 9. Grant the Glance user system-wide read access
`openstack role add --user glance --user-domain Default --system all reader`

#### 10. Populate the Glance database
`su -s /bin/sh -c "glance-manage db_sync" glance`

#### 11. Restart the service
`service glance-api restart`

---

# Placement
[Official installation guide](https://docs.openstack.org/placement/2025.1/install/)  
Placement manages resources, tracks capacity, and optimizes resource allocation (similar to the Kubernetes scheduler).

#### 1. Create a Placement database in MariaDB  
`create database placement;`
#### 2. Create a user and grant permissions
```
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_PASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_PASS';
```

#### 3. Create the Placement service user
	openstack user create --domain default --password-prompt placement

#### 4. Grant the Placement user admin privileges on the service project
	openstack role add --project service --user placement admin

#### 5. Create the Placement API service
	openstack service create --name placement --description "Placement API" placement

#### 6. Create Placement API service endpoints
```
$ openstack endpoint create --region RegionOne \
  placement public http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4579025eeb5a48c6bfa307a1e83dde4a |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f23e05bed58f4c209d12624a8c28b5b1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  placement internal http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ed7bdd8697f1419d81b80a6247a2ed33 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f23e05bed58f4c209d12624a8c28b5b1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9f0245b46c944e7c90ef4cce20ccd8f8 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f23e05bed58f4c209d12624a8c28b5b1 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

#### 7. Install and configure Placement
`apt install placement-api`  

After installation, edit `/etc/placement/placement.conf`:
```
[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_PASS@controller/placement
```

```
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
```

#### 8. Populate the Placement database
`su -s /bin/sh -c "placement-manage db sync" placement`

#### 9. Restart the web server (apache2) to load the new Placement configuration
`service apache2 restart`

---

This post would get too long, so I'll cover the more complex Nova and Neutron in the next post.

---

[OpenStack Setup, Part 2](https://blog.bumgu.com/en/post/2025/07/06/building_private_cloud_using_openstack-2/)
