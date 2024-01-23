# запустить обновление пакетного менеджера yum
yum update 
# запустить обновление пакетного менеджера yum
subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
# dnf - аналог yum, запускает скачивание и установку EPEL (Extra Packeges for Enterprise Linux) это понадобится позже
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm 
# добавить пользователя
adduser qwe  
# задать пароль пользователю
passwd qwe 
# добавить пользователя в группу администраторв, где '-a' добавления пользователя в дополнительную группу из параметра '-G', не заменяя текущюю группу
usermod -a -G wheel qwe
# редактируем параметры sudo через редактор nano
EDITOR=nano visudo 
# добавляем в файл конфигурации строку
user ALL=(ALL) NOPASSWD: ALL 
# устанавливаем доп инструментарий к yum
sudo yum install -y yum-utils 
# добавляем репозиторий для скачивания docker
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
# устанавливаем последнюю версию docker, создаем группу docker
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 
#	запускаем
sudo systemctl start docker
# проверяем работу
# Этой командой мы скачаем и запустим тестовый контейнер.  
sudo docker run hello-world
# настраиваем запуск и использование docker без sudo, добавляем группу с именем docker
sudo groupadd docker
# добавляем нашего созданного пользователя в группу
sudo gpasswd -a $USER docker
# проверяем работу docker без sudo. Так же, включим службу docker, для автоматического запуска.
sudo systemctl enable docker

# Для упрощения запуска сервисов создаем systemd unit'ы(файл описания службы\сервиса) для каждого из требуемых нам инструментов
# Общее описание структуры юнита:
# -----------------------------------------------------------------
[Unit]
# Описание сервиса
Description= 
# Очередность запуска сервиса. В нашем случае контейнер будет стартовать после того, как поднимается сеть и docker
After=network.target docker.socket
# Необходимые для запуска условия – запуск docker. Условий может быть несколько.
Requires=docker.socket
[Service] 
# УТОЧНИТЬ ОПИСАНИЕ
RestartSec=100
# Условие перезапуска – при ошибке
Restart=on-failure
# Этой командой, выполняемой перед ExecStart, мы скачиваем необходимый образ, через « : » указываем версию, если не указано - используется latest
ExecStartPre=-/usr/bin/docker pull image_name:version
# Команда на запуск исполняемого файла docker, с параметрами приведенными ниже
ExecStart=/usr/bin/docker run \
# Имя контейнера
  --name container_name \
# Удаление контейнера после остановки
  --rm \
# Проброс порта для контейнера
  -p host_port:container_port \
# Проброс каталога для контейнера
  -v /host/path:/container/path \
# Этот параметр окружения мы указываем для того, чтобы подключаться к произвольному серверу базы данных.
  -e container_environment_variables \
# Название образа контейнера с указанием версии
  image_name:version
# Время ожидания старта контейнера 
TimeoutStartSec=600
# Команда на остановку службы
ExecStop=/usr/bin/docker stop container_name
# Описание «убийства» процесса. В нашем случае, устанавливается значение none, которое позволяет процессу выйти из жизненного цикла менеджера служб 
# и управления ресурсами и продолжать работу, даже если служба считается остановленной и не потребляющей любые ресурсы. 
KillMode=none
[Install]
# multi-user.target или runlevel3.target соответствует runlevel=3 многопользовательский режим без графики
WantedBy=multi-user.target
# -----------------------------------------------------------------


# MySQL
# MySQL — это реляционная система управления базами данных (СУБД), которая распространяется как свободное программное обеспечение. 
# Слово «реляционный» означает, что базы представлены в виде связанной информации и описываются как набор связей. MySQL работает с языком запросов SQL, который традиционно используется в базах данных.
# Для начала работы с контейнером mysql предлагаю сразу использовать systemd unit, так как это в дальнейшем упростит нам работу и не потребует постоянно запускать контейнер в интерактивном режиме.
# Создаем systemd unit
sudo nano /etc/systemd/system/mysql.service
# Содержимое файла 
# -----------------------------------------------------------------
[Unit]
Description=mysql
Documentation=bebebe
After=network.target docker.socket
Requires=docker.socket

