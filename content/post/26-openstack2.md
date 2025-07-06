---
title: "(2)사내 Openstack 구축으로 VM 대체하기 - nova, neutron, cinder"
author: Bumgu
draft: false
date: 2025-07-06T12:08:08+09:00
categories: 
    - Openstack
    - Cloud
tags: 
    - Openstack
slug: "building_private_cloud_using_openstack-2"
---
[Openstack 구축 1편](https://blog.bumgu.com/post/2025/06/29/building_private_cloud_using_openstack-1/)

---

# Nova
Openstack에서 Nova는 클라우드 환경의 Compute 리소스를 관리하는 핵심 컴포넌트 이다.  
인스턴스의 생성, 관리, 삭제 등 라이프사이클 전체를 담당한다.  
또한 neutron, cinder 등과 연계하여 인스턴스에 네트워크와 스토리지를 연결하는 필수 컴포넌트 이다.

### Controller Node

#### 1. MariaDB에 Nova 데이터베이스 생성

* `create database nova_api`  
  - Nova의 API 서비스와 관련된 글로벌 정보(예: 인스턴스 요청, 셀 정보, 고수준 트래킹 등)를 저장  
* `create database nova`  
  - 실제 인스턴스(가상 머신)와 관련된 데이터를 저장하는 기본 셀(cell1)의 데이터베이스. 인스턴스의 상태, 할당된 자원, 이벤트 등 각 셀의 로컬 정보를 보관  
* `create database nova_cell0`  
  - Nova의 셀 구조에서 스케줄링에 실패한 인스턴스(어떤 컴퓨트 노드에도 할당되지 못한 인스턴스)를 저장하는 특수한 용도의 데이터베이스.

```
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_API_PASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_API_PASS';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_PASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_PASS';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_CELL0_PASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_CELL0_PASS';
```
 
#### 2. Nova 유저 생성 및 권한 부여
* 유저 생성
    - `openstack user create --domain default --password-prompt nova`  
* 관리자 권한 부여
    - `openstack role add --project service --user nova admin`  
* Nova 서비스 엔티티 생성
    - `openstack service create --name nova --description "Openstack Compute" compute`


#### 3. Nova API 서비스 엔드포인트 생성
```bash
$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 08ab870ef54647399f31472a22e3ec23 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25ca5a23aa5b4375889d931e11467e61 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 59b80313047f41eb89e429600eff9a71 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25ca5a23aa5b4375889d931e11467e61 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 12a2aab58e0f4a87b1f41e6e6486ef5c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 25ca5a23aa5b4375889d931e11467e61 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

#### 4. Nova 패키지 설치

`api install nova-api nova-conductor nova-novncproxy nova-scheduler -y`

#### 5. Nova 설정

`/etc/nova/nova.conf`

```
[api_database]
# nova 유저 nova_api 데이터베이스
connection = mysql+pymysql://nova:NOVA_API_PASS@controller/nova_api

[database]
# nova 유저 nova 데이터베이스
connection = mysql+pymysql://nova:NOVA_PASS@controller/nova
```

```
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
my_ip = 1.1.1.1 # 컨트롤러 노드의 IP
```
RabbitMQ 유저 생성은 [Openstack 구축 1편](https://blog.bumgu.com/post/2025/06/29/building_private_cloud_using_openstack-1/)

```
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
```

```
[service_user]
send_service_user_token = true
auth_url = https://controller:5000/v3
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = NOVA_PASS
```

```
[vnc]
# $my_ip로 쓰면 DEFAULT 섹션에 정의한 my_ip가 적용된다.
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

```
[glance]
api_servers = http://controller:9292
```

```
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

```
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

#### 6. nova 데이터베이스 생성
* nova-api 생성
    - `su -s /bin/sh -c "nova-manage api_db sync" nova`

* cell0 데이터베이스 register
    - `su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova`

* cell1 셀 생성
    - `su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova`

* nova 생성
    - `su -s /bin/sh -c "nova-manage db sync" nova`

* cell0과 cell1이 유효하게 register됐는지 확인
    - `su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova`

**만약 nova-manage db sync가 오류 날 경우**  
높은확률로 cell0이 잘못된 데이터베이스에 매핑이 된경우이니 셀을 삭제하고 다시 매핑하면된다.
```
CELL0_UUID=$(nova-manage cell_v2 list_cells | grep 'cell0' | awk '{print $4}')
su -s /bin/sh -c "nova-manage cell_v2 delete_cell --cell_uuid $CELL0_UUID" nova
```
위 명령어로 셀을 삭제후 다시 매핑한다.

### Compute Node

#### 1. nova-compute 패키지 설치

`api install nova-compute -y`

#### 2. nova-compute 설정
`/etc/nova/nova.conf`

```
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 2.2.2.2
```

```
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
```

```
[service_user]
send_service_user_token = true
auth_url = http://controller:5000/v3
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = NOVA_PASS
```

```
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

```
[glance]
api_servers = http://controller:9292
```

```
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

```
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

설정을 다 했다면 서비스 재시작 전에 `egrep -c '(vmx|svm)' /proc/cpuinfo` 를 실행한다.  
만약 0이상이 출력이 된다면 하드웨어 가속(Hardware acceleration)이 가능하기때문에 추가 설정이 필요 없고,  
만약 0이 출력된다면 `/etc/nova/nova-compute.conf`에서

```
[libvirt]
virt_type = qemu
```
로 수정해야한다.  참고로 **Mac은 nested virtualization을 지원하지 않는다.**  
즉 Mac위에 VM을 실행하고 오픈스택을 구성할 수 없다.

### 3. 서비스 재시작
`service nova-compute restart`


### 다시 Controller Node

* compute host가 데이터베이스에 있는지 확인
    - `openstack compute service list --service nova-compute`

* compute host 디스커버
    - `su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova`  
    위 명령어를 실행했을때 `Found 0 unmapped computes in cell`이 나오면 된다.



# Neutron
Neutron은 클라우드 환경의 네트워크 서비스와 가상네트워크인프라(VNI)를 생성, 관리하는 핵심 컴포넌트 이다.  
Neutron은 물리 네트워크 접근 계층까지 관리해서 서브넷, 라우터등을 자유롭게 생성,관리 할 수 있고,  
VXLAN등을 생성해 테넌트별 분리된 네트워크를 제공할 수 있다.  
또한 DHCP에이전트를 통해 인스턴스에 동적 IP를 할당하고 Floating IP를 통해 퍼블릭 IP를 부여할 수 있다.



#### 1. 데이터베이스 생성
`create database neutron;`

```
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_PASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_PASS';
```

#### 2. neutron 유저 생성 및 권한 부여

* 유저 생성
    - `openstack user create --domain default --password-prompt neutron`
* 관리자 권한 부여
    - `openstack role add --project service --user neutron admin`
* Neutron 서비스 엔티티 생성
    - `openstack service create --name neutron --description "OpenStack Networking" network`


#### 3. neutron API 서비스 엔드포인트 생성

```
$ openstack endpoint create --region RegionOne \
  network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6ceee173e834461da782b70208c6d4a5 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b42a173fcb674d6f867eef425e100e5a |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ecbb6670e71e4199af33a22ab0b7e5e8 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b42a173fcb674d6f867eef425e100e5a |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
$ openstack endpoint create --region RegionOne \
  network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e32b980a7e12447b84e1b2626ed5859b |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b42a173fcb674d6f867eef425e100e5a |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

#### 4. neutron 패키지 설치
```
apt install neutron-server neutron-plugin-ml2 \
  neutron-dhcp-agent \
  neutron-metadata-agent neutron-linuxbridge-agent \
  neutron-l3-agent -y
```
나는 `neutron-linuxbridge-agent`를 사용했다.

#### 5. neutron 설정
`/etc/neutron/neutron.conf`

```
[database]
connection = mysql+pymysql://neutron:NEUTRON_PASS@controller/neutron
```

```
[DEFAULT]
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
```

```
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = NEUTRON_PASS
```

```
[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
```

```
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```
[experimental]
# linuxbridge는 아직 실험적인 기술이다.
linuxbridge = true
```

`/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```
# Network Node와 agent Node 둘다 해줘야 한다

[vxlan]
enable_vxlan = True # vxlan을 사용한다면 True
local_ip = 192.168.20.8 # 노드의 실제 IP
l2_population = True


[linux_bridge]
# eno1부분을 실제 사용하는  네트워크 인터페이스로 수정한다.
physical_interface_mappings = provider:eno1
```

`/etc/neutron/dhcp_agent.ini`
```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

`/etc/nova/nova.conf`
```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```


#### 6. 서비스 재시작 및 데이터베이스 생성
```
service restart neutron-*
```

```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

# Cinder
Cinder는 블록 스토리지 기능을 제공하는 컴포넌트 이다.  
데이터의 영속성을 제공할 수 있기 때문에 안정적인 운영이 가능하다.  
Cinder로 볼륨을 생성하고 Server 에 Attach하면 마치 물리서버에 디스크 하나를 더 넣고 mount하는것과 같다.

나는 스토리지 서버에 NFS를 통해 구성했기 때문에 NFS 기준으로 글을 작성한다.

### Controller Node

#### 1. Cinder 데이터베이스 생성
`create database cinder;`
```
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```

#### 2. Cinder 유저 생성 및 권한 부여
* Cinder 유저 생성
    - `openstack user create --domain default --password-prompt cinder`
* 관리자 권한 부여
    - `openstack role add --project service --user cinder admin`
* cinderv3 서비스 엔티티 생성
    - `openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3`


#### 3. Cinder API 서비스 엔드포인트 생성
```
$ openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | b864942e6f1745e1a01d75fd172f47d1         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 511815a9644d469a928371a94d41f293         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
$ openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | f100d7e7333a4de58bfaa4fbfcbfbbeb         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 511815a9644d469a928371a94d41f293         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
$ openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | b16c62c737d4459b91b23707dcb51e9d         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 511815a9644d469a928371a94d41f293         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```

#### 4. 패키지 설치

* cinder 패키지
    - `apt install cinder-api cinder-scheduler -y`
* nfs 패키지
    - `apt install nfs-common -y`

#### 5. Cinder 설정
`/etc/cinder/cinder.conf`
```
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
```

```
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```

```
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 1.1.1.1
```

```
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

`/etc/nova/nova.conf`
```
[cinder]
os_region_name = RegionOne # 실제 region 이름
```

#### 6. Cinder 데이터베이스 생성
`su -s /bin/sh -c "cinder-manage db sync" cinder`  

#### 7. 서비스 재시작
on Compute node:
```
service nova-api restart
```

on Controller node:
```
service cinder-scheduler restart
service apache2 restart
```

### Storage Node

#### 1. 패키지 설치
* cinder 패키지
    - `apt install cinder-volume -y`
* nfs 패키지
    - `apt install nfs-common -y`

#### 2. cinder 설정
`/etc/cinder/cinder.conf`
```
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
```

```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 1.1.1.1
enabled_backends = nfs # lvm을 사용한다면 lvm으로 하면된다.
glance_api_servers = http://controller:9292
```

```
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```

```
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

```
[nfs]
# NfsDrive를 volume_driver로 설정
volume_driver = cinder.volume.drivers.nfs.NfsDriver
volume_backend_name = nfs-storage

# NFS 설정파일의 경로
nfs_shares_config = /etc/cinder/nfs_shares 

# cinder 데이터가 저장될 경로(이 경로를 nfs 마운트 해주면 된다)
nfs_mount_point_base = /var/lib/cinder/volumes
```

`/etc/cinder/nfs_shares`
```
#NFSIP:NFSPATH
2.2.2.2:/export/cinder
```
nfs서버의 IP:경로 형식으로 설정한다 Cinder가 알아서 마운트 하기때문에 따로 마운트 설정을 할 필요가 없다.

# Test

이제 모든 설정이 끝났으니 테스트를 해보자.

#### 1. provider 네트워크
**네트워크 생성**
```
# --external 옵션으로 외부 네트워크임을 명시
# --provider-physical-network provider: 우리가 linuxbridge_agent.ini에 설정한 이름
# --provider-network-type flat: VLAN 태깅 없이 물리 네트워크와 직접 연결 
openstack network create --share --external \
--provider-physical-network provider \
--provider-network-type flat provider
```

**서브넷 생성**
```
# 예: 실제 네트워크가 220.92.114.0/24 라면
openstack subnet create --network provider \
--allocation-pool start=220.92.114.8,end=220.92.114.40 \
--dns-nameserver 8.8.8.8 \
--gateway 220.92.114.1 \
--subnet-range 220.92.114.0/24 provider-subnet

- `--allocation-pool`: 인스턴스에 할당할 Floating IP의 범위. DHCP 범위와 겹치지 않게 설정
- `--gateway`: 실제 물리 네트워크의 게이트웨이 주소.
- `--subnet-range`: 실제 물리 네트워크의 전체 대역.
```
provider 네트워크는 Public 네트워크, 즉 Public IP를 사용하는 네트워크라고 생각하면된다.

#### 2. selfservice network

**네트워크 생성**
```
openstack network create selfservice
```

**서브넷 생성**
```
openstack subnet create --network selfservice \
  --dns-nameserver 8.8.8.8 --gateway 172.16.1.1 \
  --subnet-range 172.16.1.0/24 selfservice
```
selfservice 네트워크는 인스턴스 내부에서 사용하는 네트워크이다.  
네트워크 노드에 router를 생성하고 이 router에 selfservice와 provider를 연결하여 NAT를 통해 외부와 통신이 된다.  
또한 이 selfservice를 통해 테넌트별로 분리된 네트워크 생성이 가능하다.  
ex)운영환경 172.17.1.0, 개발환경 172.16.1.0

