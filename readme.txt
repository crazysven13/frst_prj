# Добавляем пользователя
sudo adduser qwe  
# Задаем пароль пользователю
sudo passwd qwe 
# Добавляем пользователя в группу администраторов, где '-a' добавления пользователя в дополнительную группу из параметра '-G', не заменяя текущюю группу
sudo usermod -a -G wheel qwe
# редактируем параметры sudo через редактор nano
sudo EDITOR=nano visudo 
# добавляем в файл конфигурации строку
user ALL=(ALL) NOPASSWD: ALL 
# запустить обновление пакетного менеджера yum
sudo yum update 
# запустить обновление пакетного менеджера yum
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
# dnf - аналог yum, запускает скачивание и установку EPEL (Extra Packeges for Enterprise Linux) это понадобится позже
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm 
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
# добавляем нашего созданного пользователя в группу
sudo gpasswd -a $USER docker
# проверяем работу docker без sudo. Так же, включим службу docker, для автоматического запуска.
sudo systemctl enable docker

# Устанавливаем Git, т.к. все конфигурационные файлы и systemd unit'ы находятся там, клонируем свой репозиторий
sudo yum install git
# Создаем SSH ключ для подключения к нашему репозиторию
ssh-keygen -t ed25519 -C "MAIL@gmail.com"
# Выводим на экран и добавляем в git свой публичный ключ
cat /home/qwe/.ssh/id_ed25519.pub
(вставить ключ в https://github.com/settings/keys "New SSH key")
# Включаем Ssh-agent
eval `ssh-agent -s`
# Добавляем в агент наш ssh-ключ
ssh-add ~/.ssh/id_ed25519
# Клонируем свой репозиторий
git clone git@github.com:crazysven13/frst_prj.git
# Копируем наши systemd unit'ы в папку systemd
sudo cp -r ~/frst_prj/systemd_unit/* /etc/systemd/system/
# Сообщаем systemd о том что появились новые службы
sudo systemctl daemon-reload

# MySQL
# Создаем каталог базы данных
sudo mkdir -p /db/mysql
# Запускаем MySQL
sudo systemctl start mysql

# Mysql exporter
# Создаем каталог конфигурации mysql exporter
sudo mkdir /etc/mysql_exporter
# Копируем файл конфигурации экспортера (для хранения кредов)
sudo cp ~/frst_prj/mysql_exporter/.my.cnf /etc/mysql_exporter/.my.cnf
# Запускаем экспортер метрик для mysql
sudo systemctl start mysql_exporter


# Phpmyadmin
# Запускаем phpmyadmin
sudo systemctl start phpmyadmin

# Node-Exporter
# Запускаем экспортер метрик хоста
sudo systemctl start node_exporter


# Cadvisor
# Запускаем экспортер метрик docker и контейнеров
sudo systemctl start cadvisor

# Prometheus 
# Создание пользователя и директорий
sudo useradd -M -u 1101 -s /bin/false prometheus
# Создание директории, которые указаны в пути «-p». Это каталог для конфигураций
sudo mkdir -p /etc/prometheus/rule_files
# Создание директории для конфигурационного файла prometheus.yml
sudo mkdir -p /etc/prometheus
# Создание каталога данных 
sudo mkdir -p /data/prometheus
# Выдаем пользователю prometheus права на каталоги. Команда chown выполняется с опцией «-R», рекурсивно.
sudo chown -R prometheus /etc/prometheus /data/prometheus
# Копируем конфигурационный файл 
sudo cp -r ~/frst_prj/prometheus /etc/
# Редактирование конфигурационного файла
# !!Вместо HOST_IP_ADDRESS указать текущий адрес хоста!!
sudo nano /etc/prometheus/prometheus.yml
# Запускаем Prometheus
sudo systemctl start prometheus

# Grafana
# Создание пользователя и директорий
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
# Копируем конфигурационные файлы grafana
sudo cp -r ~/frst_prj/grafana /etc/
# Редактируем yml файл для описания источника данных
# Указать в строке "url" адрес сервера
sudo nano /etc/grafana/provisioning/datasources/main.yml
# Копируем дашборды в каталог grafana
sudo cp -r ~/frst_prj/data /
# Запускаем grafana
sudo systemctl start grafana
