# Л/Р № 3

«Настройка системы мониторинга»
ЦЕЛЬ:
Получить практические
навыки по развертыванию
и настройке систем
мониторинга на базе
Prometheus и Grafana.
———————————————————

ПОЛЕЗНЫЕ ССЫЛКИ:
https://www.robustperception.io/configuring-prometheus-storage-retention/
https://laurvas.ru/prometheus-p3/#retention

ЗАДАНИЕ:

1. На машине ВМ1 (ROUTER) поднять Nginx и Prometheus. Prometheus должен
иметь циклическую запись 10-20 дней и/или ограничение по
размеру.
2. На машине ВМ2 поднять Grafana.
Все перечисленные сервисы должны управляться через systemd и
стартовать при загрузке системы.
На каждой машине настроить сбор следующих метрик.
3. Метрики ВМ 1 :
    - host metrics (метрики хоста: память, цпу, сеть, диски). В качестве
поставщика данных использовать node_exporter или telegraf.
    - Метрики status-страницы Nginx. Метрики должны собираться в
перезаписываемый файл (можно по крону). Файл должен
использоваться поставщиком данных node_exporter или telegraf
для сбора и предоставления метрик для Prometheus.
    - Размер папки Prometheus на диске. Метрики должны
собираться в перезаписываемый файл (можно по крону). Файл
должен использоваться поставщиком данных node_exporter или
telegraf для сбора и предоставления метрик для Prometheus.
Метрики ВМ 2: 
    - host metrics (метрики хоста: память, цпу, сеть, диски).
- Все собираемые метрики необходимо отобразить в Grafana в виде
графиков. Графики должны иметь названия, подписанные значения,
таблицу значений (min max current avg).
Для метрик хоста можно использовать готовые дашборды или создать
свой.
Также требуется поднять и настроить Alertmanager, который должен
отслеживать состояние Nginx (запущен/не запущен) и в случае падения
Nginx (нет процесса nginx или systemd – показывает статус Active:
inactive / dead) отсылать уведомление в Телеграм через бота (бота
зарегистрировать самостоятельно).
4. Установить Prometheus blackbox_exporter настроить дашборд 
для мониторинга любого публичного ресурса на ваш выбор.
https://github.com/prometheus/blackbox_exporter


ВНИМАНИЕ
Использовать docker допускается только для Grafana. Все остальные
сервисы должны быть установлены вручную и управляться через systemd.
При сдаче лабораторной работы будут использоваться утилиты stress (для
нагрузки на систему) и ab (для нагрузки на Nginx). Сбор индивидуальных
метрик (свои бизнес-метрики) и отрисовка в Grafana приветствуется.

# Решения

Тк будет ставить графану прям на ВМ2, то сразу сделаем роутинг, чтоб можно было с хоста зайти 
1. `sudo iptables -t nat -A PREROUTING -p tcp --dport 3000 -j DNAT --to-destination <ip-адрес второй ВМ>:3000`
2. `sudo netfilter-persistent save`


## Node_exporter

1. Установка и настройка node_exporter (для host metrics)
 смотрите версию архитектура
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-arm64.tar.gz
tar xvf node_exporter-1.6.1.linux-arm64.tar.gz
cd node_exporter-1.6.1.linux-arm64/
sudo cp node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

Создание systemd-сервиса:
sudo nano /etc/systemd/system/node_exporter.service
```bash
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter \
    --collector.textfile.directory=/var/lib/node_exporter

[Install]
WantedBy=multi-user.target
```
sudo mkdir /var/lib/node_exporter
sudo mkdir -p /opt/prometheus_scripts

sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

Проверка метрик
curl http://localhost:9100/metrics

<!-- Редактируем конфиг Prometheus (prometheus.yml):
sudo nano /etc/prometheus/prometheus.yml

Добавляем в scrape_configs:
scrape_configs:
- job_name: 'node_exporter'       # Произвольное имя задачи
static_configs:
    - targets: ['localhost:9100'] # Адрес node_exporter

Перезапускаем Prometheus:
sudo systemctl restart prometheus -->

### Метрики папки

sudo nano /opt/prometheus_scripts/prometheus_size.sh

#!/bin/bash
OUTPUT_FILE="/var/lib/node_exporter/prometheus_size.prom"
PROMETHEUS_DIR="/var/lib/prometheus"
SIZE=$(du -sb "$PROMETHEUS_DIR" | awk '{print $1}')
echo "prometheus_dir_size_bytes $SIZE" > "$OUTPUT_FILE"

sudo chmod +x /opt/prometheus_scripts/prometheus_size.sh
sudo crontab -e

Добавить туда строчку
* * * * * /opt/prometheus_scripts/prometheus_size.sh

### Метрики nginx

sudo nano /etc/nginx/conf.d/status.conf

server {
    listen 8080;
    server_name localhost;

    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}

sudo nginx -t
sudo systemctl restart nginx

