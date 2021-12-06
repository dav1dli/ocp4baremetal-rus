# Конфигурация OCP4/OKD4

Этот документ является протоколом кастомизаций кластера OCP4/OKD4 в контуре R&D проекта ОКП.

## ETCD backup

[Бэкап создан](https://docs.okd.io/latest/post_installation_configuration/cluster-tasks.html#backing-up-etcd-data_post-install-cluster-tasks) на admin@bastion:~/okd4-etcd-backup_2021-10-29_134253.tar.gz.

Из него кластер может быть [восстановлен](https://docs.okd.io/latest/post_installation_configuration/cluster-tasks.html#dr-scenario-2-restoring-cluster-state_post-install-cluster-tasks) в изначальное состояние.

## Хранение данных

Openshift использует Persistent Volume (PV) фреймворк для управления данными, требующими сохранности. 
Админы кластера создают PV, а пользователи на уровне проектов могут создавать Persistent Volume Claims (PVCs) для запроса томов хранения данных.

### NFS Server

Для хранения данных используется серевер nfs.dev.example.ru, поддерживающий NFS.

Статус: `systemctl status nfs-server`

* диск: /dev/sdb1 XFS 100GB
* mount point: /srv/nfs
* shares: 
 * /srv/nfs/okd4/usrvol/pv{1..50} -> /etc/exports.d/okd4-usrvol.exports
 * /srv/nfs/okd4/sysvol/registry -> /etc/exports.d/okd4-sysvol.exports

NFS config: 
```
mkdir -p /srv/nfs/okd4/sysvol/registry
chown -R nobody:nobody /srv/nfs
chmod -R 777 /srv/nfs/okd4/sysvol
echo "/srv/nfs/okd4/sysvol/registry *(rw,sync,no_wdelay,no_root_squash,insecure,fsid=0)" > /etc/exports.d/okd4-sysvol.exports
exportfs -rav
systemctl reload-or-restart nfs-server
```
После конфигурации NFS сервера его необходимо перестартовать.

## Хранение данных для пользователей

Для данных разработки/приложений созданы 50 томов на NFS сервере, которые могут одновременно использоваться. 

На стороне кластера это обеспечивается определением Persistent Volume (PV), как показано в files/nfs-pv.yaml.

Проверка: `oc get pv`

На уровне приложений создается Persistent Volume Claim (PVC), который использует доступный PV, как показано в files/nfs-pv.yaml.

## Image registry

В процессе установки кластера Image registry, имеющий требования по хранению данных, установлен в "Removed".
После завершения установки необходимо сконфигурировать Image Registry в соответствии с требованиями установки.

Removed -> Managed: `oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'`

Storage: `oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'`

Status: `oc get clusteroperator image-registry`

Operator config: `oc describe configs.imageregistry.operator.openshift.io`

ToDo: переключиться на использование NFS томов. См. файл files/registry-pv.yaml.

## Insecure registries

Каталог контейнеров: http://nexus.dev.example.ru:60001 - каталог сконфигурированный использовать http, а не https протокол.
В этом случае при попытке скачать образ будет выдана ошибка "http: server gave HTTP response to HTTPS client". Для решения этой проблемы следует сконфигурировать каталог с поддержкой https.
Если это невозможно, то можно [включить каталог в список допустимых незащищенных каталогов](https://computingforgeeks.com/allow-insecure-registries-in-openshift-okd-4-cluster/).

CLI: `oc edit image.config.openshift.io/cluster`
```
spec:
  registrySources:
    insecureRegistries:
      - 'nexus.dev.example.ru:60001'
```

Web Console: Adminstration -> Cluster Settings -> Image
```
spec:
  registrySources:
    insecureRegistries:
      - 'nexus.dev.example.ru:60001'
```

## LDAP

LDAP:
* server: ldaps://ldap.dev.example.ru:636, ldap://ldap.dev.example.ru:389
* URL: ldap://127.0.0.1:1389/dc=dev,dc=example,dc=ru?cn?sub?(&(objectClass=inetOrgPerson)(isMemberOf=cn=OpenShiftUsers,ou=DEV,dc=dev,dc=example,dc=ru)(!(isMemberOf=cn=LockedUsers,ou=DEV,dc=dev,dc=example,dc=ru))(!(isMemberOf=cn=PwdAccountLocked,ou=DEV,dc=dev,dc=example,dc=ru))(!(isMemberOf=cn=PEW,ou=DEV,dc=dev,dc=example,dc=ru)))
* bindDN: cn=openshift,ou=DEV,dc=dev,dc=example,dc=ru

Test: `ldapsearch -x -b "dc=dev,dc=example,dc=ru" -H ldap://ldap.dev.example.ru -D cn=openshift,ou=DEV,dc=dev,dc=example,dc=ru -W "objectclass=account"  cn uid displayName`

Create LDAP secret with bindPassword: `oc create secret generic ldap-bind-password-t9hd5 --from-literal=bindPassword=*** -n openshift-config`

OAuth config files/ldap-cr.yaml: `oc apply -f ldap-cr.yaml`

Создать группу files/cluster-admin-group.yaml: `oc apply -f cluster-admin-group.yaml`

Создать роль админов files/role-for-cluster-admin-group.yaml: `oc apply -f role-for-cluster-admin-group.yaml`

Проверка: `oc login -u admin --server=https://api.okd.dev.example.ru:6443`

Ожидаемый результат: доступ ко всем проектам.

## Синхронизация групп LDAP
Наряду с аутентикацией пользователей из LDAP, группы пользователей, управляемые в LDAP, тоже можно [синхронизировать](https://docs.okd.io/latest/authentication/ldap-syncing.html) в OKD.

Для синхронизации требуется конфигурационный файл.

* url: ldap://ldap.dev.example.ru:389 
* bindDN: cn=openshift,ou=DEV,dc=dev,dc=example,dc=ru
* bindPassword: passw0rd
* insecure: true
* ca: n/a (insecure=yes)

  
LDAP sync config: files/ldap-sync-cfg.yaml

Groups whitelist: files/openshift-groups.txt - синхронизация части групп LDAP в Openshift

Синхронизация: `oc adm groups sync --type=ldap --sync-config=files/ldap-sync-cfg.yaml --whitelist=files/openshift-groups.txt --confirm`

Чистка групп: `oc adm prune groups --sync-config=files/ldap-sync-cfg.yaml --whitelist=files/openshift-groups.txt` 

### Автоматическая синхронизация - cron job
* роль cluster-admin
* сконфигурированный LDAP IDP
* LDAP secret
* проект ldap-sync
* расписание: ежечасно

* Создать проект: `oc new-project ldap-sync`
* Создать service account: `oc create -f files/ldap-sync-sa.yaml`
* Создать секрет: `oc create -f files/ldap-sync-secret.yaml`
* Создать роль: `oc create -f files/ldap-sync-role.yaml`
* Выдать роль SA: `oc create -f files/ldap-sync-rolebnd.yaml`
* Создать конфигурацию cron job: `oc create -f files/ldap-sync-configmap.yaml`
* Создать cron job: `oc create -f files/ldap-sync-job.yaml`

## Отмена локального админ kubeadmin

После того, как LDAP сконфигурирован и административные роли выданы, локальный админ может быть отключен: `oc delete secrets kubeadmin -n kube-system`

## Отмена возможности создания проекта пользователями

По умолчанию Openshift позволяет аутентифицированным пользователям создавать проекты. 
Что-бы отменить эту опцию:
```
oc patch clusterrolebinding.rbac self-provisioners --type=merge \
  --patch '{"metadata":{"annotations":{"rbac.authorization.kubernetes.io/autoupdate": "false"}}}'
oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```

Чтобы разрешить создание проектов на уровне группы пользователей: `oc adm policy add-cluster-role-to-group self-provisioner power-users-group`

## Синхронизация LDAP групп

https://docs.okd.io/latest/authentication/ldap-syncing.html 

## Ограничения на проекты

[Документация](https://docs.openshift.com/container-platform/4.8/applications/projects/configuring-project-creation.html)

По умолчанию контейнеры исполняются без ограничений на потребляемые ресурсы. Это может привести к исчерпанию системных ресурсов и к деградации производительности кластера.
Используя Limit Ranges можно ограничить потребление ресурсов, таких как CPU.

Создать шаблон проекта: `oc adm create-bootstrap-project-template -o yaml > bootstrap-project-template.yaml`

LimitRange определяет значения по умолчанию, которые присваиваются создаваемым контейнерам в случае, если они сами не определяют этизначения:
* запрос: 1/2 CPU, 500Mi памяти
* максинум: 1 CPU, 1Gi памяти

ResourceQuota определяет сумму ресурсов, которые могут занять объекты в проекте: 
* запрос: 10 подов, 4 CPU, 8Gi памяти
* максинум: 6 CPU, 16Gi памяти

Создать новый шаблон проекта: `oc apply -f bootstrap-project-template.yaml -n openshift-config`

Применить шаблон к кластеру: 
```
oc patch project.config.openshift.io/cluster \
    --type=merge \
    -p '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'
```

Проекты omni, omni-common, payhub, omni-stage имеют свои собственные лимиты. Они были установлены из files/core-resource-limits.yaml.

## Интеграция с GitLab

OKD4 предоставляет в OperatorHub GitLab Runner Operator. Оператор - это способ упаковки и управления приложенем Openshift/Kubernetes. В дополнение к базовым ресурсам k8s операторы создают Custom Resources приложения, которыми оператор управляет.

В OKD4 операторы можно устанавливать как через веб консоль, так и с использованием cli. [GitLab документ](https://docs.gitlab.com/runner/install/openshift.html), описывающий установку GitLab Runner. 

После установки оператора требуется создать инстанс раннера в выбранном проекте. Это можно сделать как описано в документе выше или через веб консоль.

Параметры:
* проект: gitlab-test
* URL: http://gitlab.dev.example.ru/
* Token: 9Sd-WnA*****
* Tags: openshift, build, test

Сконфигурировать GitLab pipeline исполняться на раннере: внести 'tags: openshift' в те джобы пайплайна, которые должны исполняться на раннере с тагом openshift. 

Пример конфигурации пайплайна: files/gitlab-ci.yml


## Интеграция с TeamCity

В текущей версии TeamCity включен плагин интеграции с Kubernetes, который позволяет определить k8s кластер в качестве облачного профиля. Облачный провайдер может быть определен на уровне проекта в ТС. Для этого требуются права администратора проекта.

Параметры облачного профиля:
* name: okd4
* type: Kubernetes
* Server URL: http://teamcity.dev.example.ru/
* Kubernetes API: https://api.okd.dev.example.ru:6443
* namespace: tc-okd4
* Authentication strategy: token

Token принадлежит Service Account teamcity, который необходимо создать предварительно:
* service account: files/sa-teamcity.yaml
* role: files/sa-teamcity-role.yaml
* role binding: files/sa-teamcity-rb.yaml

## Инфраструктурные узлы
На кластерах, обслуживающих реальные рагрузки рекомендуется выносить системные компоненты, такие как 
routes, ingresses, image registry, metrics, logging, gitops, monitoring на отдельный набор инфраструктурных узлов. Рекомендованная конфигурация: 
минимум 3 узла, разпределенные между узлами инфраструктуры и географически разнесенными зонами доступности.

На гипервизоре создать 3 VM на разных серверах:  3x 4CPU, 16GB RAM, 200GB storage, DVD: fedora-coreos.iso

Добавить хосты на DHCP /etc/dhcp/fixed-addresses.conf:
* okd4-i0 { hardware ethernet 00:1c:42:71:cb:77; fixed-address 10.149.1.211; }
* okd4-i1 { hardware ethernet 00:1c:42:14:73:29; fixed-address 10.149.1.212; }
* okd4-i2 { hardware ethernet 00:1c:42:0e:ff:ad; fixed-address 10.149.1.213; }

Для операции требуется root. 

DNS:
* okd4-i0.okd4.dev.example.ru. 10.149.1.211
* okd4-i1.okd4.dev.example.ru. 10.149.1.212
* okd4-i2.okd4.dev.example.ru. 10.149.1.213

Для операции требуется root. На сервере dhcp1 есть скрипт /opt/libexec/nsupdate.sh, позволяющий добавлять хосты в DNS домен dev.example.ru.
Синтаксис команд находится в файлах  nsupdate.A, nsupdate.PTR.

Igntion: http://10.149.1.68/worker.ign

Установить как новые VMs как цорерабочие узлы:
```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://10.149.1.68/fedora-coreos.raw.xz
coreos.inst.ignition_url=http://10.149.1.68/worker.ign
coreos.inst.insecure
rd.neednet=1
ip=ens3:dhcp
console=tty0 
console=ttyS0
```

Присоединить новые узлы: 
`oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve`

Поменять метки узлов:
```
oc label node okd4-i0 node-role.kubernetes.io/infra=
oc label node okd4-i1 node-role.kubernetes.io/infra=
oc label node okd4-i2 node-role.kubernetes.io/infra=
oc label node okd4-i0 node-role.kubernetes.io/worker-
oc label node okd4-i1 node-role.kubernetes.io/worker-
oc label node okd4-i2 node-role.kubernetes.io/worker-
```

Создать Machine Config Pool:
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: infra
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - compute
  maxUnavailable: 1
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/infra: ""
  paused: false
```
oc create -f infra-machineconfigpool.yaml

Проверка: `oc get nodes`

```
NAME      STATUS   ROLES    AGE     VERSION
okd4-i0   Ready    infra    6m56s   v1.21.2+6438632-1505
okd4-i1   Ready    infra    6m59s   v1.21.2+6438632-1505
okd4-i2   Ready    infra    6m59s   v1.21.2+6438632-1505
okd4-m0   Ready    master   25d     v1.21.2+6438632-1505
okd4-m1   Ready    master   25d     v1.21.2+6438632-1505
okd4-m2   Ready    master   25d     v1.21.2+6438632-1505
okd4-w0   Ready    worker   25d     v1.21.2+6438632-1505
okd4-w1   Ready    worker   25d     v1.21.2+6438632-1505
okd4-w2   Ready    worker   25d     v1.21.2+6438632-1505
```

Проверка: `oc get mcp`
```
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
infra    rendered-infra-656f113274fffc87b64ca7e8f007d5e9    True      False      False      3              3                   3                     0                      107s
master   rendered-master-8d0f1b45c520f70300dbfe40b8d92d58   True      False      False      3              3                   3                     0                      25d
worker   rendered-worker-656f113274fffc87b64ca7e8f007d5e9   False     True       False      6              3                   3                     0                      25d
```

Изменить defaultNodeSelector: `oc patch scheduler cluster --type=merge -p '{"spec":{"defaultNodeSelector":"node-role.kubernetes.io/app="}}'`

## Рауты на инфра узлах
Перенести рауты по умолчанию на инфра узлы: 
```
oc patch ingresscontrollers.operator.openshift.io default -n openshift-ingress-operator --type=merge \
--patch '{"spec":{"nodePlacement":{"nodeSelector":{"matchLabels":{"node-role.kubernetes.io/infra":""}}}}}'
```
По числу инфра узлов создать 3 реплики раутов:
```
oc patch ingresscontrollers.operator.openshift.io default -n openshift-ingress-operator --type=merge \
--patch '{"spec":{"replicas": 3}}'
```

Переконфигурировать балансировщик на инфра узлы: на okd4-lb2 в /etc/haproxy/haproxy.cfg:
```
frontend router_https
    bind 0.0.0.0:443
    default_backend router_https
    mode tcp
    option tcplog

backend router_https
    balance source
    mode tcp
    server infra0 okd4-i0.okd4.dev.example.ru:443 check
    server infra1 okd4-i1.okd4.dev.example.ru:443 check
    server infra2 okd4-i2.okd4.dev.example.ru:443 check

frontend router_http
    bind 0.0.0.0:80
    default_backend router_http
    mode tcp
    option tcplog

backend router_http
    server infra0 okd4-i0.okd4.dev.example.ru:80 check
    server infra1 okd4-i1.okd4.dev.example.ru:80 check
    server infra2 okd4-i2.okd4.dev.example.ru:80 check
```
Перестартовать сервис: `service haproxy restart`

Проверить доступность консоли на https://console-openshift-console.apps.okd.dev.example.ru/

## Встроенный каталог образов на инфра узлах
```
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge \
  --patch '{"spec":{"nodeSelector":{"node-role.kubernetes.io/infra": ""}}}'
oc patch configs.imageregistry,operator.openshift.io/cluster --type merge \
  --patch '{{"spec": {"defaultRoute": true}}'
```

## Local volumes
Поизводительность и надежность, предоставляемые самодельным NFS сервером могут быть недостаточными для компонентов,
обрабатывающих большие объемы данных, поступающих с высокой скоростью, таким как мотиторинг и централизованные логи.
В условиях отсутствия сетевого хранилища данных, способного удовлетворить эти требования, существует возможность 
[использования локальных томов](https://docs.okd.io/latest/storage/persistent_storage/persistent-storage-local.html) (local volumes).
Смысл заключается в том, что Local Storage Operator предоставляет доступ к дискам и файловым системам на узле. 
Такой подход удовлетворяет потребности по стабильности и производительности, но затрудняет скалирование и выживаемость компонентов, 
полагающихся на такой тип хранения данных. Альтернативой может быть интеграция с СХД, способной предоставлять тома подам в Openshift или 
создание собственного решения, типа Ceph кластера СХД. 

Несмотря на то, что написано в дoкументации, в текущей версии OKD4 Local Storage Operator недоступен в OperatorHub.
Есть возможность установить оператор из [репозитория проекта](https://github.com/openshift/local-storage-operator), следуя
[инструкциям](https://github.com/openshift/local-storage-operator/blob/master/docs/deploy-with-olm.md).

Т.к. это решение призвано удовлетворить потребности инфраструктурных компонентов, то локальные тома реализованы на инфра узлах.

Для этого требуется на уровне инфраструктуры добавить по 500ГБ диску узлам okd4-i0, okd4-i1, okd4-i2. 
Для того, что бы была возможность увеличивать тома без их пересоздания, на добавленных дисках созданы логические тома с использованием LVM.
Доступ к хостам осуществляется через SSH: ` ssh -i okd4_id_rsa core@okd4-i0.okd.dev.example.ru`.

### На каждом хосте
```
pvcreate /dev/sdb
vgcreate vg_locvol /dev/sdb
lvcreate -L 250G -n lv_es vg_locvol
lvcreate -L 100G  -n lv_prom vg_locvol
lvcreate -L 50G  -n lv_alert vg_locvol
```
Созданные устройства: 
* /dev/mapper/vg_locvol-lv_alert - алерты - 50GB
* /dev/mapper/vg_locvol-lv_es - Elasticsearch - 250GB
* /dev/mapper/vg_locvol-lv_prom - Prometheus - 100GB

### Кластер

* Создать проект: `oc adm new-project openshift-local-storage`
* Установить оператор: `oc apply -f https://raw.githubusercontent.com/openshift/local-storage-operator/master/examples/olm/catalog-create-subscribe.yaml`
* Создать тома и классы томов (storageclass):
```
oc create -f localstor-crd-es.yaml
oc create -f localstor-crd-prom.yaml
oc create -f localstor-crd-alert.yaml
```

Проверка: `oc get pv`
```
local-pv-11643fb5   100Gi      RWO            Delete           Available   localvol-sc-prom             94m
local-pv-33825d56   250Gi      RWO            Delete           Available   localvol-sc-es               103m
local-pv-401f48e6   100Gi      RWO            Delete           Available   localvol-sc-prom             94m
local-pv-49fa7e8e   50Gi       RWO            Delete           Available   localvol-sc-alert            92m
local-pv-6ee973c9   250Gi      RWO            Delete           Available   localvol-sc-es               103m
local-pv-8fe18fa1   50Gi       RWO            Delete           Available   localvol-sc-alert            92m
local-pv-a51c8183   50Gi       RWO            Delete           Available   localvol-sc-alert            92m
local-pv-bf89516f   250Gi      RWO            Delete           Available   localvol-sc-es               103m
local-pv-cd652af4   100Gi      RWO            Delete           Available   localvol-sc-prom             94m
```
Создано по 3 тома на каждый тип системного компонента для обеспечения работы 3х реплик каждого.

## Elasticsearch
Оператор Elasticsearch в данный момент недоступен в OperatorHub.


## Логи
Установка по умолчанию зависит от Elasticsearch поэтому функцияй в данный момент недоступна.

## Мониторинг
Мониторинг конфигурируется через configMap openshift-monitoring/cluster-monitoring-config: `oc apply -f monitoring-configmap.yaml`

Интерфеисы:
* [Prometheus dashboard](https://prometheus-k8s-openshift-monitoring.apps.okd.dev.example.ru/)
* [Grafana](https://grafana-openshift-monitoring.apps.okd.dev.example.ru/)
* [Alert Manager](https://alertmanager-main-openshift-monitoring.apps.okd.dev.example.ru/)

## Высокодоступный балансировщик
Архитектура кластера включает в себя балансировщик HAProxy, обеспечивающий распределение входящего траффика между несколькими копиями API сервера и раутов приложений.
Однако, сам балансировщик создан в виде единственной копии и т.о. представляет собой слабое звено в в общем высокодоступной системе.
Для повышения доступности применена высокодоступная конфигурация из 2х балансировщиков, работающих в активном-пассивном режиме. Входной виртуальный адрес находится на активном балансировщике. 
Виртуальный адрес управляется сервисом keepalived.

```
        okd4-lb
     +-----+------+------+-okd4-m0
     |            |      |
+--------+   +--------+  +-okd4-m1
|okd4-lb1|   |okd4-lb2|  |
+--------+   +--------+  +-okd4-m2
```

Для функционирования балансировщика требуется, что бы конфигурация DNS была завершена.

Конфигурация балансировщика: files/haproxy.cfg

Выполнить на обоих балансировщиках:

Порты: 
* 1936/tcp - status
* 6443/tcp - Kubernetes API
* 22623/tcp - machine config server
* 443/tcp - ingress HTTPS
* 80/tcp - ingress HTTP

Установка:
* создать 2 VM (2 CPU, 4GB RAM, 10GB disk) okd4-lb1, okd4-lb2; 
* добавить okd4-lb, okd4-lb1, okd4-lb2 в DHCP
* добавить okd4-lb, okd4-lb1, okd4-lb2 в DNS

```
server 10.149.1.22
add okd4-lb.dev.example.ru. 3600 A 10.149.1.5
add okd4-lb1.dev.example.ru. 3600 A 10.149.1.180
add 5.1.149.10.in-addr.arpa. 3600 PTR okd4-lb.dev.example.ru.
add 180.1.149.10.in-addr.arpa. 3600 PTR okd4-lb1.dev.example.ru.
send
quit
```

* установить VM с ISO CentOS8 x86_64 из библиотеки образов
* системные операции: `dnf update`, `dnf install haproxy policycoreutils-python-utils keepalived psmisc`
* конфигурация балансировщика: files/haproxy.cfg
* Открыть порты в firewall:

```
firewall-cmd --permanent --add-port 1936/tcp
firewall-cmd --permanent --add-port 6443/tcp
firewall-cmd --permanent --add-port 22623/tcp
firewall-cmd --permanent --add-port 443/tcp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
firewall-cmd --reload
```

* Fix SELinux: 

```
semanage port  -a 22623 -t http_port_t -p tcp
semanage port -a 6443 -t http_port_t -p tcp
semanage port -a 1936 -t http_port_t -p tcp
semanage port -a 80 -t http_port_t -p tcp
semanage port -a 443 -t http_port_t -p tcp
setsebool -P haproxy_connect_any 1
```

* Старт: `systemctl start haproxy`
* Стартовать автоматически: `systemctl enable haproxy`

Keepalived:

/etc/sysctl.conf:

```
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
```

Restart: `sysctl -p`

okd4-lb2 /etc/keepalived/keepalived.conf:

```
vrrp_script chk_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}
vrrp_instance VI_1 {
    state MASTER
    interface ens3
    virtual_router_id 5
    priority 101
    virtual_ipaddress {
        10.149.1.5
    }
	track_script {
        chk_haproxy
    }
}
```

okd4-lb1 /etc/keepalived/keepalived.conf:

```
vrrp_script chk_haproxy {
    script "killall -0 haproxy" # check the haproxy process
    interval 2 # every 2 seconds
    weight 2 # add 2 points if OK
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens3
    virtual_router_id 5
    priority 100
    virtual_ipaddress {
        10.149.1.5
    }
	track_script {
        chk_haproxy
    }
}
```

На обеих узлах:

```
systemctl start keepalived
systemctl enable keepalived
```

Проверить на мастере (okd4-lb2): `ip add show`

```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:1c:42:7f:36:88 brd ff:ff:ff:ff:ff:ff
    inet 10.149.1.203/24 brd 10.149.1.255 scope global dynamic noprefixroute ens3
       valid_lft 1743sec preferred_lft 1743sec
    inet 10.149.1.5/32 scope global ens3
       valid_lft forever preferred_lft forever
```

Проверить failover: 
* на мастере (okd4-lb2) `systemctl stop haproxy`
* на бэкапе (okd4-lb1) `ip add show` VIP должен переключиться

Отредактировать DNS:
| Type | Name | IP |  Notes |
|-------|--------|----|-----|
|A |api.okd.dev.example.ru |10.149.1.5 |внешний Kubernetes API |
|A |api-int.okd.dev.example.ru |10.149.1.5 |внутренний API |
|A |*.apps.okd.dev.example.ru |10.149.1.5 |app ingress/routes |

```
server 10.149.1.22
delete api.okd.dev.example.ru. A
delete api-int.okd.dev.example.ru. A
delete *.apps.okd.dev.example.ru. A
add api.okd.dev.example.ru. 3600 A 10.149.1.5
add api-int.okd.dev.example.ru. 3600 A 10.149.1.5
add *.apps.okd.dev.example.ru. 3600 A 10.149.1.5
send
quit
```

Проверить доступ к консоли: https://console-openshift-console.apps.okd.dev.example.ru/