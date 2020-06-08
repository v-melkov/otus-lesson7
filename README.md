# Стенд для домашнего занятия "Systemd"

#### Отключим установку дополнений VirtualBox
    config.vbguest.no_install = true

## Задание 1.
Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл лога и ключевое слово должны задаваться в /etc/sysconfig.

Файл лога - /var/log/messages

Ключевое слово - systemd
##### Создадим файл настроек /etc/sysconfig/monitoring...
    touch /etc/sysconfig/monitoring
    echo -e "LOGFILE=/var/log/messages\nKEYWORD=systemd" >> /etc/sysconfig/monitoring

##### Создадим файл таймера...
    touch /etc/systemd/system/monitoring.timer
    chmod 664 /etc/systemd/system/monitoring.timer

    cat <<'EOF1_1' >/etc/systemd/system/monitoring.timer
    [Unit]
    Description=monitoring timer
    After=syslog.target
    [Timer]
    OnBootSec=0sec
    OnUnitActiveSec=30sec
    [Install]
    WantedBy=timers.target.target
    EOF1_1

##### Создадим файл сервиса...
    touch /etc/systemd/system/monitoring.service
    chmod 664 /etc/systemd/system/monitoring.service

    cat <<'EOF1_2' >/etc/systemd/system/monitoring.service
    [Unit]
    Description=monitoring service
    After=syslog.target
    [Service]
    User=root
    Type=oneshot
    EnvironmentFile=/etc/sysconfig/monitoring
    ExecStart=/bin/grep $KEYWORD $LOGFILE
    [Install]
    WantedBy=timers.target.target
    EOF1_2

##### Запускаем таймер...
    systemctl daemon-reload
    systemctl enable --now monitoring.timer
##### Проверим работу таймера и сервиса...
    systemctl list-timers | grep monitoring.timer
    systemctl status monitoring.service

Задание 1 выполнено.

--------
## Задание 2.
Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл
##### Установим необходимый софт...
    yum install -y -q epel-release
    yum install -y -q spawn-fcgi php-cli httpd
##### Добавим Настройки spawn-fcgi...
    echo "OPTIONS=-u apache -g apache -s /var/run/spawn-fcgi/php-fcgi.sock -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi/spawn-fcgi.pid -- /usr/bin/php-cgi" >> /etc/sysconfig/spawn-fcgi
##### Создаем unit-файл для spawn-fcgi...
    touch /etc/systemd/system/spawn-fcgi.service
    chmod 664 /etc/systemd/system/spawn-fcgi.service

    cat <<'EOF2' >/etc/systemd/system/spawn-fcgi.service
    [Unit]
    Description=spawn-fcgi
    After=syslog.target
    [Service]
    Type=forking
    User=apache
    Group=apache
    EnvironmentFile=/etc/sysconfig/spawn-fcgi
    PIDFile=/var/run/spawn-fcgi/spawn-fcgi.pid
    RuntimeDirectory=spawn-fcgi
    ExecStart=/usr/bin/spawn-fcgi $OPTIONS
    ExecStop=
    [Install]
    WantedBy=multi-user.target
    EOF2

##### Запустим сервис spawn-fcgi.service
    systemctl daemon-reload
    systemctl enable --now spawn-fcgi.service
##### Проверим работу сервиса
    systemctl status spawn-fcgi.service

Задание 2 выполнено.

--------
## Задание 3.
Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами
##### Установим необходимый софт и настроим selinux...
    yum install -y policycoreutils-python
    semanage port -m -t http_port_t -p tcp 8081
    semanage port -m -t http_port_t -p tcp 8082
##### Скопируем и изменим файл сервиса httpd.service в шаблон
    cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
    sed -i 's*EnvironmentFile=/etc/sysconfig/httpd*EnvironmentFile=/etc/sysconfig/%i*' /etc/systemd/system/httpd@.service

##### Скопируем и изменим файлы настройки сервиса httpd.service
    cp /etc/sysconfig/httpd /etc/sysconfig/conf1
    cp /etc/sysconfig/httpd /etc/sysconfig/conf2
    sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd1.conf*' /etc/sysconfig/conf1
    sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd2.conf*' /etc/sysconfig/conf2
##### Скопируем и изменим файлы настройки демона httpd
    cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd1.conf
    cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd2.conf
    # используем разные порты
    sed -i 's/Listen 80/Listen 8081/' /etc/httpd/conf/httpd1.conf
    sed -i 's/Listen 80/Listen 8082/' /etc/httpd/conf/httpd2.conf
    # и разные pid файлы
    echo "PidFile /var/run/httpd/httpd1.pid" >> /etc/httpd/conf/httpd1.conf
    echo "PidFile /var/run/httpd/httpd2.pid" >> /etc/httpd/conf/httpd2.conf
##### Запустим два инстанса httpd
    systemctl daemon-reload
    systemctl enable --now httpd@conf1.service
    systemctl enable --now httpd@conf2.service
##### Проверим статус сервисов
    systemctl status httpd@conf1.service
    systemctl status httpd@conf2.service

Задание 3 выполнено

## Задание со звездочкой
Скачать демо-версию Atlassian Jira и переписать основной скрипт запуска на unit-файл.
##### Установим необходимый софт...
    yum install -y -q wget fontconfig
##### Загружаем Atlassian Jira...
    wget -q https://product-downloads.atlassian.com/software/jira/downloads/atlassian-jira-software-8.5.4-x64.bin
##### Запускаем unattended установку (unattended файл создан заранее)...
    chmod +x atlassian-jira-software-8.5.4-x64.bin
    ./atlassian-jira-software-8.5.4-x64.bin -q -varfile /vagrant/response.varfile
##### Создаем unit-файл для Atlassian Jira...
    touch /etc/systemd/system/jira.service
    chmod 664 /etc/systemd/system/jira.service

    cat <<'EOF4' >/etc/systemd/system/jira.service
    [Unit]
    Description=Atlassian Jira
    After=network.target
    [Service]
    Type=forking
    User=jira
    PIDFile=/opt/atlassian/jira/work/catalina.pid
    ExecStart=/opt/atlassian/jira/bin/start-jira.sh
    ExecStop=/opt/atlassian/jira/bin/stop-jira.sh
    [Install]
    WantedBy=multi-user.target
    EOF4

##### Запустим сервис...
    systemctl daemon-reload
    systemctl enable --now jira.service
##### Проверим статус сервиса
    systemctl status jira.service

Задание со * выполнено

------

Список команд для проверки:

##### Задание 1.
    systemctl list-timers | grep monitoring.timer
    systemctl status monitoring.service
##### Задание 2.
    systemctl status spawn-fcgi.service
##### Задание 3.
    systemctl status httpd@conf1.service
    systemctl status httpd@conf2.service
##### Задание 4.
    systemctl status jira.service    
## Спасибо за проверку!