[Service]
RestartSec=100
Restart=on-failure

ExecStartPre=-/usr/bin/docker pull mysql:5.7

ExecStart=/usr/bin/docker run \
  --name mysql_1 \
  --rm \
  -p 3306:3306 \
  -v /db/mysql:/var/lib/mysql \
  -e MYSQL_ROOT_USER=root \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=tst_db \
  -e MYSQL_USER=juser \
  -e MYSQL_PASSWORD=jpassword \
  mysql:5.7

TimeoutStartSec=600

ExecStop=/usr/bin/docker stop mysql_1

KillMode=none

[Install]
WantedBy=multi-user.target
# -----------------------------------------------------------------

# Mysql exporter
# Создаем каталог конфигурации mysql exporter
sudo mkdir /etc/mysql_exporter
# Создаем файл конфигурации экспортера (для хранения кредов)
sudo nano /etc/mysql_exporter/.my.cnf

# Содержимое файла
# --------------------------------------------------------------------
[client]
user = root
password = rootpassword
# --------------------------------------------------------------------

# Создаем systemd unit
sudo nano /etc/systemd/system/mysql_exporter.service
# Содержимое файла
# -----------------------------------------------------------------
[Unit]
Description=mysql_exporter_1
Documentation=123
After=network.target docker.socket
Requires=docker.socket

[Service]
RestartSec=100
Restart=on-failure

ExecStartPre=-/usr/bin/docker pull prom/mysqld-exporter

ExecStart=/usr/bin/docker run \
  --name mysql_exporter_1 \
  --rm \
  -p 9104:9104 \
  --link=mysql_1:bdd  \
  -v /etc/mysql_exporter:/cfg \
  -e DATA_SOURCE_NAME="root:rootpassword@(bdd:3306)/tst_db" \
  prom/mysqld-exporter --config.my-cnf=/cfg/.my.cnf

TimeoutStartSec=600

KillMode=none

[Install]
WantedBy=multi-user.target
# --------------------------------------------------------------------


# Phpmyadmin
# Веб-приложение phpMyAdmin — это программа, которая позволяет управлять базами данных через удобный графический интерфейс. 
# Ее устанавливают на сервер, где хранится БД, а затем открывают в браузере для удаленного администрирования СУБД.
# Создаем systemd unit
sudo nano /etc/systemd/system/phpmyadmin.service
# Содержимое файла 
# -----------------------------------------------------------------
[Unit]
Description=phpmyadmin_1
Documentation=123
After=network.target docker.socket
Requires=docker.socket

[Service]
RestartSec=100
Restart=on-failure

ExecStartPre=-/usr/bin/docker pull phpmyadmin

ExecStart=/usr/bin/docker run \
  --name phpmyadmin_1 \
  --rm \
  -p 8000:80 \
  -e PMA_ARBITRARY=1 \
  phpmyadmin

TimeoutStartSec=600

KillMode=none

[Install]
WantedBy=multi-user.target
# -----------------------------------------------------------------


# Node-Exporter
# Node Exporter — http-приложение, собирающее метрики операционной системы. Prometheus собирает данные с одного, как в нашем случае, или нескольких экземпляров Node Exporter.
# Создаем systemd unit
sudo nano /etc/systemd/system/node_exporter.service
# Содержимое файла 
# -----------------------------------------------------------------
[Unit]
Description=node_exporter_1
Documentation=123
After=network.target docker.socket
Requires=docker.socket

[Service]
RestartSec=100
Restart=on-failure

ExecStartPre=-/usr/bin/docker pull bitnami/node-exporter

ExecStart=/usr/bin/docker run \
  --name node_exporter_1 \
  --rm \
  -p 9100:9100 \
  bitnami/node-exporter

TimeoutStartSec=600

KillMode=none

[Install]
WantedBy=multi-user.target
# -----------------------------------------------------------------

