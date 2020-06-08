# -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :lesson7 => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.11.107',
  },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "1804.02"
    MACHINES.each do |boxname, boxconfig|
        config.vbguest.no_install = true
        config.vm.define boxname do |box|

            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s

            #box.vm.network "forwarded_port", guest: 8080, host: 8080

            box.vm.network "private_network", ip: boxconfig[:ip_addr]

            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "1024"]
            #        vb.gui = true
            end

        box.vm.provision "shell", inline: <<-SHELL

          echo -e "\nЗадание 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова"
          echo "Файл лога и ключевое слово должны задаваться в /etc/sysconfig."
          echo "Создадим файл настроек /etc/sysconfig/monitoring..."
          touch /etc/sysconfig/monitoring
          echo -e "LOGFILE=/var/log/messages\nKEYWORD=systemd" >> /etc/sysconfig/monitoring

          echo "Создадим файл таймера..."
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
          echo -e "\nСоздадим файл сервиса..."
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

          echo "Запускаем таймер..."
          systemctl daemon-reload
          systemctl enable --now monitoring.timer
          echo -e "\nПроверим работу таймера и сервиса..."
          systemctl list-timers | grep monitoring.timer
          systemctl status monitoring.service
          echo "\nЗадание 1 выполнено.\n\n"
          sleep 10
          echo -e "\nЗадание 2. Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл"
          echo "Установим необходимый софт..."
          yum install -y -q epel-release > /dev/null 2>&1
          yum install -y -q spawn-fcgi php-cli httpd > /dev/null 2>&1
          echo "Добавим Настройки spawn-fcgi..."
          echo "OPTIONS=-u apache -g apache -s /var/run/spawn-fcgi/php-fcgi.sock -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi/spawn-fcgi.pid -- /usr/bin/php-cgi" >> /etc/sysconfig/spawn-fcgi
          echo "Создаем unit-файл для spawn-fcgi..."
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
          systemctl daemon-reload
          systemctl enable --now spawn-fcgi.service
          systemctl status spawn-fcgi.service
          echo -e "\nЗадание 2 выполнено\n\n"
          sleep 10
          echo -e "\nЗадание 3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами"
          echo "Установим необходимый софт и настроим selinux..."
          yum install -y policycoreutils-python
          semanage port -m -t http_port_t -p tcp 8081
          semanage port -m -t http_port_t -p tcp 8082
          echo "Скопируем и изменим файл сервиса httpd.service в шаблон"
          cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
          sed -i 's*EnvironmentFile=/etc/sysconfig/httpd*EnvironmentFile=/etc/sysconfig/%i*' /etc/systemd/system/httpd@.service

          echo "Скопируем и изменим файлы настройки сервиса httpd.service"
          cp /etc/sysconfig/httpd /etc/sysconfig/conf1
          cp /etc/sysconfig/httpd /etc/sysconfig/conf2
          sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd1.conf*' /etc/sysconfig/conf1
          sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd2.conf*' /etc/sysconfig/conf2
          echo "Скопируем и изменим файлы настройки демона httpd"
          cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd1.conf
          cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd2.conf
          sed -i 's/Listen 80/Listen 8081/' /etc/httpd/conf/httpd1.conf
          sed -i 's/Listen 80/Listen 8082/' /etc/httpd/conf/httpd2.conf
          echo "PidFile /var/run/httpd/httpd1.pid" >> /etc/httpd/conf/httpd1.conf
          echo "PidFile /var/run/httpd/httpd2.pid" >> /etc/httpd/conf/httpd2.conf

          systemctl daemon-reload
          systemctl enable --now httpd@conf1.service
          systemctl enable --now httpd@conf2.service
          systemctl status httpd@conf1.service
          systemctl status httpd@conf2.service
          echo -e "\nЗадание 3 выполнено\n\n"
          sleep 10

          echo -e "\nЗадание со звездочкой"
          echo -e "Скачать демо-версию Atlassian Jira и переписать основной скрипт запуска на unit-файл.\n"
          echo "Установим необходимый софт..."
          yum install -y -q wget fontconfig > /dev/null 2>&1
          echo "Загружаем Atlassian Jira..."
          wget -q https://product-downloads.atlassian.com/software/jira/downloads/atlassian-jira-software-8.5.4-x64.bin
          echo "Запускаем unattended установку..."
          chmod +x atlassian-jira-software-8.5.4-x64.bin
          ./atlassian-jira-software-8.5.4-x64.bin -q -varfile /vagrant/response.varfile
          echo "Создаем unit-файл для Atlassian Jira..."
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
          systemctl daemon-reload
          systemctl enable --now jira.service
          systemctl status jira.service
          echo -e "\nЗадание со * выполнено"
          echo -e "\nСпасибо за проверку!"

          SHELL

        end
    end
  end
