---
title: "Terraform으로 GKE를 구축하고 Tailscale로 접근하기"
author: Bumgu
draft: false
date: 2025-12-18T14:21:02+09:00
categories: 
    - DevOps
tags: 
    - GKE
    - GCP
    - Terraform
    - Tailscale
slug: "building-gke-with-terraform-access-with-tailscale"
---
GCP Free trial Credit이 많이 남아서 실무라 가정하고 GKE를 구축하고, Tailscale을 통해 네트워크단에서 접근을 제한하는 작업을 해보았습니다.



# 1. Terraform
폴더구조는 아래와 같습니다.
```bash
.
├── main.tf
├── modules
│   ├── gke
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── network
│   │   ├── main.tf
│   │   └── output.tf
│   └── vm
│       ├── main.tf
│       └── variables.tf
```
`modules`안에서 필요한 리소스를 정의해놓고 root의 `main.tf`에서 실제 리소스를 생성하는 식으로 모듈화 하였습니다.  

## network
먼저 VPC와 서브넷을 생성하는 리소스를 정의하겠습니다.
```terraform
// main.tf
resource "google_compute_network" "vpc" {
  name                    = "gke-module-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  # node의 IP 대역
  name          = "gke-module-subnet"
  region        = "asia-northeast3"
  network       = google_compute_network.vpc.id
  ip_cidr_range = "10.10.0.0/24"

  # GKE용 IP대역 (Pod, Service)
  secondary_ip_range {
    range_name    = "pod-ranges"
    ip_cidr_range = "192.168.0.0/18"
  }
  secondary_ip_range {
    range_name    = "service-ranges"
    ip_cidr_range = "192.168.64.0/18"
  }
}
```
워커노드의 IP는 `10.10.0.0/24`대역을 할당하고, pod와 service가 사용할 IP는 `192.168.0.0/18`,`192.168.64.0/18` 로 할당하였습니다.  
이후 출력된것을 사용할 수 있도록 출력해줍니다. 
```terraform
// output.tf
output "network_name" {
  value = google_compute_network.vpc.name
}

output "subnet_name" {
  value = google_compute_subnetwork.subnet.name
}
```
`network_name`과 `subnet_name`은 root `main.tf`에서 사용할 수 있습니다.


---

## GKE
쿠버네티스 리소스를 정의하도록 하겠습니다.

```terraform
// variables.tf
variable "region" {}

variable "network_name" {}

variable "subnet_name" {}
```
network 모듈로부터 넘겨진 값을 사용할 것 이기 때문에 빈 값으로 둡니다.

```terraform
// main.tf
resource "google_container_cluster" "primary" {
  name     = "modular-cluster"
  location = "${var.region}-a"

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = var.network_name // 변수 사용
  subnetwork = var.subnet_name

  deletion_protection = false

  ip_allocation_policy {
    // network 모듈에서 정의한 pod-ranges와 service-ranges를 사용함
    cluster_secondary_range_name  = "pod-ranges"
    services_secondary_range_name = "service-ranges"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = true
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      display_name = "Internal VPC"
      cidr_block   = "10.10.0.0/24"
    }
  }
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "modular-node-pool"
  location   = "${var.region}-a"
  cluster    = google_container_cluster.primary.name
  node_count = 1 // 워커노드의 수

  node_config {
    spot         = true // 학습목적이기에 spot 인스턴스 사용
    machine_type = "e2-medium"
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}
```
- `priavte_cluster_config`: 외부접근을 모두 차단하는 설정  
- `master_ipv4_cidr_block`: controlplane의 IP대역을 지정하는 것입니다.   
- `enable_private_endpoint`: controlplane의 IP를 퍼블릭 IP가 아닌 사설IP(`master_ipv4_cidr_block`)로 설정하여 외부접근을 차단합니다.  
- `master_authorized_networks_config`: 는 해당 대역의 접근을 허용하는 화이트 리스트입니다. 즉 같은 VPC내에서는 접근이 가능합니다. Tailsclae vm을 같은 VPC내에 둘 것 이기 때문에 이와 같이 설정했습니다.

`deletion_protection`은 학습목적이기에 false로 설정하였습니다.

---

# 실행
root의 `main.tf`파일을 작성합니다.
```terraform
provider "google" {
  project = "gke-learning-481507" // 내가만든 GCP 프로젝트 이름
  region  = "asia-northeast3"
}

module "my_network" { // network 모듈 사용
  source = "./modules/network"
}

module "my_gke" { // gke 모듈 사용
  source = "./modules/gke"
  region = "asia-northeast3"

  network_name = module.my_network.network_name
  subnet_name  = module.my_network.subnet_name
}
```
이대로 두고 `terraform plan` -> `terraform apply`를 합니다.

![image](/images/post/30-gke-tailscale/1.png) 이렇게 실행이 완료되었고, 콘솔에서 확인해보면
![image](/images/post/30-gke-tailscale/2.png) 쿠버네티스 클러스터 생성완료
![image](/images/post/30-gke-tailscale/3.png) vm인스턴스(워커노드) 생성완료
vm instance가 1대인 이유는 node를 1로 설정했고, controlplane은 내가 관리주체가 아니기 때문에 보이지 않습니다.

