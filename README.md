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

Тк будем ставить графану прям на ВМ2, то сразу сделаем роутинг, чтоб можно было с хоста зайти 
1. Подключаемся к ВМ2 (рекомендую делать это с хоста по ssh)
1. `sudo iptables -t nat -A PREROUTING -p tcp --dport 3000 -j DNAT --to-destination <ip-адрес второй ВМ внутренней>:3000`
2. `sudo netfilter-persistent save`


## Node_exporter

*Делаем эту часть на ВМ1 и ВМ2*

- Установка и настройка node_exporter (для host metrics)

  Смотрите версию архитектура (если хост не мак, то меняйте на amd)

  1. `wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-arm64.tar.gz`
  2. `tar xvf node_exporter-1.6.1.linux-arm64.tar.gz`
  3. `cd node_exporter-1.6.1.linux-arm64/`
  4. `sudo cp node_exporter /usr/local/bin/`
  5. `sudo useradd --no-create-home --shell /bin/false node_exporter`
  6. `sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter`

- Создание systemd-сервиса:
  1. `sudo nano /etc/systemd/system/node_exporter.service`

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

  2. `sudo mkdir /var/lib/node_exporter`
  3. `sudo mkdir -p /opt/prometheus_scripts`

- Запуск 

  1. `sudo systemctl daemon-reload`
  2. `sudo systemctl start node_exporter`
  3. `sudo systemctl enable node_exporter`

- Проверка метрик
  1. `curl http://localhost:9100/metrics`

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

*Делаем эту часть на ВМ1*

- Создаем скрипт для складывания метрик

  1. `sudo nano /opt/prometheus_scripts/prometheus_size.sh`

      ```bash
      #!/bin/bash
      OUTPUT_FILE="/var/lib/node_exporter/prometheus_size.prom"
      PROMETHEUS_DIR="/var/lib/prometheus"
      SIZE=$(du -sb "$PROMETHEUS_DIR" | awk '{print $1}')
      echo "prometheus_dir_size_bytes $SIZE" > "$OUTPUT_FILE"
      ```


  2. `sudo chmod +x /opt/prometheus_scripts/prometheus_size.sh`
  3. `sudo crontab -e`

      Добавить туда строчку
      `* * * * * /opt/prometheus_scripts/prometheus_size.sh`

### Метрики nginx

*Делаем эту часть на ВМ1*

- Добавляем отдельный ендпоинт для проверки статуса

1.  `sudo nano /etc/nginx/conf.d/status.conf`
    ```
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
    ```

2. `sudo nginx -t`
3. `sudo systemctl restart nginx`

- Создам скрипт для складывания метрик

1. `sudo nano /opt/prometheus_scripts/nginx_metrics.sh`
    ```bash
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
    ```


2. `sudo chmod +x /opt/prometheus_scripts/nginx_metrics.sh`
3. `sudo crontab -e`

4. `* * * * * /opt/prometheus_scripts/nginx_metrics.sh`

## Prometheus

*Делаем эту часть на ВМ1*

- Создание пользователя для Prometheus (опционально)
Рекомендуется запускать Prometheus под отдельным пользователем:

    1. `sudo useradd --no-create-home --shell /bin/false prometheus`

- Скачивание и установка Prometheus

  смотрите на архитектуру(если мак, то надо arm)

  1. `wget https://github.com/prometheus/prometheus/releases/download/v2.47.0/prometheus-2.47.0.linux-amd64.tar.gz`


  2. `tar xvf prometheus-2.47.0.linux-amd64.tar.gz`
  3. `cd prometheus-2.47.0.linux-amd64/`


  4. `sudo mkdir -p /etc/prometheus`
  5. `sudo mkdir -p /var/lib/prometheus`
  6. `sudo cp prometheus promtool /usr/local/bin/`
  7. `sudo cp -r consoles/ console_libraries/ /etc/prometheus/`
  8. `sudo cp prometheus.yml /etc/prometheus/`


  9. `sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus`
  10. `sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}`


- Настройка конфигурации
  1. `sudo nano /etc/prometheus/prometheus.yml`

      ```
      global:
        scrape_interval: 15s
        evaluation_interval: 15s

      scrape_configs:
        - job_name: "prometheus"
          static_configs:
            - targets: ["localhost:9090"]
        - job_name: "node_exporter"
          static_configs:
            - targets: ["localhost:9100", "<ip ВМ2 внутренней сети>:9100"]   
      ```


  2. `sudo nano /etc/systemd/system/prometheus.service`
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

- Запуск Prometheus
  1. `sudo systemctl daemon-reload`
  2. `sudo systemctl start prometheus`
  3. `sudo systemctl enable prometheus`
  4. `sudo systemctl status prometheus`


- Проверка работы
  1. с хоста в браузере `http://<ваш_IP_сервера>:9090`

2. Тут же чекаем что метрики папки и nginx собираются 
  `prometheus_dir_size_bytes`
  `nginx_requests_total`
  `nginx_active_connections`


## Grafana

*Делаем эту часть на ВМ2*

- установка Grafana
  1. `sudo apt-get install -y apt-transport-https`
  2. `sudo apt-get install -y software-properties-common wget`
  3. `wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -`
  4. `echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list`
  5. `sudo apt-get update`
  6. `sudo apt-get install grafana`

