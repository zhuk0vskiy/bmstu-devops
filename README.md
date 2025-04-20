# Л/Р №4

«Система управления конфигурациями»

ЦЕЛЬ:
Получить практические
навыки по использованию
системы управления
конфигурациями Ansible.

ЗАДАНИЕ:
1. Создать две локальных ВМ без графического интерфейса.
2. Установить сервер Ansible на ВМ1.
3. Подключить ВМ1 и ВМ2 к управлению через сервер Ansible.
Проверить доступность хостов (ansible -m ping all).
4. Научиться устанавливать пакеты на управляемые ВМ с помощью
Ansible.
5. Выбрать список из не менее трех любых пакетов и настроить
установку этого списка пакетов отдельно на ВМ1, отдельно на ВМ2 и на
группу хостов, состоящую из ВМ1 и ВМ2. 
6. Настроить установку любой бд. (запуск через systemd, состояние запущена и включена в автозагрузку)
7. Настроить установку любого web-сервера  (запуск через systemd, состояние запущена и включена в автозагрузку)
8. Выбрать любой готовый плейбук из репозитория Ansible Galaxy и
продемонстрировать работу с ним. Уметь рассказать возможности
данного плейбука.

ВНИМАНИЕ
Управление конфигурацией должно осуществляться на ВМ2 только
средствами Ansible сервера на ВМ1 (открываете консоль на ВМ1 →
вводите команды Ansible → результат смотрим на ВМ2).
Сервисы из пунктов 6,7 должены устанавливаться через роль, стартовать после отработки роли.
Также установленные сервисы должны стартовать через автозагрузку systemd.
Особое внимание уделить решению проблемы повышения прав. Список
основных директив Ansible для работы с правами:
- become – указывает на необходимость эскалации привилегий;
- become_method – метод эскалации привилегий;
- become_user — пользователь под которым мы заходим с помощью
become_method;

Примеры
- Управление nginx (уметь управлять конфигурацией nginx);
- управление БД Postgresql (создание бд, создание пользователей,
создание дампа) и/или
- управление любым другим инструментом, который вы
используете в рабочих/учебных проектах.

# Решение

1. Качаем ansible
    ```
    sudo apt update
    sudo apt install -y ansible
    ansible --version
    ```
2. Прокидываем ssh ключи, чтоб вм могли взаимодействовать
    ```
    ssh-keygen
    ssh-copy-id юзер_на_вм2@IP_ВМ2
    ```

2. Прописываем вм для работы

- `sudo nano /etc/ansible/hosts`

  ```
  [web]  # Группа web (ВМ1 и ВМ2)
  vm1 ansible_host=IP_ВМ1 ansible_connection=local
  vm2 ansible_host=IP_ВМ2 ansible_user=admin ansible_ssh_private_key_file=~/.ssh/id_ed25519

  [db]   # Группа db (только ВМ2)
  vm2 ansible_host=IP_ВМ2

  [all:vars]  # Общие переменные
  ansible_user=<юзер на вм2>  # Пользователь для подключения
  ansible_ssh_private_key_file=~/.ssh/id_ed25519  # Путь к SSH-ключу

  [all:children]
  web
  db
  
  ```
- `ansible all -m ping`

4. Выбрать список из не менее трех любых пакетов и настроить
установку этого списка пакетов отдельно на ВМ1, отдельно на ВМ2 и на группу хостов, состоящую из ВМ1 и ВМ2
- `cd ~`
- `mkdir ansible` (вот тут уже можете копировать папку прям из репо. Сделать это можно через midnight commander)
- `cd ansible`
- `nano install_common.yml`

  ```
  ---
  - name: Install common packages on all VMs
    hosts: all
    become: yes
    tasks:
      - name: Install common packages
        apt:
          name: "{{ common_packages }}"
          state: present
        vars:
          common_packages:
            - git       # система контроля версий

  - name: Install packages only on VM1
    hosts: web
    become: yes
    tasks:
      - name: Install web packages
        apt:
          name: "{{ vm1_packages }}"
          state: present
        vars:
          vm1_packages:
            - python3-dev # инструменты для разработки на Python

  - name: Install packages only on VM2
    hosts: db
    become: yes
    tasks:
      - name: Install db packages
        apt:
          name: "{{ vm2_packages }}"
          state: present
        vars:
          vm2_packages:
            - net-tools  # сетевые утилиты

  ```

5. Настроить установку любой бд. (запуск через systemd, состояние запущена и включена в автозагрузку)

- `nano db_setup.yml`
  ```
  ---
  - name: Deploy and configure PostgreSQL
    hosts: db
    become: yes
    roles:
      - role: postgresql
        vars:
          postgresql_databases:
            - name: app_db
              encoding: UTF-8
              lc_collate: en_US.UTF-8
              lc_ctype: en_US.UTF-8
            - name: test_db
              encoding: UTF-8
              lc_collate: C
              lc_ctype: C
  ```
