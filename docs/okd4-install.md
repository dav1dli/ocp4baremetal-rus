# Установка OCP4/OKD4
Этот документ является протоколом установки кластера OCP4/OKD4 по методу UPI - user provisioned infrastructure.

Кластер OCP4/OKD4 предназначен для поддержки разработчиков.

[Документация по установке OCP4](https://docs.openshift.com/container-platform/4.8/installing/index.html) 
[Документация по установке OKD4](https://docs.okd.io/latest/installing/index.html)

## Параметры кластера

* Имя кластера: okd
* DNS domain: dev.example.ru
* Pull secret: XXX
* SSH key: XXX
* Сеть: 10.149.1.0/24, gw: 10.149.1.1
* Сеть сервисов: 172.30.0.0/16
* Доступ к интернету: есть, прямой

### Узлы
* 1x 4CPU, 16GB RAM, 100GB storage - bootstrap
* 3x 4CPU, 16GB RAM, 100GB storage - masters
* 3x 4CPU, 16GB RAM, 100GB storage - workers

## Системные зависимости

* Система управления VM - Скала-Р: https://vms.dev.example.ru/ - система виртуализации на основе KVM.
* Сервер управления (bastion) - Линукс сервер, с которого осуществляются действия по установке
* DHCP: 10.149.1.11, 10.149.1.12
* DNS: 10.149.163, 10.149.1.39,  10.149.1.24
* Load balancer: okd4-lb2.okd.dev.example.ru HAProxy
* NTP: по умолчанию

## Подготовка к установке
Кластер устанавливается с минимальным набором машин на системе виртуализации Скала-Р. 
Ввиду отсутствия интеграции с установщиком, выбран метод установки UPI (User provided infrastructure), т.е. все ресурсы создаются вручную перед началом установки кластера.

Выбранный метод инициализации ВМ - загрузка с образа [live CD](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211004.3.1/x86_64/fedora-coreos-34.20211004.3.1-live.x86_64.iso).
 

[Документация](https://docs.okd.io/latest/installing/installing_bare_metal/preparing-to-install-on-bare-metal.html)

### Требования к сети / портам

All to all

|Proto |Port |Notes |
|------|-----|------|
|ICMP | | |
|TCP |1936 | |
|    |9000 - 9999 | |
|	 |10250-10259 | |
|	 |10256 | |
|UDP |4789 | |
|    |6081 | |
|	 |9000 - 99999 | |
|TCP |30000 - 32767 | |

All to masters
TCP 6443

Masters to masters
TCP 2379 - 2380

#### DNS

| Type | Name | IP |  Notes |
|-------|--------|----|-----|
|A |api.okd.dev.example.ru |10.149.1.203 |внешний Kubernetes API |
|A |api-int.okd.dev.example.ru |10.149.1.203 |внутренний API |
|A |*.apps.okd.dev.example.ru |10.149.1.203 |app ingress/routes |
|A |okd4-lb2.okd.dev.example.ru |10.149.1.203 |loadbalancer |
|A |okd4-bs2.okd.dev.example.ru |10.149.1.204 |временный ВМ |
|A |okd4-m0.okd.dev.example.ru |10.149.1.205 |OKD4 master 1 |
|A |okd4-m1.okd.dev.example.ru |10.149.1.206 |OKD4 master 2 |
|A |okd4-m2.okd.dev.example.ru |10.149.1.207 |OKD4 master 3 |
|A |okd4-w0.okd.dev.example.ru |10.149.1.208 |OKD4 worker 1 |
|A |okd4-w1.okd.dev.example.ru |10.149.1.209 |OKD4 worker 2 |
|A |okd4-w2.okd.dev.example.ru |10.149.1.210 |OKD4 worker 3 |

Важно создать как прямые (А) записи, так и обратные (PTR). [Документация](https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal-network-customizations.html#installation-dns-user-infra_installing-bare-metal-network-customizations).

Примеры конфигурации прямой и обратной зон можно найти в files/dev.example.ru.db, files/dev.example.ru.rev.

Проверка:
```
dig +noall +answer api.okd.dev.example.ru
dig +noall +answer api-int.okd.dev.example.ru
dig +noall +answer 123.apps.okd.dev.example.ru
dig +noall +answer console-openshift-console.apps.okd.dev.example.ru
dig +noall +answer okd4-bs2.okd.dev.example.ru
dig +noall +answer -x  10.149.1.203
```

#### DHCP
* Subnet: 10.149.1.0 netmask 255.255.255.0
* Router: 10.149.1.1
* DNS: 10.149.1.24
* Domain search: dev.example.ru

Hosts:
```
host okd4-lb2      { hardware ethernet 00:1c:42:7f:36:88; fixed-address 10.149.1.203; }
host okd4-bs2      { hardware ethernet 00:1c:42:cf:ad:e4; fixed-address 10.149.1.204; }
host okd4-m0       { hardware ethernet 00:1c:42:14:11:1b; fixed-address 10.149.1.205; }
host okd4-m1       { hardware ethernet 00:1c:42:2e:b6:99; fixed-address 10.149.1.206; }
host okd4-m2       { hardware ethernet 00:1c:42:f5:e1:6e; fixed-address 10.149.1.207; }
host okd4-w0       { hardware ethernet 00:1c:42:e1:1d:66; fixed-address 10.149.1.208; }
host okd4-w1       { hardware ethernet 00:1c:42:ed:d2:78; fixed-address 10.149.1.209; }
host okd4-w2       { hardware ethernet 00:1c:42:a3:03:c5; fixed-address 10.149.1.210; }
```

#### Load balancing 
В текущей конфигурации в качестве балансировщика установлен HAProxy на отдельной VM okd4-lb2. Предполагается, что 1 балансировщик сможет управлять сетевым траффиком и для API сервера и для ингрессов приложений.

Для функционирования балансировщика требуется, что бы конфигурация DNS была завершена.

Конфигурация балансировщика: files/haproxy.cfg

### Создание ресурсов

#### Downloads

* [Fedora COS ISO](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211004.3.1/x86_64/fedora-coreos-34.20211004.3.1-live.x86_64.iso)
* [Kernel](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211004.3.1/x86_64/fedora-coreos-34.20211004.3.1-live-kernel-x86_64)
* [initramfs](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211004.3.1/x86_64/fedora-coreos-34.20211004.3.1-live-initramfs.x86_64.img)
* [rootfs](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20211004.3.1/x86_64/fedora-coreos-34.20211004.3.1-live-rootfs.x86_64.img)
* [OC client](https://github.com/openshift/okd/releases/download/4.8.0-0.okd-2021-10-24-061736/openshift-client-linux-4.8.0-0.okd-2021-10-24-061736.tar.gz)
* [OKD installer](https://github.com/openshift/okd/releases/download/4.8.0-0.okd-2021-10-24-061736/openshift-install-linux-4.8.0-0.okd-2021-10-24-061736.tar.gz)

Fedora COS ISO должен быть доступен на гипервизоре во время установки узлов кластера.

ОС CLI и установщик должны быть установлены на бастионе.

#### Серверы / Виртуальные машины

##### Jump host / bastion
Этот хост предназначен для выполнения установки. Он может быть использован для временного запуска сервисов, необходимых для установки. На нем выполняется установка кластера.

Установить зависимости: `dnf install libvirt`
Скачать [OC CLI](https://github.com/openshift/okd/releases/download/4.8.0-0.okd-2021-10-24-061736/openshift-client-linux-4.8.0-0.okd-2021-10-24-061736.tar.gz) и распаковать выполняемый файл в PATH.

Скачать [OKD Installer](https://github.com/openshift/okd/releases/download/4.8.0-0.okd-2021-10-24-061736/openshift-install-linux-4.8.0-0.okd-2021-10-24-061736.tar.gz) и распаковать его там, откуда он может быть исполнен.


###### SSH
По умолчанию и следуя рекомендациям Red Hat SSH доступ на узлы кластера не предоставляется. Но во время установки можно предоставить публичный SSH ключ, который впоследствии можно будет использовать для доступа через пользователя core. 

Ключи сгенерироаны командой `ssh-keygen -N '' -f .ssh/okd4_id_rsa` для установки okd4 и размещены в files/.

###### Pull secret

Pull secret может быть скачан с [сайта поддержки Red Hat](https://console.redhat.com/openshift/install/metal/user-provisioned). Файл скачивается в виде текстового файла и используется во время установки кластера.

###### HTTP
Сервер HTTP используется для передачи файлов во время установки.

* Установить пакет: `dnf install httpd`
* Firewall: `firewall-cmd --zone=public --permanent --add-service=http; firewall-cmd --reload`
* Системные сервисы: `systemctl enable httpd; systemctl start httpd`
* Проверка: `curl http://bastion.dev.example.ru`

После публикации IGN файлов на сервере необходимо сконфигурировать безопасный доступ к ним: `chmod 644 /var/www/html/*; restorecon -RFv /var/www/html/`

##### Loadbalancer

Балансировщик служит для распределения входящего траффика между узлами кластера. Во время установки требуется сконфигурировать балансировку доступа к API Server кластера и к appliction ingress/routes.

Порты: 
* 1936/tcp - status
* 6443/tcp - Kubernetes API
* 22623/tcp - machine config server
* 443/tcp - ingress HTTPS
* 80/tcp - ingress HTTP

Установка:
* создать VM (2 CPU, 4GB RAM, 10GB disk) okd4-lb1, использовать MAC адрес для конфигурации DHCP
* добавить okd4-lb1 в DHCP
* добавить okd4-lb1 в DNS
* установить VM с ISO CentOS8 x86_64 из библиотеки образов
* системные операции: `dnf update`, `dnf install haproxy policycoreutils-python-utils`
* конфигурация балансировщика: files/haproxy.cfg
* Открыть порты в firewall:
```
firewall-cmd --permanent --add-port 1936/tcp
firewall-cmd --permanent --add-port 6443/tcp
firewall-cmd --permanent --add-port 22623/tcp
firewall-cmd --permanent --add-port 443/tcp
firewall-cmd --permanent --add-port=80/tcp
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
* Старт: `service haproxy restart`
* Стартовать автоматически: `chkconfig haproxy on`

## Установка
### Создание конфигурации установки на бастионе

Установка кластера okd4 выполняется с bastion хоста bastion.dev.example.ru, подготовленного согласно инструкциям выше.

* Создать фолдер: `mkdir okd4inst`
* Скопировать конфигурацию инсталлятора: `cp files/install-config.yaml okd4inst` 
* Сгенерировать манифесты: `./openshift-install create manifests --dir=okd4inst`
* Изменить планировщик кластера в  okd4inst/manifests/cluster-scheduler-02-config.yml: `mastersSchedulable: false`
* Сгенерировать IGN файлы: `./openshift-install create ignition-configs --dir=okd4inst`
* Скопировать IGN файлы в http: `cp okd4inst/*.ign /var/www/html/` . Файлы IGN валидны 24 часа, за которые установка должна быть закончена.

URLs:
* http://10.149.1.68/bootstrap.ign
* http://10.149.1.68/master.ign
* http://10.149.1.68/worker.igncoreos

С этого момента можно запускать VM для начала установки кластера.

### Установка Ignition на узлы

#### Интерактивный установщик

VM okd4-bs2 сконфигурирована запуститься с ISO установки FCOS. Запустить VM. Система загрузится в промпт шелла. Выполнить комманду:

`sudo coreos-installer install --ignition-url=http://10.149.1.68/bootstrap.ign /dev/sda --insecure-ignition`

Замечание: т.к. копирование в окно VNC не работает, то вместо набора вручную SHA512 суммы файла bootstrap.ign можно использовать опцию --insecure-ignition.

Повторить для мастеров okd4-m0, okd4-m1, okd4-m2:

`sudo coreos-installer install --ignition-url=http://10.149.1.68/master.ign /dev/sda --insecure-ignition`

И для рабочих узлов okd4-w0, okd4-w1, okd4-w2:

`sudo coreos-installer install --ignition-url=http://10.149.1.68/worker.ign /dev/sda --insecure-ignition`

Перегрузить узлы с установленной системой FCOS, начиная с okd4-bstr, мастера и рабочие узлы.

#### Конфигурация через параметры загрузчика 

Так же возможно выполнить неинтерактивную установку, сконфигурировав ее через параметры ядра. Для этого загрузка системы прерывается на стадии загрузчика (grub) и ядру добавляются параметры:
```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://10.149.1.68//fedora-coreos.raw.xz
coreos.inst.ignition_url=http://10.149.1.68/bootstrap.ign
coreos.inst.insecure
rd.neednet=1
ip=ens3:dhcp
ip=10.149.1.186::10.149.1.1:255.255.255.0:okd4-mstr3.okd4.dev.example.ru:ens3:none nameserver=10.149.1.24
```
если DHCP не используется, то сеть можно определить статически:
```
coreos.inst=yes
coreos.inst.install_dev=sda
coreos.inst.image_url=http://10.149.1.68//fedora-coreos.raw.xz
coreos.inst.ignition_url=http://10.149.1.68/bootstrap.ign
coreos.inst.insecure
rd.neednet=1
console=tty0 console=ttyS0,115200n8
ip=10.149.1.205::10.149.1.1:255.255.255.0:okd4-m0.okd.dev.example.ru:ens3:none nameserver=10.149.1.24
```

#### Решение проблем установки

Установка по умолчанию не ведет к рабочей конфигурации. Все узлы кроме бутстрапа не могут пройти процесс инициализации из-за недоступности сети. 
Для обхода этой проблемы можно использовать версию Fedora COS старше, чем та, в которой эта проблема появилась: [34.20210611.3.0](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210626.3.0/x86_64/fedora-coreos-34.20210626.3.0-metal.x86_64.raw.xz). 

### Завершение установки
Когда ВМы запущены с соответствующими стартовыми параметрами они проходят процесс установки (bootstrap) и в конце концов формируют кластер.

За процессом можно следить с бастиона командой: `./openshift-install --dir=okd4inst wait-for bootstrap-complete --log-level=info`
```
INFO Waiting up to 20m0s for the Kubernetes API at https://api.okd.dev.example.ru:6443...
INFO API v1.22.0-rc.0+894a78b up
INFO Waiting up to 30m0s for bootstrapping to complete...
INFO It is now safe to remove the bootstrap resources
INFO Time elapsed: 34s
```
На этом этапе следует убрать okd4-bs2 из конфигурации балансировщика /etc/haproxy/haproxy.cfg. Перезапустить балансировшик: `systemctl restart haproxy`

Удалить VM okd4-bs2.

Проверить кластер:
```
export KUBECONFIG=`pwd`/okd4inst/auth/kubeconfig
oc whoami
oc get nodes
oc get clusterversion
```

```
NAME      VERSION                         AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.8.0-0.okd-2021-10-01-221835   True        False         102m    Cluster version is 4.8.0-0.okd-2021-10-01-221835
```

### Присоединить рабочие узлы к кластеру
Рабочие узлы создают CSRs с запросом присоединиться к кластеру. Эти запросы следует рассмотреть `oc get csr`

и если они валидны - подтвердить: `oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve`
```
NAME                                 STATUS   ROLES    AGE     VERSION
okd4-m0   Ready    master   79m   v1.22.1+8719299-1678
okd4-m1   Ready    master   81m   v1.22.1+8719299-1678
okd4-m2   Ready    master   85m   v1.22.1+8719299-1678
okd4-w0   Ready    worker   32m   v1.22.1+8719299-1678
okd4-w1   Ready    worker   32m   v1.22.1+8719299-1678
okd4-w2   Ready    worker   32m   v1.22.1+8719299-1678
```

### Веб консоль

Веб консоль: https://console-openshift-console.apps.okd.dev.example.ru/

После установки кластера доступен только юзер kubeadmin с паролем из okd4inst/auth/kubeadmin-password.