# Cadvisor
# Создаем systemd unit
sudo nano /etc/systemd/system/cadvisor.service
# Содержимое файла
# -----------------------------------------------------------------
[Unit]
Description=cadvisor
Requires=docker.service
After=network.target docker.socket
 
[Service]
Restart=always
ExecStartPre=-/usr/bin/docker rm cadvisor
ExecStart=/usr/bin/docker run \
  --rm \
  --publish=8080:8080 \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --privileged=true \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:v0.44.0
 
ExecStop=/usr/bin/docker stop -t 10 cadvisor
 
[Install]
WantedBy=multi-user.target
# --------------------------------------------------------------------

# Prometheus 
# Prometheus — система мониторинга серверов и программ с открытым исходным кодом.
# Главное отличие Prometheus от остальных систем мониторинга — метод сбора данных. 
# Prometheus сам берёт нужную ему информацию с серверов и устройств, обращаясь к целевым объектам.
# Сервер Prometheus считывает параметры (метрики) целевых объектов с интервалами, которые задаёт пользователь. 
# Данные от целевых объектов передаются на сервер в формате http и хранятся в базе данных временных рядов (time-series database). 
# Создание пользователя и директорий
# Создание пользователя без домашней директории «–М», с определенным числовым значением «-u» user id 1101, 
# что важно в нашем случае для привязки имени пользователя в контейнере конкретно для prometheus. 
# «-s» указание shell (среды выполнения) для пользователя, /bin/false – запрет shell login на аккаунт. В данном случае - prometheus
sudo useradd -M -u 1101 -s /bin/false prometheus
# Создание директории, которые указаны в пути «-p». Это каталог для конфигураций
sudo mkdir -p /etc/prometheus/rule_files
# Создание директории для конфигурационного файла prometheus.yml
sudo mkdir -p /etc/prometheus
# Создание каталога данных 
sudo mkdir -p /data/prometheus
# Выдаем пользователю prometheus права на каталоги. Команда chown выполняется с опцией «-R», рекурсивно.
sudo chown -R prometheus /etc/prometheus /data/prometheus

# Создание конфигурационного файла
# !!Вместо HOST_IP_ADDRESS указать текущий адрес хоста!!
sudo nano /etc/prometheus/prometheus.yml
# содержимое файла
# -----------------------------------------------------------------
global:
# интервал опроса 
  scrape_interval: 15s
# таймаут опроса
  scrape_timeout: 10s
# проверка правил опроса
  evaluation_interval: 30s

# конфигурация опроса
scrape_configs:
# имя job’ы (задание)
  - job_name: 'prometheus'
# постоянные настройки, в нашем случае – адрес сервера.
    static_configs:
      - targets:
        - HOST_IP_ADDRESS:9090

# имя job’ы (задание)
  - job_name: 'node'
# постоянные настройки, в нашем случае – адрес сервера.
    static_configs:
      - targets:
        - HOST_IP_ADDRESS:9100
# имя job’ы (задание)
  - job_name: 'mysql'
# постоянные настройки, в нашем случае – адрес сервера.
    static_configs:
      - targets:
          - HOST_IP_ADDRESS:3306
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: HOST_IP_ADDRESS:9104
# имя job’ы (задание)
  - job_name: 'cadvisor'
# постоянные настройки, в нашем случае – адрес сервера.
    static_configs:
      - targets:
          - HOST_IP_ADDRESS:8080
# -----------------------------------------------------------------

# Создаем systemd unit
sudo nano /etc/systemd/system/prometheus.service
# Содержимое файла
# -----------------------------------------------------------------
[Unit]
Description=prometheus
Documentation=123
After=network.target docker.socket
Requires=docker.socket

[Service]
RestartSec=100
Restart=on-failure

ExecStartPre=-/usr/bin/docker pull prom/prometheus

ExecStart=/usr/bin/docker run \
  --rm \
  --user=1101 \
  --publish=9090:9090 \
  --volume=/etc/prometheus/:/etc/prometheus/ \
  --volume=/data/prometheus/:/prometheus/ \
  --name=prometheus_1 \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=1d