이제 `gcloud`명령어를 통해 접근해보겠습니다.
`gcloud container clusters get-credentials modular-cluster --location=asia-northeast3-a` 로 context를 가져오고, `kubectl`을 통해 API서버에 요청을 해보면 당연하게도 통신이 안됩니다.  
모든 외부접근을 차단하고, controlplane도 사설IP로 지정했기때문에 외부접근이 원천적으로 불가능합니다.  

그럴때 보통 Bastion서버를 띄우는데 저는 관리가 편한 Tailscale을 사용할 VM을 띄워 접근해보도록 하겠습니다.


## VM
```terraform
variable "region" {}
variable "network_name" {}
variable "subnet_name" {}
```
vm도 역시 GKE처럼 값을 받아와서 사용할 것이기에 빈값으로 선언만해둡니다.

```terraform
// main.tf
resource "google_compute_instance" "tailscale_router" {
  name         = "tailscale-router"
  machine_type = "e2-micro"
  zone         = "${var.region}-a"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  tags = ["tailscale-router"]

  network_interface {
    network    = var.network_name
    subnetwork = var.subnet_name
    access_config {}
  }
  can_ip_forward          = true
  metadata_startup_script = "echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf && sudo sysctl -p /etc/sysctl.d/99-tailscale.conf"
}

# 2. SSH 접속 허용 방화벽 (초기 설정용)
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh-to-router"
  network = var.network_name # 변수 사용

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["tailscale-router"]
}
```
단순 라우터의 역할만 할 것 이기 때문에 가벼운 타입으로 설정합니다.
또한 밑의 Security rule의 `target_tags`가 `tailscale-router`로 설정했기 때문에 vm의 `tags`도 `tailscale-router`를 사용해야 해당 룰이적용되어 SSH 접근이 가능합니다.

`can_ip_forward`와 `metadata_startup_script`는 tailscale의 기능을 사용하기 위해 하는 설정입니다. 제거하고 원격 접근 후에 직접 해도 됩니다.


이제 다시 root의 `main.tf`로 돌아와서 해당 모듈을 사용하는 코드를 작성합니다.

```terraform
provider "google" {
  project = "gke-learning-481507"
  region  = "asia-northeast3"
}

module "my_network" {
  source = "./modules/network"
}

module "my_gke" {
  source = "./modules/gke"
  region = "asia-northeast3"

  network_name = module.my_network.network_name
  subnet_name  = module.my_network.subnet_name
}

// vm 모듈 사용
module "my_tailscale" {
  source = "./modules/vm"
  region = "asia-northeast3"

  network_name = module.my_network.network_name
  subnet_name  = module.my_network.subnet_name
}
```
이제 다시 `terraform plan` -> `terraform apply`를 합니다.

![image](/images/post/30-gke-tailscale/4.png) 2개의 리소스(firewall, vm)이 추가되었습니다.  

`gcloud`명령어를 통해 ssh 접근합니다.  
`gcloud compute ssh tailscale-router --zone asia-northeast3-a` 
접근후 `ip_forward`를 확인합니다.
```sh
~$ cat /etc/sysctl.d/99-tailscale.conf
net.ipv4.ip_forward = 1
```

## Tailscale
`ip_forward`설정까지 확인했으니 이제 tailscale을 설치하고, 라우팅 설정만 해주면 됩니다.  

`curl -fsSL https://tailscale.com/install.sh | sh`  
설치가 완료되면 특정 대역을 라우팅하는 명령어로 실행합니다.

```sh
sudo tailscale up --advertise-routes=10.10.0.0/24,172.16.0.0/28
```
이러면 해당 VM은 `10.10.0.0/24`대역(내부 VPC)과 `172.16.0.0/28`(controlplane)대역으로 라우팅 할 수 있습니다.  
명령어를 입력하면 로그인하라고 뜨는데 로그인을 하고, 이후 넘어가는 admin페이지에서 subnet 승인을 해야됩니다.  
admin페이지에서 해당 머신을 클릭하면 아래와 같이 subnet 항목이보입니다.

![image](/images/post/30-gke-tailscale/5.png)
edit을 눌러주고 승인을 해줍니다.
![image](/images/post/30-gke-tailscale/6.png)


이제 다시 `kubectl`명령어로 API서버에 요청을 해봅니다.
![image](/images/post/30-gke-tailscale/7.png)



# 마무리
Tailscale을 활용하여 GKE에 접근권한을 제한하는 작업을 해봤습니다.  
실무에선 IAM과 Service account 기반으로 조금 더 세세하게 권한을 나누겠지만 지금은 저 혼자이니 아래 레이어인 네트워크단에서 접근을 제한하는 방법을 알아보았습니다.  
베어메탈환경에서 쿠버네티스를 다루다가 매니지드를 다뤄보니 정말 간편하고 쉽습니다.  
크레딧을 활용하면 공짜로 가능하고, 크레딧이 아니어도 비용이 크지 않으니 학습시 충분히 활용할 수 있을 것 같습니다.  
