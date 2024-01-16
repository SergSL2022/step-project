# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.provision "shell", inline: "echo Hello"

# =============================================== VM1 =====================================

  config.vm.define "VM1" do |vm1|
    vm1.vm.box = "bento/ubuntu-22.04"
    vm1.vm.hostname = "VM1"
    vm1.vm.network "public_network", ip: "192.168.1.144"
    vm1.vm.provider "virtualbox" do |v|
      v.memory = "1024"
      v.cpus = 1
    end

    vm1.vm.provision "shell", inline: <<-SHELL
  echo hello VM1
  apt-get update

  # Встановлення MYSQL server
  apt-get install -y vim mysql-server
  sudo service mysql start

  # Створення користувача MYSQL для mysql_exporter і налаштвання прав доступу
  groupadd --system mysqld_exporter
  useradd -s /bin/false -r -g mysqld_exporter mysqld_exporter

  mysql -u root -e "CREATE USER 'mysqld_exporter'@'localhost' IDENTIFIED BY 'devops';"
  mysql -u root -e "GRANT ALL PRIVILEGES ON *.* TO 'mysqld_exporter'@'localhost';"
  mysql -u root -e "FLUSH PRIVILEGES;"
  mysql -u root -e "quit;"

  sudo cp /etc/mysql/my.cnf /etc/my.cnf
  sudo nano /etc/my.cnf
  cat <<EOL > /etc/my.cnf
[client]
user=mysqld_exporter
password=devops
EOL
  sudo chown root:mysqld_exporter /etc/my.cnf
  sudo systemctl restart mysql.service

  # Створення бази данних Shop
  mysql -umysqld_exporter -p'devops' -e "CREATE DATABASE IF NOT EXISTS Shop;"

  # Встановлення Prometheus MySQL exporter
  apt-get install -y vim
  wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz
  tar xvf mysqld_exporter-0.14.0.linux-amd64.tar.gz
  cp mysqld_exporter-0.14.0.linux-amd64/mysqld_exporter /usr/local/bin/
  rm -rf mysqld_exporter-0.14.0.linux-amd64.tar.gz mysqld_exporter-0.14.0.linux-amd64

  cat <<EOL > /etc/systemd/system/mysqld_exporter.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/mysqld_exporter --config.my-cnf=/etc/my.cnf
User=mysqld_exporter
Group=mysqld_exporter
Type=simple
Restart=always

[Install]
WantedBy=multi-user.target
EOL

  systemctl daemon-reload
  systemctl start mysqld_exporter
  systemctl enable mysqld_exporter

  # Встановлення Prometheus Node Exporter
  wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
  tar xvf node_exporter-1.2.2.linux-amd64.tar.gz
  cp node_exporter-1.2.2.linux-amd64/node_exporter /usr/local/bin/
  rm -rf node_exporter-1.2.2.linux-amd64.tar.gz node_exporter-1.2.2.linux-amd64

  cat <<EOL > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/usr/local/bin/node_exporter
User=root
Restart=always

[Install]
WantedBy=default.target
EOL

  systemctl daemon-reload
  systemctl start node_exporter
  systemctl enable node_exporter
SHELL
  end

# =============================================== VM2 =====================================

config.vm.define "VM2" do |vm2|
    vm2.vm.box = "bento/ubuntu-22.04"
    vm2.vm.hostname = "VM2"
    vm2.vm.network "public_network", ip: "192.168.1.155"
    vm2.vm.provider "virtualbox" do |v|
      v.memory = "2048"
      v.cpus = 2
    end

    vm2.vm.provision "shell", inline: <<-SHELL
      echo hello VM2
      apt-get update

    #Встановлення і налаштування Alertmanager
      cd ~/
      wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-amd64.tar.gz
      tar xvzf alertmanager-0.26.0.linux-amd64.tar.gz
      sudo mv alertmanager-0.26.0.linux-amd64/alertmanager /usr/local/bin/
      sudo mkdir /etc/alertmanager/
      sudo mv alertmanager-0.26.0.linux-amd64/amtool /etc/alertmanager
      rm alertmanager-0.26.0.linux-amd64.tar.gz

      cat <<EOL > /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 1m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 30s
  receiver: 'gmail-notifications'

receivers:
- name: 'gmail-notifications'
  email_configs:
  - to: 'test2024sss@gmail.com'
    from: 'test2024sss@gmail.com'
    smarthost: smtp.gmail.com:587
    auth_username: 'test2024sss@gmail.com'
    auth_identity: 'test2024sss@gmail.com'
    auth_password: 'nrtn zjvx cfov tsqn'
    send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance' ]
EOL

      cat <<EOL > /etc/systemd/system/alertmanager.service
[Unit]
Description=AlertManager Server Service
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/alertmanager --config.file /etc/alertmanager/alertmanager.yml --web.external-url=http://192.168.1.155:9093
User=root
Group=root
Type=simple

[Install]
WantedBy=multi-user.target
EOL

      sudo systemctl daemon-reload
      sudo systemctl start alertmanager
      sudo systemctl enable alertmanager

    #Встановлення і налаштування Prometheus
      sudo apt-get install -y prometheus
      sudo cp /vagrant/prometheus.yml /etc/prometheus/prometheus.yml

      cat <<EOL > /etc/prometheus/alert_rules.yml
groups:
- name: VM1_CPU_alerts
  rules:
  - alert: HighCpuUsage
    expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", job="vm1_node_exporter"}[5m])) * 100) > 10
    for: 5s
    labels:
      severity: "critical"
    annotations:
      summary: "High CPU "
      description: "CPU usage is above 10% on instance VM1"
EOL

      cat <<EOL > /etc/prometheus/prometheus.yml
# Sample config for Prometheus.

global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'example'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['192.168.1.155:9093']

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "alert_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'vm1_node_exporter'
    static_configs:
      - targets: ['192.168.1.144:9100']

  - job_name: 'vm1_mysql_exporter'
    static_configs:
      - targets: ['192.168.1.144:9104']
EOL
      sudo systemctl daemon-reload
      sudo systemctl enable prometheus
      sudo systemctl restart prometheus

  #Встановлення Grafana-server
      apt-get install -y vim adduser libfontconfig1
      wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
      sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
      sudo apt-get update
      sudo apt-get install -y grafana
      sudo systemctl start grafana-server
      sudo systemctl enable grafana-server
    SHELL
  end
end