- Запуск и автоматический старт при загрузке
  1. `sudo systemctl daemon-reload`
  2. `sudo systemctl start grafana-server`
  3. `sudo systemctl enable grafana-server`
  4. `sudo systemctl status grafana-server`

### Настройка datasource

- Откройте файл конфигурации
1. `sudo nano /etc/grafana/grafana.ini`

    Добавьте в секцию [datasources]
    ```
    api_version = 1
    datasources.yaml = /etc/grafana/provisioning/datasources/default.yaml
    ```


2. `sudo mkdir -p /etc/grafana/provisioning/datasources`
3. `sudo nano /etc/grafana/provisioning/datasources/default.yaml`
    ```
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://<ip-ВМ1 внутренней сети>:9090  # Укажите IP, если Prometheus на другом сервере
        access: proxy
        isDefault: true
        version: 1
        editable: false
    ```

### Настройка dashboard

- Создайте директорию для дашбордов:
  1. `sudo mkdir -p /etc/grafana/provisioning/dashboards`

  2. `sudo nano /etc/grafana/provisioning/dashboards/default.yaml`

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

  3. `sudo mkdir -p /var/lib/grafana/dashboards`

- Качаем dashboard для метрик системы + для папки и nginx
    1. `wget https://grafana.com/api/dashboards/1860/revisions/37/download`
    2. `wget https://raw.githubusercontent.com/zhuk0vskiy/bmstu-devops/refs/heads/lab_03/my_dashboard.json`
    3. `sudo mv download /var/lib/grafana/dashboards/node_exporter.json`
    4. `sudo mv my_dashboard.json /var/lib/grafana/dashboards/my_dashboard.json` (тут еще метрика cpu есть, но она не работает, а убирать лень (но она и не нужна по сути))
    5. `sudo chown root:grafana /var/lib/grafana/dashboards/node_exporter.json`
    6. `sudo chown root:grafana /var/lib/grafana/dashboards/my_dashboard.json`
    7. `sudo systemctl restart grafana-server`
    8. `wget https://grafana.com/api/dashboards/13659/revisions/1/download`
    9. `sudo mv download /var/lib/grafana/dashboards/black_box.json` 
    10. `sudo chown root:grafana /var/lib/grafana/dashboards/black_box.json`
    11. `sudo systemctl restart grafana-server`


## Alertmanager 

*Делаем эту часть на ВМ1*

- смотрите на архитектуру (если не мак, то смените на amd)

  1. `wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-arm64.tar.gz`
  2. `tar xvf alertmanager-0.26.0.linux-arm64.tar.gz`
  3. `cd alertmanager-0.26.0.linux-arm64`

  4. `sudo cp alertmanager amtool /usr/local/bin/`

  5. `sudo useradd --no-create-home --shell /bin/false alertmanager`

  6. `sudo mkdir -p /etc/alertmanager /var/lib/alertmanager`
  7. `sudo chown alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager`

  8. `sudo nano /etc/systemd/system/alertmanager.service`
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

  9. `sudo nano /etc/prometheus/prometheus.yml`
      ```
      alerting:
        alertmanagers:
          - static_configs:
              - targets: ['localhost:9093']  # Alertmanager слушает на 9093
      ```

  10. `sudo nano /etc/prometheus/alert.rules.yml`
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
  11. `sudo nano /etc/prometheus/prometheus.yml`
  добавляем
      ```
      rule_files:
        - 'alert.rules.yml'
      ```

12. `sudo nano /etc/alertmanager/alertmanager.yml`

    Как получить токен написано ниже
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


13. `sudo systemctl daemon-reload`
14. `sudo systemctl start alertmanager`
15. `sudo systemctl enable alertmanager`
16. `sudo systemctl status alertmanager`

### Настраиваем бота

*Делаем эту часть в Телеге*

1. Откройте Telegram, найдите @BotFather.
2. Отправьте команду /newbot.
3. Укажите имя бота.
4. Получите токен (сохраните его!).

5. Напишите боту /start.
6. Отправьте GET-запрос c хоста (подставьте токен):

    `curl https://api.telegram.org/bot<ВАШ_ТОКЕН>/getUpdates`

    В ответе найдите chat.id (например, 123456789).

### Теперь стопните nginx и чекните что через минуту пришло уведовление в боте


## blackbox_exporter

*Делаем эту часть на ВМ1*

- Опять смотрим на архитектуру (amd / arm)

1. `wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-arm64.tar.gz`
2. `tar xvf blackbox_exporter-*.tar.gz`
3. `cd blackbox_exporter-*/`
4. `sudo cp blackbox_exporter /usr/local/bin/`
5. `sudo mkdir /etc/blackbox_exporter`
6. `sudo cp blackbox.yml /etc/blackbox_exporter/`
7. `sudo useradd --no-create-home --shell /bin/false blackbox_exporter`
8. `sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter /etc/blackbox_exporter`


9. `sudo nano /etc/systemd/system/blackbox_exporter.service`

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

10. `sudo nano /etc/prometheus/prometheus.yml`

    Добавляем 
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


11. `sudo systemctl daemon-reload`
12. `sudo systemctl start blackbox_exporter`
13. `sudo systemctl enable blackbox_exporter`
14. `sudo systemctl status blackbox_exporter`



