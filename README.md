# Настройка мониторинга

## Задание

Настроить дашборд с 4-мя графиками:

- память;
- процессор;
- диск;
- сеть.

## Реализация

Задание сделано на **almalinux/9** версии **v9.4.20240805**. Для автоматизации процесса написаны следующие роли **Ansible**, поднимающие **Zabbix 7** в порядке их выполнения:

- **tls_ca** - генерит корневой сертификат, который используется для подписи всех остальных сертификатов;
- **tls_cert** - генерит сертификаты для узлов **zabbix** и **grafana** с нужным **Subject Alternative Name**;
- **zabbix_release** - устанавливает официальные репозитории **Zabbix** для **AlmaLinux**;
- **node_exporter** - устанавливает **Prometheus Node Exporter** (с аутентификацией и **TLS**);
- **zabbix_agent2** - устанавилвает и настраивает **Zabbix Agent 2** (фактически в конфиге прописывается только **Server**, **ServerActive** и **Hostname** из переменных **zabbix_server_ip** и **zabbix_hostname** в [all.yml](group_vars/all.yml)), а также устанавливает сертификаты и параметры для подключения к серверу и его аутентификации;
- **prometheus** - ставит **prometheus** (с аутентификацией и **TLS**) и настраивает сбор метрик со всех узлов в **inventory**;
- **grafana** - ставит **grafana** и настраивает **provision** из конфигурационных файлов для **datasources** и **dashboards**, настраивает **datasource** для сервера **prometheus**;
- **grafana_dashboards** - устанавливает **dashboard** [otus/otus](roles/grafana_dashboards/files/otus/otus.json);
- **zabbix_server** - поднимает **Zabbix Server 7**:
  - ставит и настраивает **zabbix**, **postgresql**, **nginx**;
  - настраивает **SELinux**, включая политики **httpd_can_connect_zabbix** и **httpd_can_network_connect_db**;
  - создаёт базу данных **zabbix_db_name** (в файле [zabbix.yml](host_vars/zabbix.yml)) и настраивает к ней локальный доступ по паролю (через сокет, смотри [pg_hba](roles/zabbix_server/templates/pg_hba.conf)), пароль от базы данных сохраняется в файле `passwords/zabbix_db.txt`;
  - меняет имя пользователя и пароль администратора на значения из переменных **zabbix_admin_user** и **zabbix_admin_password** в файле [zabbix.yml](host_vars/zabbix.yml), пароль сохраняется в `passwords/zabbix_admin.txt`;
  - настраивает **zabbix-server**, создавая конфигурационные файлы [zabbix_server.conf](roles/zabbix_server/templates/zabbix_server.conf) для сервера и [zabbix.conf.php](roles/zabbix_server/templates/zabbix.conf.php) для веб-интерфейса;
  - настраивает **nginx** и запускает его и **php-fpm**;
  - удаляет `Zabbix server` из **Hosts** в базе данных **zabbix** (через его **API**) и генерит токен для подключения к серверу через **API**, токен сохраняется в `passwords/zabbix_token.txt`;
- **zabbix_inventory** - добавляет узлы **zabbix** и **grafana** в группы **Zabbix servers** и **Linux servers** узлов **Zabbix**, назначает на них шаблоны **Zabbix server health** и **Linux by Zabbix agent** и параметры аутентификации агентов сервером.
- **zabbix_dashboards** - через **Zabbix API** создаёт **Dashboard** с именем **otus** с 3-мя вкладками **CPU And Memory**. **Disk**, **Network** из шаблона [otus](roles/zabbix_dashboards/templates/otus.yml).

## Запуск

Необходимо скачать **VagrantBox** для **almalinux/9** версии **v9.4.20240805** и добавить его в **Vagrant** под именем **almalinux/9/v9.4.20240805**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/almalinux/boxes/9/versions/9.4.20240805/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "almalinux/9/v9.4.20240805"
rm vagrant.box
```

Также необходимо обновить коллекцию **community.zabbix** до версии **3.1.2** и выше:

```shell
ansible-galaxy collection install --upgrade community.zabbix
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.0.20_SUSE r163906**
- **Ansible 2.17.4**
- **community.zabbix 3.1.2**
- **Python 3.11.10**
- **Jinja2 3.1.4**

## Проверка

1. Заходим на [localhost:3000](https://localhost:3000), вводим имя пользователя **otus** и пароль из файла `passwords/grafana_admin.txt`.
2. Переходим в [Dashboards](https://localhost:3000/dashboards) и выбираем **dashboard** `otus` в директории `otus`.
3. Наблюдаем графики:

    ![графики Grafana](images/grafana.png)

4. Также можем зайти непосредственно на сам сервер **Prometheus**. Заходим [localhost:9090](https://localhost:9090/targets). Логин **prometheus**, пароль в файле `passwords/prometheus.txt` (видно, что все метрики успешно получаются через **TLS**).

    ![prometheus targets](images/prometheus.png)

5. Заходим на [localhost:8443](https://localhost:8443/index.php), вводим имя пользователя **otus** и пароль из файла `passwords/zabbix_admin.txt`.
6. Переходим в [Dashboards](https://localhost:8443/zabbix.php?action=dashboard.list) и выбираем [otus](https://localhost:8443/zabbix.php?action=dashboard.view&dashboardid=347).
7. Для того, чтобы на всех графиках появились данные необходимо подождать 10 минут, пока отработают **Discovery rules**.
8. Наблюдаем графики:

    ![процессор и память](images/zabbix_cpu.png)
    ![диск](images/zabbix_disk.png)
    ![сеть](images/zabbix_network.png)

9. На вкладке [Hosts](https://localhost:8443/zabbix.php?action=host.view) можно увидеть, что агенты подключились к серверу и для подключения используется **TLS**:

    ![шифрование](images/zabbix_encryption.png)
