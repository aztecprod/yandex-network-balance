# "Отказоустойчивость в облаке" - Александр Шевцов
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/ef92684e-a78f-4d5c-89aa-48ef31438e1b)

1) Конфигурация файла main.tf
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token     = "y0_AgAAAAAe5bzAAATuwQAAAADegFtPgh5mgJFKSmmN3h_gtGKaIoQD6b8"
  cloud_id  = "b1glk6l56hkqpmajlhbc"
  folder_id = "b1g6c24ejqn6hsh955bf"
  zone      = "ru-central1-b"
}

resource "yandex_compute_instance" "vm" {
  count = 2
  name = "vm${count.index}"


  resources {
    core_fraction = 20
    cores  = 2
    memory = 2
  }

boot_disk {
    initialize_params {
    image_id = "fd8a67rb91j689dqp60h"
    size = 6
    }
  }

metadata = {
 user-data = file("./metadata.yaml")
  }


network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

placement_policy {
    placement_group_id = "${yandex_compute_placement_group.group1.id}"
  }
}



#Target group

resource "yandex_lb_target_group" "testtr-1" {
  name      = "testtr-1"
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }
}

#
#Сетевой Балансировщик

resource "yandex_lb_network_load_balancer" "balance-1" {
  name = "balance-1"
  listener {
    name = "my-list"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

attached_target_group {
    target_group_id = yandex_lb_target_group.testtr-1.id
    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
#


resource "yandex_vpc_network" "network-1" {
  name = "network-1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet-1"
  zone           = "ru-central1-b"
  v4_cidr_blocks = ["192.168.10.0/24"]
  network_id     = "${yandex_vpc_network.network-1.id}"

}

resource "yandex_compute_placement_group" "group1" {
  name = "test-pg1"
}

resource "yandex_compute_snapshot" "snapshot-1" {
  name           = "disk-snapshot"
  source_disk_id = "${yandex_compute_instance.vm[1].boot_disk[0].disk_id}"
}

```
2) Конфигурация metadata.yaml
```
#cloud-config
disable_root: true
timezone: Europe/Moscow
repo_update: true
repo_upgrade: true

apt:
  preserve_sources_list: true

packages:
  - nginx

runcmd:
  - [systemctl, nginx-reload]
  - [systemctl, enable, nginx.service ]
  - [systemctl, start, --no-block, nginx.service]
  - [sh, -c "echo $(hostname | cut -d '.' -f 1)" > /var/www/html/index.html" ]

users:
 - name: user
   groups: sudo
   shell: /bin/bash
   sudo: ['ALL=(ALL) NOPASSWD:ALL']
   ssh-authorized-keys:
     - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCfry91+PfnoYgP0R0E0vQcotrMBm4tesooq4nuL8sTo1a+8ovphhuGUGHCAmIJZqY7N2rnOpBpEsvQtZW0c8dGoFrYeR1ZaC7evHT5gxnbrbW4RTDeWZLsZt84wmlbadYLcCQ2UM6JSESDbd8bY4Vl795LS7+dj1rIgqfiJ2qci169sPY4As73emSFfIslS>

```
2) Cтатус балансировщика и целевой группы.
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/a2067846-b86c-405b-999d-91fe6066bcc4)

![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/2dfe5b3d-9576-4856-9180-dcc1b9d28a96)

![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/16b5f66e-01ba-4352-8752-1840ec86de2e)

![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/cbb75a6d-fc20-4fd7-9354-9048a7f046cc)

3) Скриншот страницы, которая открылась при запросе IP-адреса балансировщика
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/2e0082ac-4b00-4e63-8b18-b96a91a90110)

1) Конфигурация main.tf 
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token     = "y0_AgAAAAAe5bzAAATuwQAAAADegFtPgh5mgJFKSmmN3h_gtGKaIoQD6b8"
  cloud_id  = "b1glk6l56hkqpmajlhbc"
  folder_id = "b1g6c24ejqn6hsh955bf"
  zone      = "ru-central1-b"
}


resource "yandex_compute_instance_group" "ig-1" {
  name               = "fixed-ig-with-balancer"
  folder_id          = "b1g6c24ejqn6hsh955bf"
  service_account_id = "ajeqjqsad0ut1jjhbmsc"
  instance_template {
    platform_id = "standard-v3"
    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      mode = "READ_WRITE"
      initialize_params {
        image_id = "fd8a67rb91j689dqp60h"
      }
    }

    network_interface {
      network_id = yandex_vpc_network.network-1.id
      subnet_ids = ["${yandex_vpc_subnet.subnet-1.id}"]
      nat = true
    }

    metadata = {
      user-data = file("./metadata.yaml")
    }
  }

  scale_policy {
    fixed_scale {
      size = 2
    }
  }

  allocation_policy {
    zones = ["ru-central1-b"]
  }

  deploy_policy {
    max_unavailable = 1
    max_expansion   = 0
  }

  load_balancer {
    target_group_name        = "target-group"
    target_group_description = "load balancer target group"
  }
}
resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "network-load-balancer-1"

  listener {
    name = "network-load-balancer-1-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_compute_instance_group.ig-1.load_balancer.0.target_group_id

    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = "${yandex_vpc_network.network-1.id}"
  v4_cidr_blocks = ["192.168.100.0/24"]
}


```

2) Cтатус балансировщика и целевой группы.
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/73a5bf41-13d7-4410-887e-4db0c8c98b17)
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/35cf6275-dabc-4d6d-9f96-6293a5e1ac3f)
3) Скриншот страницы, которая открылась при запросе IP-адреса балансировщика
![image](https://github.com/aztecprod/yandex-network-balance/assets/25949605/4ef0470c-aab7-4901-819b-8a1c001a9039)