sudo nano /opt/prometheus_scripts/nginx_metrics.sh

#!/bin/bash
OUTPUT_FILE="/var/lib/node_exporter/nginx_metrics.prom"
STATUS=$(curl -s http://localhost:8080/nginx_status)
ACTIVE_CONNECTIONS=$(echo "$STATUS" | awk '/Active connections/ {print $3}')
REQUESTS=$(echo "$STATUS" | awk 'NR==3 {print $3}')
echo "nginx_active_connections $ACTIVE_CONNECTIONS" > "$OUTPUT_FILE"
echo "nginx_requests_total $REQUESTS" >> "$OUTPUT_FILE"

check_nginx() {
    # 1. Проверка через systemctl
    if systemctl is-active --quiet nginx; then
        echo "nginx_systemd_active 1" >> "$OUTPUT_FILE"
    else
        echo "nginx_systemd_active 0" >> "$OUTPUT_FILE"
    fi

    # 2. Проверка процесса
    if pgrep -x "nginx" >/dev/null; then
        echo "nginx_process_running 1" >> "$OUTPUT_FILE"
    else
        echo "nginx_process_running 0" >> "$OUTPUT_FILE"
    fi

    # 3. Проверка статус-страницы
    if STATUS=$(curl -s -m 2 http://localhost:8080/nginx_status 2>/dev/null); then
        echo "nginx_status_available 1" >> "$OUTPUT_FILE"
        # Парсим дополнительные метрики
        echo "nginx_active_connections $(echo "$STATUS" | awk '/Active connections/ {print $3}')" >> "$OUTPUT_FILE"
        echo "nginx_requests_total $(echo "$STATUS" | awk 'NR==3 {print $3}')" >> "$OUTPUT_FILE"
    else
        echo "nginx_status_available 0" >> "$OUTPUT_FILE"
        echo "nginx_active_connections 0" >> "$OUTPUT_FILE"
    fi

    # Агрегированная метрика доступности
    if [[ $(grep "nginx_systemd_active 1" "$OUTPUT_FILE") && \
          $(grep "nginx_process_running 1" "$OUTPUT_FILE") && \
          $(grep "nginx_status_available 1" "$OUTPUT_FILE") ]]; then
        echo "nginx_up 1" >> "$OUTPUT_FILE"
    else
        echo "nginx_up 0" >> "$OUTPUT_FILE"
    fi
}

check_nginx

sudo chmod +x /opt/prometheus_scripts/nginx_metrics.sh
sudo crontab -e

* * * * * /opt/prometheus_scripts/nginx_metrics.sh

## Prometheus

Создание пользователя для Prometheus (опционально)
Рекомендуется запускать Prometheus под отдельным пользователем:
sudo useradd --no-create-home --shell /bin/false prometheus

Скачивание и установка Prometheus
3.1. Загрузка последней версии
смотрите на архитектуру, если мак, то надо arm
wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz

3.2. Распаковка архива
tar xvf prometheus-2.47.0.linux-amd64.tar.gz
cd prometheus-2.47.0.linux-amd64/

3.3. Копирование файлов в системные директории
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles/ console_libraries/ /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/

3.4. Настройка прав
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}


4. Настройка конфигурации
sudo nano /etc/prometheus/prometheus.yml
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9100", "<ip ВМ2 внутренней сети>:9090"]   
```

5. Создание systemd-сервиса
sudo nano /etc/systemd/system/prometheus.service
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --storage.tsdb.retention.time=1d \ # срок метрик
  --storage.tsdb.retention.size=1GB # количество метрик

Restart=always

[Install]
WantedBy=multi-user.target
```

6. Запуск Prometheus
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
статус
sudo systemctl status prometheus


7. Проверка работы
http://<ваш_IP_сервера>:9090

8. Тут же чекаем что метрики папки и nginx собираются 
prometheus_dir_size_bytes
nginx_requests_total
nginx_active_connections


## Grafana

1. Добавление репозитория Grafana
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

2. Обновление пакетов и установка
sudo apt-get update
sudo apt-get install grafana

3. Запуск и автоматический старт при загрузке
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

4. Проверка статуса
sudo systemctl status grafana-server

### Настройка datasource

5. Откройте файл конфигурации
sudo nano /etc/grafana/grafana.ini

6. Добавьте в секцию [datasources]
api_version = 1
datasources.yaml = /etc/grafana/provisioning/datasources/default.yaml

7. Создайте файл с настройками источника данных:
sudo mkdir -p /etc/grafana/provisioning/datasources
sudo nano /etc/grafana/provisioning/datasources/default.yaml

8. Вставьте конфиг для Prometheus:
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://<ip-ВМ1 внутренней сети>:9090  # Укажите IP, если Prometheus на другом сервере
    access: proxy
    isDefault: true
    version: 1
    editable: false

### Настройка dashboard

1. Создайте директорию для дашбордов:
    `sudo mkdir -p /etc/grafana/provisioning/dashboards`

    `sudo nano /etc/grafana/provisioning/dashboards/default.yaml`

2. Добавьте конфиг:
    ```
    apiVersion: 1
    providers:
    - name: 'Default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        updateIntervalSeconds: 10
        options:
        path: /var/lib/grafana/dashboards
    ```

3. Создайте директорию для дашбордов и скопируйте их:
sudo mkdir -p /var/lib/grafana/dashboards

4. Качаем dashboard для метрик системы + для папки и nginx
    - `wget https://grafana.com/api/dashboards/1860/revisions/37/download`
    - `wget https://raw.githubusercontent.com/zhuk0vskiy/bmstu-devops/refs/heads/lab_03/my_dashboard.json`
    - `sudo mv download /var/lib/grafana/dashboards/node_exporter.json`
    - `sudo mv my_dashboard.json /var/lib/grafana/dashboards/my_dashboard.json` (тут еще метрика cpu есть, но она не работает, а убирать лень (но она и не нужна по сути))
    - `sudo chown root:grafana /var/lib/grafana/dashboards/node_exporter.json`
    - `sudo chown root:grafana /var/lib/grafana/dashboards/my_dashboard.json`
    - `sudo systemctl restart grafana-server`


## Alertmanager 

смотрите на архитектуру
```
wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-arm64.tar.gz
tar xvf alertmanager-0.26.0.linux-arm64.tar.gz
cd alertmanager-0.26.0.linux-arm64

sudo cp alertmanager amtool /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false alertmanager

sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```
sudo nano /etc/systemd/system/alertmanager.service
```
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=alertmanager
Group=alertmanager
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager
Restart=always

[Install]
WantedBy=multi-user.target
```
sudo nano /etc/prometheus/prometheus.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']  # Alertmanager слушает на 9093


sudo nano /etc/prometheus/alert.rules.yml
```
groups:
- name: nginx-alerts
  rules:
  - alert: NginxDown
    expr: absent(nginx_up)
    for: 10s
    labels:
      severity: critical
    annotations:
      summary: "Nginx is DOWN on {{ $labels.instance }}"
      description: |
        Nginx failed health checks. Current metrics:
        - nginx_up: {{ $value }}
        - nginx_systemd_active: {{ query "nginx_systemd_active" | first | value }}
        - nginx_process_running: {{ query "nginx_process_running" | first | value }}
        - nginx_status_available: {{ query "nginx_status_available" | first | value }}
```
sudo nano /etc/prometheus/prometheus.yml

rule_files:
  - 'alert.rules.yml'

### Настраиваем бота

3.1. Создаем бота через BotFather

Откройте Telegram, найдите @BotFather.
Отправьте команду /newbot.
Укажите имя бота.
Получите токен (сохраните его!).

3.2. Узнаем Chat ID

Напишите боту /start.
Отправьте GET-запрос (подставьте токен):
bash
Copy
curl https://api.telegram.org/bot<ВАШ_ТОКЕН>/getUpdates
В ответе найдите chat.id (например, 123456789).

#### Возвращаемся на ВМ1

sudo nano /etc/alertmanager/alertmanager.yml

```
route:
  receiver: 'telegram'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h

receivers:
- name: 'telegram'
  telegram_configs:
  - api_url: "https://api.telegram.org"
    bot_token: "<ВАШ_ТОКЕН>"
    chat_id: <ВАШ_CHAT_ID>
    message: "{{ .CommonAnnotations.summary }}\n{{ .CommonAnnotations.description }}"
    send_resolved: true
```

```
sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
sudo systemctl status alertmanager
```
## blackbox_exporter

архитектура
```
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-arm64.tar.gz
tar xvf blackbox_exporter-*.tar.gz
cd blackbox_exporter-*/


sudo cp blackbox_exporter /usr/local/bin/
sudo mkdir /etc/blackbox_exporter
sudo cp blackbox.yml /etc/blackbox_exporter/


sudo useradd --no-create-home --shell /bin/false blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter /etc/blackbox_exporter
```

sudo nano /etc/systemd/system/blackbox_exporter.service

```
[Unit]
Description=Blackbox Exporter
After=network.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml \
  --web.listen-address=:9115
Restart=always

[Install]
WantedBy=multi-user.target
```

sudo nano /etc/prometheus/prometheus.yml
```
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - 'https://google.com'   # Пример дополнительного таргета
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # Адрес blackbox_exporter
```

```
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter
sudo systemctl enable blackbox_exporter
sudo systemctl status blackbox_exporter
```


ВМ2
4. Качаем dashboard для метрик системы + для папки и nginx
    - `wget https://grafana.com/api/dashboards/13659/revisions/1/download`
    - `sudo mv download /var/lib/grafana/dashboards/black_box.json` 
    - `sudo chown root:grafana /var/lib/grafana/dashboards/black_box.json`
    - `sudo systemctl restart grafana-server`