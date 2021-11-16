# OKD4 кластер в контуре R&D
OKD4/OCP4 кластер, являющийся средой разработки.

## Параметры кластера
* Имя кластера: okd
* Веб консоль: https://console-openshift-console.apps.okd.dev.example.ru/
* DNS domain: dev.example.ru
* Сеть: 10.149.1.0/24, gw: 10.149.1.1
* Доступ к интернету: есть, прямой

## Узлы / VMs
* 3x 4CPU, 16GB RAM, 100GB storage - masters
* 3x 4CPU, 16GB RAM, 100GB storage - workers

| Name | IP |  Notes |
|------|----|--------|
|api.okd.dev.example.ru |10.149.1.203 |внешний Kubernetes API |
|api-int.okd.dev.example.ru |10.149.1.203 |внутренний API |
|*.apps.okd.dev.example.ru |10.149.1.203 |app ingress/routes |
|okd4-lb2.okd.dev.example.ru |10.149.1.203 |loadbalancer |
|okd4-m0.okd.dev.example.ru |10.149.1.205 |OKD4 master 1 |
|okd4-m1.okd.dev.example.ru |10.149.1.206 |OKD4 master 2 |
|okd4-m2.okd.dev.example.ru |10.149.1.207 |OKD4 master 3 |
|okd4-w0.okd.dev.example.ru |10.149.1.208 |OKD4 worker 1 |
|okd4-w1.okd.dev.example.ru |10.149.1.209 |OKD4 worker 2 |
|okd4-w2.okd.dev.example.ru |10.149.1.210 |OKD4 worker 3 |

## Системные зависимости

* Система управления VM - Скала-Р: https://vms.dev.example.ru/
* Сервер управления (bastion): bastion.dev.example.ru
* Load balancer: okd4-lb2.okd.dev.example.ru HAProxy
* LDAP: ldap://ldap.dev.example.ru:389
* NFS: nfs.dev.example.ru