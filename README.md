# diplom-dorabotka

Задание на дипломную работу 
==========
* [Задача](#Задача)
* [Инфраструктура](#Инфраструктура)
    * [Сайт](#Сайт)
    * [Мониторинг](#Мониторинг)
    * [Логи](#Логи)
    * [Сеть](#Сеть)
    * [Резервное копирование](#Резервное-копирование)


---------

## Задача
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в [Yandex Cloud](https://cloud.yandex.com/) и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от облака в git. Используйте [инструкцию](https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-quickstart#get-credentials).

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**
## Инфраструктура
Для развёртки инфраструктуры используйте Terraform и Ansible.  

Не используйте для ansible inventory ip-адреса! Вместо этого используйте fqdn имена виртуальных машин в зоне ".ru-central1.internal". Пример: example.ru-central1.internal  

Важно: используйте по-возможности **минимальные конфигурации ВМ**:2 ядра 20% Intel ice lake, 2-4Гб памяти, 10hdd, прерываемая. 

**Так как прерываемая ВМ проработает не больше 24ч, перед сдачей работы на проверку дипломному руководителю сделайте ваши ВМ постоянно работающими.**

Ознакомьтесь со всеми пунктами из этой секции, не беритесь сразу выполнять задание, не дочитав до конца. Пункты взаимосвязаны и могут влиять друг на друга.

### Сайт
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Виртуальные машины не должны обладать внешним Ip-адресом, те находится во внутренней сети. Доступ к ВМ по ssh через бастион-сервер. Доступ к web-порту ВМ через балансировщик yandex cloud.

Настройка балансировщика:
1. Создайте [Target Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/target-group), включите в неё две созданных ВМ.

2. Создайте [Backend Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/backend-group), настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

3. Создайте [HTTP router](https://cloud.yandex.com/docs/application-load-balancer/concepts/http-router). Путь укажите — /, backend group — созданную ранее.

4. Создайте [Application load balancer](https://cloud.yandex.com/en/docs/application-load-balancer/) для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

Протестируйте сайт
`curl -v <публичный IP балансера>:80` 

### Мониторинг
Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление метрик в Zabbix. 

Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие графики.
### Логи
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch.

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### Сеть
Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application load balancer определите в публичную подсеть.

Настройте [Security Groups](https://cloud.yandex.com/docs/vpc/concepts/security-groups) соответствующих сервисов на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh.  Эта вм будет реализовывать концепцию  [bastion host]( https://cloud.yandex.ru/docs/tutorials/routing/bastion) . Синоним "bastion host" - "Jump host". Подключение  ansible к серверам web и Elasticsearch через данный bastion host можно сделать с помощью  [ProxyCommand](https://docs.ansible.com/ansible/latest/network/user_guide/network_debug_troubleshooting.html#network-delegate-to-vs-proxycommand) . Допускается установка и запуск ansible непосредственно на bastion host.(Этот вариант легче в настройке)

Исходящий доступ в интернет для ВМ внутреннего контура через [NAT-шлюз](https://yandex.cloud/ru/docs/vpc/operations/create-nat-gateway).

### Резервное копирование
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное копирование.
# Моё решение 

### Создание облачной инфраструктуры при помощи Terraform
Установка и подключение Terraform производилось по официальной инструкции на которое ссылается задание
https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart?utm_referrer=https%3A%2F%2Fgithub.com%2Fnetology-code%2Fsys-diplom%2Ftree%2Fdiplom-zabbix%3Ftab%3Dreadme-ov-file

При помощи манифеста Terraform main.tf были созданы и настроены:
* Сеть и подсети
* Виртуальные машины
* Балансировщик
* Группы безопасности
* Бекап образов виртуальных машин

![image](https://github.com/user-attachments/assets/e4d3cb64-a865-4d92-b5d8-d23d83d9d186)

![image](https://github.com/user-attachments/assets/88c9711f-a0c7-4a3f-8310-4847c494ae0a)


Все в соответствии с заданием, созданные объекты в файле main.tf находятся в том же порядке, как и в списке. Настройка облачной инфраструктуры так же производится полностью автоматически в соответствии со скриптом.
Скрипты создания ВМ ссылаются на файл meta.txt, который главным образом содержит публичную половину ssh ключа.
Файл variables.tf содержит ID облака и каталога, где производится развертка инфраструктуры.
### Установка софта при помощи Ansible

На bastion-host установлен Ansible, подключены хосты и написаны плейбуки для установки и настройки необходимого софта. Почти все настройки происходят автоматически при запуске плейбуков. Для обеспечения доступа bastion-host ко всем остальным хостам инфраструктуры на него необходимо скопировать ssh ключ при помощи scp id_ed25519 в директорию ~/.ssh
Также ключ скопирован на все остальные ВМ для корректной работы софта.
Файл hosts.txt содержит хосты по их fqdn имени и логин и ссылку на ключ для подключения по ssh.
Проверка: ansible -i hosts.txt all -m ping
Файлы *_pb.yaml это запускаемые плейбуки, которые устанавливают и конфигурируют софт в соответствии с заданием. 

![image](https://github.com/user-attachments/assets/5bf29ff9-b4ea-45fd-900a-e231daafc590)

![image](https://github.com/user-attachments/assets/5cb5ba2c-9eeb-428b-908d-8e5c0d4f28d6)

![image](https://github.com/user-attachments/assets/10adede9-d5e1-46a0-9b97-982f6afaaf5c)

![image](https://github.com/user-attachments/assets/6aa7161f-7b65-4d93-a9ac-45a4db2b20c4)

![image](https://github.com/user-attachments/assets/3e9bd7bf-940a-4fea-8470-830b56386b94)

![image](https://github.com/user-attachments/assets/0e687452-60cd-447d-b480-a7edb65e1839)

![image](https://github.com/user-attachments/assets/9616360c-332d-4551-847f-039d5d333a06)
Таким образом вся инфраструктура почти полностью развернута в соответствии с заданием автоматически при помощи манифеста Terraform и плейбуков Ansible. Для того, чтобы сервера nginx начали посылать логи filebeat на машинах web1 и web2 необходимо дать команды:
* sudo filebeat setup
* sudo service filebeat start

![image](https://github.com/user-attachments/assets/62f73573-dfc1-4071-80e8-2e294c3ff638)

После подключения хостов Zabbix для того, чтобы они отобрализись как доступные на всех подключаемых машинах дается команда:
zabbix_agentd -V
Настройка дешборда и сбора метрик производится в GUI в браузере. 

![image](https://github.com/user-attachments/assets/3577307a-06a0-4357-9e3c-711d789b663f)

Веб сервера доступны по адресу балансировщика:
62.84.112.212

Kibana доступна по публичному адресу по порту :5601
Zabbix доступен по публичному адресу по порту :8080 лог пасс Admin netology (Admin именно с большой буквы).


![image](https://github.com/user-attachments/assets/39f6c291-b98b-4898-9107-b2052b3491b1)

![image](https://github.com/user-attachments/assets/e65588b9-046a-4003-9a21-3e52ecee70eb)

![image](https://github.com/user-attachments/assets/793f6b6a-267d-4101-a284-d8bc5989b38f)

![image](https://github.com/user-attachments/assets/96aba7d4-8862-42e6-a7d7-8683fe215257)

![image](https://github.com/user-attachments/assets/02314a07-c992-48ca-a9cf-dedb84a5d324)

![image](https://github.com/user-attachments/assets/ee3c630c-aa1b-439e-bf2f-38fa7e1f3153)

### Доработка
* выставила nat = false в main.tf при создании ВМок web и elastic
* добавила enabled: true во все роли и плейбуки ansible
* добавила пароль для kibana (файл теперь не копируется из локального репо, а создается прямо в плейбуке), vault добавила 
* загрузила все файлы 