ExecStop=/usr/bin/docker stop prometheus_1
 
[Install]
WantedBy=multi-user.target
# -----------------------------------------------------------------


# Grafana
# Grafana — универсальный инструмент для работы с аналитическими данными, которые хранятся в разных источниках.
# Она сама ничего не хранит и не собирает, а является универсальным клиентом для систем хранения метрик. 
# Например, с помощью нее можно обращаться за данными как в обычную БД, так и в специализированные аналитические системы типа Prometheus, как в нашем случае.
# Создание пользователя и директорий
# Создание пользователя без домашней директории «–М», с определенным числовым значением «-u» user id 1102, 
# что важно в нашем случае для привязки имени пользователя в контейнере конкретно для grafana. 
# «-s» указание shell (среды выполнения) для пользователя, /bin/false – запрет shell login на аккаунт. В данном случае - grafana
sudo useradd -M -u 1102 -s /bin/false grafana
# Создание директории, которые указаны в пути «-p». Это каталог декларативного описания источников данных
sudo mkdir -p /etc/grafana/provisioning/datasources 
# Создание каталога декларативного описания дашбордов
sudo mkdir /etc/grafana/provisioning/dashboards 
# Создание каталога данных 
sudo mkdir -p /data/grafana/dashboards 
# Выдаем пользователю grafana права на каталоги. 
# Команда chown выполняется с опцией «-R», рекурсивно.
sudo chown -R grafana /etc/grafana/ /data/grafana
# Создаем systemd unit
sudo nano /etc/systemd/system/grafana.service
# Содержимое файла
# -----------------------------------------------------------------
[Unit]
Description=grafana
Documentation=bebebe
After=network.target docker.socket
Requires=docker.socket

[Service]
Restart=always

ExecStartPre=-/usr/bin/docker pull grafana/grafana:9.2.8

ExecStart=/usr/bin/docker run \
  --rm \
  --user=1102 \
  --publish=3000:3000 \
  --volume=/etc/grafana/provisioning:/etc/grafana/provisioning \
  --volume=/data/grafana:/var/lib/grafana \
  --name=grafana_1 \
  grafana/grafana:9.2.8
  
ExecStop=/usr/bin/docker stop grafana
 
[Install]
WantedBy=multi-user.target
# -----------------------------------------------------------------

# yml файл для описания источника данных

# Создаем main.yml
sudo nano /etc/grafana/provisioning/datasources/main.yml

# Содержимое файла
# -----------------------------------------------------------------
# Версия конфигурационного файла
apiVersion: 1
# список источников данных для grafana
datasources:
# имя источника данных для grafana
  - name: Prometheus
    # тип источника данных
    type: prometheus
    # версия протокола соединения с источником данных
    version: 1
    # тип доступа к источнику данных (см документацию)
    access: proxy
    # айди организации (часть ролевой модели grafana), к которой применятеся концигурция
    orgId: 1
    # авторизация на источнике данных
    basicAuth: false
    # Признак разрешения пользователям изменения параметров источника данных 
    editable: false
    # адрес источника данных
    url: http://<hostname>:9090
# --------------------------------------------------------------------

# yml файл для описания дашбордов

# Создаем main.yml
sudo nano /etc/grafana/provisioning/dashboards/main.yml

# Содержимое файла
# --------------------------------------------------------------------
# Версия конфигурационного файла
apiVersion: 1

# Источник дашборда
providers:
# Имя источника дашборда
  - name: 'main'
# ID организации (явл. осн. частью ролевой модели grafana), 
# к которой применяется конфигурация
    orgId: 1
# Расположение заргужаемых дашбордов внутри структуры grafana
    folder: ''
# Тип загружаемых дашбордов
    type: file
# Ограничение на удаление пользователем
    disableDeletion: false
# Ограничение на редактирование дашборда
    editable: True
# Путь расположения дашбордов (внутри контейнера, т.е. проброшенный с хоста)
    options:
      path: /var/lib/grafana/dashboards
# --------------------------------------------------------------------