- `mkdir roles/{defaults,tasks}`
- `nano roles/defaults/main.yml`
  ```
  ---
  # Версия PostgreSQL
  postgresql_version: 14

  # Пакеты для установки
  postgresql_packages:
    - "postgresql-{{ postgresql_version }}"
    - "postgresql-client-{{ postgresql_version }}"
    - "postgresql-contrib-{{ postgresql_version }}"
    - libpq-dev
    - python3-psycopg2

  # Настройки сервиса
  postgresql_service: postgresql
  postgresql_service_state: started
  postgresql_service_enabled: true

  # Настройки базы данных
  postgresql_databases:
    - name: ansible_db
      encoding: UTF-8
      lc_collate: en_US.UTF-8
      lc_ctype: en_US.UTF-8

  ```
- `nano roles/tasks/main.yml`
  ```
  ---
  - name: Install PostgreSQL packages
    apt:
      name: "{{ postgresql_packages }}"
      state: present
      update_cache: yes

  - name: Ensure PostgreSQL service is running and enabled
    systemd:
      name: "{{ postgresql_service }}"
      state: "{{ postgresql_service_state }}"
      enabled: "{{ postgresql_service_enabled }}"
    notify: restart postgresql

  - name: Create PostgreSQL databases
    become: yes
    become_user: postgres
    postgresql_db:
      name: "{{ item.name }}"
      encoding: "{{ item.encoding }}"
      lc_collate: "{{ item.lc_collate }}"
      lc_ctype: "{{ item.lc_ctype }}"
    loop: "{{ postgresql_databases }}"
    when: postgresql_databases is defined
  ```
6. Настроить установку любого web-сервера  (запуск через systemd, состояние запущена и включена в автозагрузку). Вот тут я лоханулся и сделал через плейбук, хотя надо было через роль.

- `nano web_setup.yml`

  ```
  ---
  - name: Install and configure Nginx
    hosts: web  # или конкретный хост для веб-сервера
    become: yes
    tasks:
      - name: Install Nginx
        apt:
          name: nginx
          state: present
          update_cache: yes

      - name: Ensure Nginx service is running and enabled
        systemd:
          name: nginx
          state: started
          enabled: yes

      - name: Configure basic index page
        copy:
          content: |
            <!DOCTYPE html>
            <html>
            <head>
                <title>Ansible-managed Nginx</title>
            </head>
            <body>
                <h1>Welcome to {{ ansible_hostname }}</h1>
                <p>Managed by Ansible</p>
            </body>
            </html>
          dest: /var/www/html/index.html
          mode: 0644

      - name: Allow HTTP traffic in firewall
        ufw:
          rule: allow
          port: 80
          proto: tcp

      - name: Verify Nginx installation
        uri:
          url: "http://{{ ansible_host }}"
          return_content: yes
        register: nginx_result
        until: nginx_result.status == 200
        retries: 5
        delay: 3

      - name: Show access info
        debug:
          msg: "Nginx is running. Access at: http://{{ ansible_host }}"
  ```

7. Выбрать любой готовый плейбук из репозитория Ansible Galaxy и
продемонстрировать работу с ним. Уметь рассказать возможности
данного плейбука.
- `ansible-galaxy install geerlingguy.docker`
- `nano install_docker.yml`
  ```
  - name: Install and configure Docker with log limits
    hosts: web
    become: yes
    vars:
      # Основные переменные для роли geerlingguy.docker
      docker_users:
        - "{{ ansible_user }}"
      
      docker_install_compose: true
      docker_compose_version: "1.29.2"
      
      # Конфигурация демона Docker
      docker_configure_daemon: true
      docker_daemon_config:
        log-driver: "json-file"
        log-opts:
          max-size: "10m"    # Максимальный размер файла лога
          max-file: "3"      # Максимальное количество файлов логов
        storage-driver: "overlay2"
        default-ulimits:
          nofile:
            soft: 65536
            hard: 65536

    roles:
      - role: geerlingguy.docker

    tasks:
      - name: Verify Docker log configuration
        command: docker info --format '{{ "{{.LoggingDriver}}" }}'
        register: docker_log_driver
        changed_when: false

      - name: Verify Docker log options
        command: docker info --format '{{ "{{json .LoggingConfig}}" }}'
        register: docker_log_config
        changed_when: false
  ```
  при запуске этого файла должен появиться /etc/docker/daemon.json в котором настройки логов будут указаны

Ну и командой ниже запускаем исполнение

`ansible-playbook имя_файла --ask-become-pass`

вообще я даже не запускал жти файлы, ибо половина из того что там указано уже было установлено на вм. Я бы на вашем месте тоже бы не запускал) только если попросят

