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
