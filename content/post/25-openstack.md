---
title: "(1)사내 Openstack 구축으로 VM 대체하기 - keystone, glance, placement"
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

# 발단
VM이 더이상 유지보수도 안되고 라이센스도 끝나서 대체 하기로 마음을 먹고 Openstack 구축을 시작했다.  
이전에 [Openstack Helm](https://docs.openstack.org/openstack-helm/latest/) 으로 ~~날먹~~ 해보려했으나 잘 되지 않아서 흐지부지 됐었으나.. 
6월4일에 열렸던 [Cloud bro](www.cloudbro.ai) 세미나에서 NHN Cloud 조성수님에게 여쭤봤다가 처음부터 helm으로 한번에 설치하려하지 말고 가이드보고 컴포넌트별로 설치하며 트러블 슈팅을 해보란 말씀에 하나하나씩 처음부터 설치를 해봤다.  



# 환경
* OS: Ubuntu 24.04 3대  
* 1번서버 : Controller, Network, Compute  
* 2번서버 : Compute  
* 3번서버 : Compute  

자원의 한계로 사내에 남는 서버를 주워서 구축했다... 1번서버야 미안해


# Prerequisite
오픈스택 컴포넌트(Keystone, Nova 등등)을 제외하고 사전에 필요한 소프트웨어들이 있다.  
* Mysql(MariaDB)  
    >Mysql로 해도 되지만 공식문서에서 MariaDB로 하는것 같아서 MariaDB로 설치했다.

    `apt-get update -y && apt install mariadb-server -y`
* Openstack client
    >openstack 명령어를 통해 openstack을 컨트롤할 패키지다 (Kubernetes의 kubectl이라고 생각하면 된다.)  

    `apt install python3-openstackclient`
* RabbitMQ
    >Openstack은 여러 서비스가 상호작용하며 동작하는 복잡한 상호작용하며 동작하는 복잡한 분산 시스템이다.  
    >RabbitMQ는 이러한 서비스 간의 작업과 상태 정보를 교환하고 조정하는 데 사용된다.  
    >메시지 큐를 통해 각 서비스가 비동기적으로 데이터를 주고받을 수 있어 시스템의 응답성을 높인다.

`apt install rabbitmq-server`  
`systemctl start rabbitmq-server`

공식문서에서 IP 기반이 아니라 `controller`와 같은 호스트네임으로 설정파일에 지정하기때문에  
모든 서버 `/etc/hosts`에 `192.168.64.22(컨트롤러 ip) controller` 와 같이 호스트네임 기반으로 통신할 수 있게 설정해놓는 것이 편하다.    
또한 `openstack client`를 통해 openstack과 통신할 때 필요한 인증정보를 `admin-openrc`파일에 작성하고 편한곳에 저장한다.  
```
export OS_USERNAME=admin
export OS_PASSWORD=사용할 admin비밀번호
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
openstack 명령어를 사용하기전에 `source admin-openrc`로 꼭 적용하고 명령어를 사용해야 한다.  

openstack 이 사용할 RabbitMQ 유저 생성 및 권한 부여
```bash
sudo rabbitmqctl add_user openstack RABBIT_PASS
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

---



# Keyston
[공식 설치 문서](https://docs.openstack.org/keystone/2025.1/install/)  

keystone은 openstack의 중앙 집중식 인증 및 인가 서비스를 담당한다.  
사용자와 서비스의 접근 권한을 관리하며 시스템 보안/인증을 유지하는 구성 요소이다.  
openstack에서 모든서비스 (nova, neutron, cinder 등등)은 keystone을 통해 사용자 인증을 수행하고, 권한을 부여받아 자원에 접근한다.  

#### 1. MariaDB에 keystone 데이터베이스 생성  
`create database keystone;`
#### 2. 유저 생성 및 권한 부여

```
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_PASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_PASS';
```

#### 3. keystone 설치 및 설정
`apt install keystone`

설치 후, `/etc/keystone/keystone.conf`  
```
[database]
connection = mysql+pymysql://keystone:KEYSTONE_PASS@controller/keystone
[token]
provider = fernet
```

#### 4. keystone DB 생성
`su -s /bin/sh -c "keystone-manage db_sync" keystone`


#### 5. Fernet key repository 초기화

`keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`  
`keystone-manage credential_setup --keystone-user keystone --keystone-group keystone`  

#### 6. 인증 서비스 부트스트랩

```
keystone-manage bootstrap --bootstrap-password KEYSTONE_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne # region이름은 원하는대로 하면된다.
```
`/etc/apache2/apache2.conf`
```
ServerName controller
```

---

# Glance
[공식 설치 문서](https://docs.openstack.org/glance/2025.1/install/)
glance는 openstack의 이미지 관리 서비스이다. VM의 이미지를 저장, 검색, 등록 및 관리하는데 사용하며,  
인스턴스를 생성할 때 필요한 OS 이미지도 glance를 통해서 등록이된다.

#### 1. MariaDB에 glance 데이터베이스 생성  
`create database glance;`
#### 2. 유저 생성 및 권한 부여

```
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_PASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_PASS';
```

#### 3. glance 유저 생성
	`openstack user create --domain default --password-prompt glance`
#### 4. project service 생성
	`openstack project create --domain default --description "Service Project" service`
#### 5. glance 유저와 service project에게 admin role 부여
	`openstack role add --project service --user glance admin`
#### 6. glance service entity 생성
	`openstack service create --name glance --description "OpenStack Image" image`



#### 7. image service API Endpoint 생성

여기서 `RegionOne`은 keystone 부트스트랩 할때 설정했던 region 이름을 써주면 된다.  
같은 명령어를 public, internal, admin 에게 해준다.
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


#### 8. glance 설치 및 설정
`apt install glance`

설치 후 `/etc/glance/glance-api.conf`
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
region_name = RegionOne #사용한 region 이름
```


#### 9. glance 유저에게 시스템 전역 읽기 권한 부여
`openstack role add --user glance --user-domain Default --system all reader`

#### 10. glance database 생성
`su -s /bin/sh -c "glance-manage db_sync" glance`

#### 11. 서비스 재시작
`service glance-api restart`

---

# Placement
[공식 설치 문서](https://docs.openstack.org/placement/2025.1/install/)  
placement는 리소스를 관리하며 자원을 추적하고 자원 할당을 최적화 한다.(kubernetes의 scheduler 같은 역할)

#### 1. MariaDB에 placement 데이터베이스 생성  
`create database placement;`
#### 2. 유저 생성 및 권한 부여
```
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_PASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_PASS';
```

#### 3. placement service 유저 생성
	openstack user create --domain default --password-prompt placement

#### 4. placement 유저에게 service project admin 권한 부여
	openstack role add --project service --user placement admin

#### 5. placement API 서비스 생성
	openstack service create --name placement --description "Placement API" placement


#### 6. placement API Service Endpoint 생성
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

#### 7. placement 생성 및 설정
`apt install placement-api`  

설치 후, `/etc/placement/placement.conf`  

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

#### 8. placement 데이터베이스 생성
`su -s /bin/sh -c "placement-manage db sync" placement`

#### 9. placement로부터 새로운 설정을 받아서 조정하기 위해 웹서버(apache2) 재기동
`service apache2 restart`

---


글이 너무 길어질거같아 더 복잡한 Nova, Neutron은 다음 글로 작성하겠습니다.
