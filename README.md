# Дипломная работа по профессии «Системный администратор» - Савин Алексей  

# Содержание
- [Задача](#Задача)  
- [Инфраструктура](#Инфраструктура)  
    - [Сайт](#Сайт)
    - [Мониторинг](#Мониторинг)
    - [Логи](#Логи)
    - [Сеть](#Сеть)
    - [Резервное копирование](#backup)
# Выполнение дипломной работы  
### Terraform
- [Инфраструктура](#terra)
    - [Сеть](#net)
    - [Группы безопасности](#group)
    - [Load Balancer](#balancer)
    - [Резервное копирование](#snapshot)
### Ansible
- [Установка и настройка ansible](#cfg)
- [NGINX](#web)
- [Мониторинг](#zabbix)
- Логи
    -  [Установка Elasticsearch](#elastic)
    -  [Установка Kibana](#kibana)
    -  [Установка Filebeat](#filebeat)

**ссылки на ресурсы**  
[Сайт](http://158.160.163.89/)  
[Kibana](http://84.252.136.178:5601/)  
[Zabbix](http://84.201.155.91/zabbix)  

--- 

### <a id="Задача">Задача</a>  
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от облака в git. 

### <a id="Инфраструктура">Инфраструктура</a> 
Для развёртки инфраструктуры используйте Terraform и Ansible.  

Не используйте для ansible inventory ip-адреса! Вместо этого используйте fqdn имена виртуальных машин в зоне ".ru-central1.internal". Пример: example.ru-central1.internal  

Важно: используйте по-возможности минимальные конфигурации ВМ:2 ядра 20% Intel ice lake, 2-4Гб памяти, 10hdd, прерываемая.  

Так как прерываемая ВМ проработает не больше 24ч, перед сдачей работы на проверку дипломному руководителю сделайте ваши ВМ постоянно работающими.  

Ознакомьтесь со всеми пунктами из этой секции, не беритесь сразу выполнять задание, не дочитав до конца. Пункты взаимосвязаны и могут влиять друг на друга.  

### <a id="Сайт">Сайт</a> 
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Создайте Target Group, включите в неё две созданных ВМ.

Создайте Backend Group, настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

Создайте HTTP router. Путь укажите — /, backend group — созданную ранее.

Создайте Application load balancer для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

Протестируйте сайт curl -v <публичный IP балансера>:80

### <a id="Мониторинг">Мониторинг</a>
Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление метрик в Zabbix.

Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие графики.

### <a id="Логи">Логи</a> 
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch.

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### <a id="Сеть">Сеть</a> 
Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application load balancer определите в публичную подсеть.

Настройте Security Groups соответствующих сервисов на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh. Эта вм будет реализовывать концепцию bastion host . Синоним "bastion host" - "Jump host". Подключение ansible к серверам web и Elasticsearch через данный bastion host можно сделать с помощью ProxyCommand . Допускается установка и запуск ansible непосредственно на bastion host.(Этот вариант легче в настройке)

### <a id="backup">Резервное копирование</a> 
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное копирование.  

---

# Выполнение дипломной работы  

## Terraform

### <a id="terra">Инфраструктура</a>  

Поднимаем инфраструктуру в Yandex Cloud используя **terraform**  

в целях безопасности указываем `token id, cloud id и folder id` через задание переменных текущего сеанса в консоли подставляя свои ***id*** в команды  
```
export YC_TOKEN=
export YC_CLOUD_ID=
export YC_FOLDER_ID=
```
далее проверяем правильность конфигурации командой
`terraform plan` сверяем что все верно и запускаем процесс поднятия инфраструктуры  командой `terraform apply`  
в конце выполнения получаем данные output получение которые мы прописывали в файле **outputs.tf**  

![output](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/fa4cadcc-6960-4d50-b26a-fb09f56877c4)  

после завершения работы **terraform** проверяем в web консоли YC созданную инфраструктуру.  Сервера WEB-1 и WEB-2 созданы в разных зонах.

![VM](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/20547ee8-e272-4fd9-a294-b920226be8d2)  

### <a id="net">Сеть</a>  

 **VPC и subnet** 

создаем 1 VPC с публичными и внутренними подсетями и таблицей маршрутизации для доступа к интернету ВМ находящихся внутри сети за Бастионом который будет выступать в роли интернет-шлюза.  

![maps_VPC](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/a41a67a6-1b8a-4fff-a76f-2a29e772f418)  

### <a id="group">Группы безопасности</a>  

![SG_ALL](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/bd9eadc4-6246-41d6-8195-b31d6b9aa999)  

**SG_LB**  

![sg_L7B](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/1ae82a3e-b5fc-41e9-b37c-e76272148c4d)  

**SG_internal** с разрешением любого трафика между ВМ кому присвоена данная SG  

![sg_internal](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/4c571125-3ea8-441d-9934-18957034db38)

**SG_bastion** c с открытием только 22 порта для работы SSH  

![sg_bastion](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/5185cac3-b7da-4b51-9d1b-b2cb41705821)  


**SG_kibana** c открытым портом 5601 для доступа c интернета к Fronted Kibana  

![sg_kibana](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/fcb7e596-1652-4e0a-aab1-68f8fd8ca86c)  

**SG_zabbix** с открытым портом 80 и 10051 для доступа с интернета к Fronted Zabbix и работы Zabbix agent  

![sg_zabbix](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/b46cd2ed-9f54-44ba-8a81-b3e84b7afb56)  

### <a id="balancer">Load Balancer</a>  

**Создаем Target Group**  

![target-group](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/3884fdfa-ecb2-42c1-b8cc-6c0e5d6a65d6)  

**Создаем Backend Group**  

Настраиваем backends на target group, ранее созданную, healthcheck на корень (/) и порт 80, протокол HTTP.  

![backend-group](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/2756527b-c4f7-4068-8cc2-e00982ed6e70)  

![health_status](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/647df8e2-8a23-4580-9203-693874a19c13)  

**Создаем HTTP router**  

![http-router](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/644bd5f3-609a-446c-a0e0-fe259d41e8ef)  

**Создаем Application load balancer**  

для распределения трафика на веб-сервера созданные ранее. Указываем HTTP router, созданный ранее, задаем listener тип auto, порт 80.  

![loadBalancer](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/24ea7fd4-7ae5-4c8c-8450-0e67f04399ab)  

**карта балансировки**  

![maps_balancer](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/00a6e78f-d645-415f-866e-9cd388712d82)  

### <a id="snapshot">Резервное копирование</a>  

**snapshot**  

создаем в terraform блок с расписанием snapshots  

![snap](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/7cea4203-10d3-48f1-9063-8461e85a9f91)  

**проверяем на следующий день что снимки создались по расписанию**  

![snap_work](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/4da52588-0426-4fe1-83dd-b20867be9f1d)  


## Ansible  

### <a id="cfg">Установка и настройка ansible</a>  

устанавливаем ***ansible*** на локальном хосте где работали с ***terraform***  и настраиваем его на работу через ***bastion***  

**файл конфигурации**

![ansible_cfg](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/19f5a4c9-9bca-4bc7-99d5-f25256cdd9ba)  

**файл inventory**

создаем файл hosts.ini c использованием FQDN имен серверов вместо ip  
(т.к. DNS имя для ВМ bastion не регистрировалось глобально, то я просто использовал iр для доступа из интернета)  

![ans_inventori](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/5ce63647-2701-44ff-8554-2a1d67f16eb0)  

**проверяем доступность ВМ используя модуль ping**  

![ans_ping](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/8c810d11-7202-4a91-8995-3bede4464397)  


### <a id="web">Установка NGINX и загрузка сайта</a>  

**ставим nginx**  
запускаем playbook установки Nginx с созданием web страницы  

![ans_nginx](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/adaa1824-28cb-4162-ad2a-fa8d26cb0c6e)  

**проверяем доступность сайта в браузере по публичному ip адресу Load Balancer**  

![site](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/a3161999-a48c-44c2-8eb3-b22423a6de3f)  

**делаем запрос curl -v 158.160.163.89:80**  

![curl](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/101e2c0c-db99-40dc-8b24-840c7ebd45a9)  

**проверяем работу Load Balancer в web консоли YC**  

видим что меняется ip backend, значит балансировщик работает.  

![logs_work_balancer](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/4d4d251f-1155-459c-83b7-7eefdca9cdba)  

### <a id="zabbix">Мониторинг</a>  

**установка Zabbix сервера**  

![ans_zabbix](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/828cee3a-01f4-4c05-a77f-6adaa4877bca)  

**проверяем доступность frontend zabbix сервера**  

![zabbix_web](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/bf293e3a-d961-4a00-9e60-710eac6a877c)  

**установка Zabbix агентов на web сервера**  

![ans_zabbix_agent](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/89e28c48-cffe-449a-9d43-901f2c0d0023)  

**проверяем статус служб zabbix agent на web серверах**  
WEB-1  
![status_zabbix_agent_web1](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/c161288b-574b-4acf-9e94-4045d44f07d0)  

WEB-2  
![status_zabbix_agent_web2](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/c0ed9d3d-2bf5-4f6f-b69e-951779705755)  

**добавляем хосты используя FQDN имена в zabbix сервер и настраиваем дашборды**  

![zabbix_hosts](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/aeab9b35-d996-49c8-a252-5d9003dfd444)  

![dashboard_1](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/76b13656-b3ad-4756-939f-20d393be9c17)  

![dashboard_2](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/9a7f3db3-44f7-47a5-a6e2-e3071e78f551)  

## Установка стека ELK для сбора логов  

### <a id="elastic">Установка Elasticsearch</a>  

![ans_elastic](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/e446b9c1-9d64-439e-b73e-8860bb4cc4f7)  

**Проверяем статус Elasticsearch**  

![status_elasticserch](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/68a4385f-8cfa-4e83-ba01-6dfa060d4039)  

### <a id="kibana">Установка Kibana</a>  

**устанавливаем Kibana**  

![ans_kibana](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/85fd8624-395d-4ede-a5b0-ab8ba86be5c2)  

**проверяем что Kibana работает**  

![status_kibana](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/b135a85c-bb10-47ff-9c96-3605c6a7caf6)  

![kibana](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/7f332afe-dd60-4ace-9323-ffff50343b1c)  

![log_kibana](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/89a7ca32-096b-4d4f-bab6-736544251c4d)  

### <a id="filebeat">Установка Filebeat</a>  

**Устанавливаем Filebeat на web сервера**  

![ans_filebaet](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/13501fe0-319c-4925-bfd8-09762f86fd68)  

**проверяем статус службы filebeat на серверах**  
WEB-1  
![status_filebeat_web1](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/dc03343d-047b-43f9-a368-5a6918792fbc)  

![log_filebeat_web1](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/077484a7-008b-4783-8f50-586ca490f1f6)  


WEB-2  
![status_filebeat_web2](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/ef8ce53c-f8c7-4845-bb7b-2ae30bb51e1e)  

![log_filebeat_web2](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/69717628-ce4f-4aa8-89fd-9580300b6da9)  

**Проверяем в Kibana что Filebeat доставляет логи в Elasticsearch**  

![kibana_logs_filebeat](https://github.com/AI-Savin/Netology_Diplom/assets/149881259/b5842136-05ab-4427-ba0f-0e2620cdb2a2)  


---

