# Использование OKD4 кластера

В этом документе предоставляется общая информация по кластеру OKD4 и по его использованию.

Кластер OKD4 является средой разработки, совместимой с кластерами Openshift 4, предоставляемыми компанией.
OKD является проектом разработки с участием сообщества разработчиков, из которого Openshift выпускается в качестве коммерческого и поддерживаемого продукта.
Установленная версия OKD4.8, что соответствует версии OCP4.8.

## Инструменты
* [ОС CLI](https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/)
* [Windows OC CLI](https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/windows/oc.zip)
* [Helm3](https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/)

## Доступ
Доступ к кластеру возможен только из сети компании и через VPN.

Доступ предоставляется после аутентификации с использованием имен пользователей из LDAP, т.е. обычных логинов, используемых для доступа к другим системам. 
Дополнительно доступ из CLI можно осуществлять по токену (см. раздел Веб консоль)

### Веб консоль
Консоль: https://console-openshift-console.apps.okd.dev.example.ru/

[Консоль](https://docs.okd.io/latest/web_console/web-console.html) - это веб интерфейс, предоставляющий доступ к основным функциям управления кластером и приложениями, в зависимости от предоставленного уровня доступа.

Меню консоли слева разбита на несколько логических секций:
* Home - секция, где можно выбрать [рабочие проекты](https://docs.okd.io/latest/applications/projects/working-with-projects.html), к которым пользователю предоставлен доступ и в рамках которых можно [создавать приложения](https://docs.okd.io/latest/applications/creating_applications/odc-creating-applications-using-developer-perspective.html)
* Workloads - раздел управления основными свойствами приложения, такими как pods, [deployments](https://docs.okd.io/latest/applications/deployments/what-deployments-are.html), secrets, [configmaps](https://docs.okd.io/latest/applications/config-maps.html)
* Networking - раздел управления сетевыми объектами приложения, такими как services, ingresses/routes
* [Storage](https://docs.okd.io/latest/storage/index.html) - раздел управления томами хранения данных приложений 
* Builds  - раздел управления [встроенными билдами](https://docs.okd.io/latest/cicd/builds/understanding-image-builds.html), поддерживаемыми платформой OKD4/OCP4

#### Консоль разработчика
В верхнем левом углу можно переключить консоль с административной на [консоль разработчика](https://docs.okd.io/latest/web_console/odc-about-developer-perspective.html).
Консоль разработчика предоставляет перспективу более ориентированную на задачи разработчика, такие как создание приложений, графическая репрезентация объектов приложения в проекте.
#### Quickstarts
Quickstarts: https://console-openshift-console.apps.okd.dev.example.ru/quickstart
[Quickstarts](https://docs.okd.io/latest/web_console/creating-quick-start-tutorials.html) помогают разработчикам ознакомиться с платформой посредством выполнения заданий.

#### Строка логина / токен
В меню пользователя в верхнем правом углу доступна опция "Copy login command". Она выдает полную строку вызова утилиты коммандной строки ОС, 
включая токен [аутентификации](https://docs.okd.io/latest/authentication/understanding-authentication.html#understanding-authentication) и выглядит как 
```
oc login --token=sha256~XXXXX --server=https://api.okd.dev.example.ru:6443
```
### CLI - коммандная строка
[Openshift CLI](https://docs.okd.io/latest/cli_reference/openshift_cli/getting-started-cli.html) - OC - инструмент коммандной строки, который позволяет взаимодействовать с функциями кластера ОКД, администрировать его и манипулировать приложениями на нем.
OC является расширением утилиты [kubectl](https://docs.okd.io/latest/cli_reference/openshift_cli/usage-oc-kubectl.html) с поддержкой особенностей Openshift.

OC позволяет [создавать](https://docs.okd.io/latest/applications/creating_applications/creating-applications-using-cli.html), обновлять, просматривать, смотреть логи, [редактировать](https://docs.okd.io/latest/applications/odc-editing-applications.html) и удалять объекты приложений на OKD4 кластере. ОС имеет встроенную подсказку. См. [документацию](https://docs.okd.io/latest/cli_reference/openshift_cli/developer-cli-commands.html).

OKD4 поддерживает дополнительные [утилиты CLI](https://docs.okd.io/latest/cli_reference/index.html):
* [Developer CLI - odo](https://docs.okd.io/latest/cli_reference/developer_cli_odo/understanding-odo.html)
* [Knative CLI - kn](https://docs.okd.io/latest/cli_reference/kn-cli-tools.html) - [serverless](https://docs.openshift.com/container-platform/4.8/serverless/serverless-getting-started.html)
* Helm3 - управление [helm charts](https://docs.okd.io/latest/applications/working_with_helm_charts/understanding-helm.html)

## Типовой сценарий
Базисной единицей управления приложением является контрейнер, в который приложение упаковано и который опубликован в каталог (container repository).
Приложения на OKD4 описываются набором объектов:
* [pod](https://docs.okd.io/latest/rest_api/workloads_apis/pod-core-v1.html) - один или более контейнеров с общими ресурсами, который может исполняться на кластере
* [configmap](https://docs.okd.io/latest/rest_api/metadata_apis/configmap-core-v1.html) - [содержит данные](https://docs.okd.io/latest/nodes/pods/nodes-pods-configmaps.html), используемые в подах в виде переменных окружения или файлов
* [secret](https://docs.okd.io/latest/rest_api/security_apis/secret-core-v1.html) - [содержит секретные данные](https://docs.okd.io/latest/nodes/pods/nodes-pods-secrets.html)
* [deployment](https://docs.okd.io/latest/rest_api/workloads_apis/deployment-apps-v1.html) - [описывает приложение](https://docs.okd.io/latest/applications/deployments/what-deployments-are.html), из каких подов оно состоит и сколько реплик должно функционировать
* [service](https://docs.okd.io/latest/rest_api/network_apis/service-core-v1.html) - абстрактный прокси, предоставляющий доступ к ассоциированным с ним подам приложения
* [ingress](https://docs.okd.io/latest/rest_api/network_apis/ingress-networking-k8s-io-v1.html), [route](https://docs.okd.io/latest/rest_api/network_apis/route-route-openshift-io-v1.html) - методы доступа к приложению из-за границ кластера

Все эти объекты приложения описываются декларативно в манифестах. Манифесты могут быть в YAML или JSON форматах. 
Манифесты могут быть переданы утилите ОС для создания или обновления объектов на кластере. 
"Декларативно" означает, что приложение описано, как оно должно быть в манифестах, манифесты загружены на платформу/кластер и платформа сама предпринимает усилия по приведении приложения к описанному виду, включая случаи, когда состояние приложения не соответствует задекларированному вследствии каких-то непредвиденных событий в системе. 
Если платформа не в состоянии привести элементы приложения к декларации, она сообщает об ошибке.

Т.о. типовой сценарий создания приложения на платформе OKD4 будет следующим:
* завершить итеррацию разработки кода приложения, построить контейнер, опубликовать контейнер в доступный каталог
* создать манифесты определяющие объекты приложения
* аутентифицироваться на платформе: `oc login --server=https://api.okd.dev.example.ru:6443`
* выбрать проект: `oc project <PROJECT>`
* загрузить манифесты: `oc apply -f pod.yaml`
* просмотреть объекты: `oc get pods; oc get all`
* просмотреть детали выбранного объекта: `oc get pod my-pod-XYZA - o yaml`
* просмотреть более детальную информацию, включая события, об объекте: `oc describe pod my-pod-XYZA` 
* просмотреть логи пода: `oc logs my-pod-XYZA`
* открыть шелл на под: `oc rsh my-pod-XYZA`
* получить имя раута (если создан) для захода на приложения через браузер: `oc get routes`. URL будет обычно выглядеть как https://APP-PROJECT.apps.okd.dev.example.ru
* удалить объект: `oc delete pod my-pod-XYZA`

## Helm
Манифесты приложения являются декларативными, включая данные.
Для выделения данных и применения шаблонов и програмной логики к манифестам используется [Helm](https://helm.sh/) и [Helm Charts](https://artifacthub.io/).
* [Helm Charts](https://helm.sh/docs/topics/charts/) - набор файлов, описывающих ресурсы OKD4
* Helm v3 - утилита коммандной строки, управляющая чартами

[Документация о применении Helm на OKD4](https://docs.okd.io/latest/applications/working_with_helm_charts/understanding-helm.html).