#### 3. router 생성

```
# 라우터 생성
openstack router create router

# 라우터를 내부망(selfservice)에 연결
openstack router add subnet router selfservice

# 라우터를 외부망(provider)에 연결 (게이트웨이 설정)
openstack router set router --external-gateway provider
```

#### 4. OS 이미지 등록
VM을 생성할때 쓰는 이미지 말고 클라우드 이미지가 필요하다. 테스트용으로 가벼운 cirros를 설치한다.  
```
# 이미지 다운로드
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img

# Glance에 이미지 업로드
openstack image create "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

#### 5. Key 생성
```
openstack keypair create --private-key mykey.pem mykey
chmod 600 mykey.pem
```

#### 6. 보안그룹 규칙 생성

```
openstack security group rule create --proto tcp --dst-port 22 default
```

#### 7. Flavor 생성

flavor는 AWS ec2인스턴스를 생성할때 선택하는 타입이다. (m2.large 같은)  
cirros는 굉장히 가벼운 os라 작게 생성한다.  
```
# ram 512MB, Disk 1GB, vCPU 1개 flavor이름은 m1.tiny
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny
```

#### 8. 인스턴스 생성

```
# selfservice 네트워크에 Cirros 이미지를 사용하여 인스턴스 생성
openstack server create --flavor m1.tiny --image cirros \
  --nic net-id=$(openstack network show selfservice -f value -c id) \
  --security-group default --key-name mykey test-instance
```

`openstack server list` 로 확인 할 수 있다.

#### 9. Floating IP 생성 및 연결

```
# Floating IP 생성 (외부 IP)
openstack floating ip create provider

# 생성한 Floating IP를 인스턴스에 연결
# FLOATING_IP 부분에는 위 명령으로 출력된 IP 주소를 입력

openstack server add floating ip test-instance [FLOATING_IP]
```

#### 10. ping 및 ssh
`ping <floating_ip>`

keypair를 등록해놨기 때문에 바로 사용이가능하다.  
cirros의 기본사용자는 `cirros`이다.  
`ssh -i mykey.pem cirros@<floating_ip>`

![image](/images/post/26-openstack2/1.png)

글을 적는 시점으로 난 글만 적었지 실제 테스트 인스턴스는 생성하지 않았습니다.  
다음글에서 Terraform, Pyton으로 IaC 적용 및 실제 인스턴스 생성을 하겠습니다!

---

[Openstack 구축 1편](https://blog.bumgu.com/post/2025/06/29/building_private_cloud_using_openstack-1/)
