# CM Minecraft Ansible (RU)

Опинионный Ansible-плейбук и роль для развертывания и эксплуатации Minecraft-сервера на удалённом хосте. Поддерживает версионные, идемпотентные деплои, автоматическое принятие EULA при первом деплое конкретной версии, синхронизацию модов, запуск под systemd + screen, healthcheck и выгрузку логов при сбоях.

## Структура репозитория
```
inventory/
  hosts.ini            # Инвентарь Ansible
playbook.yml           # Входной плейбук
roles/
  minecraft/
    defaults/main.yml  # Параметры (версия, пути, память, порт)
    tasks/main.yml     # Логика версионного деплоя
    templates/
      eula.txt.j2
      systemd-minecraft.service.j2
mods/                  # Необязательно: локальные моды для синка (путь по умолчанию)
```

## Предварительные требования
- Нода управления: Ansible, Python 3, rsync
- Удалённый хост: семейство Debian/Ubuntu или RHEL/CentOS; роль установит: OpenJDK 17, screen, curl, unzip, rsync
- Сеть: откройте TCP-порт 25565 (или ваш server_port) в файрволе/SG

## Инвентарь
inventory/hosts.ini
```
[minecraft]
178.72.129.136 ansible_user=root
```
Добавляйте хосты или настраивайте host_vars/group_vars (рекомендуется SSH-ключ).

## Быстрый старт
- Сложите ваши моды (если есть) в ./mods
- При необходимости отредактируйте vars в playbook.yml (версия/пути/память/порт)
- Запустите: `ansible-playbook -i inventory/hosts.ini playbook.yml`

## Переменные (defaults) – roles/minecraft/defaults/main.yml
- minecraft_version: "1.20.1"
- server_dir: "/opt/minecraft"
- versions_dir: "{{ server_dir }}/versions"
- version_dir: "{{ versions_dir }}/{{ minecraft_version }}"
- current_link: "{{ server_dir }}/current"
- service_name: minecraft
- server_port: 25565
- xms: 1G
- xmx: 2G
- mods_src: "{{ lookup('env','MC_MODS') | default('mods', true) }}"  # Локальный путь, который синкается на удалённый

Fallback через переменные окружения (ниже по приоритету, чем vars плейбука/инвентаря):
- MC_VERSION, MC_SERVER_DIR, MC_MODS

## Как это работает (идемпотентность и версии)
- Директория на версию: {{server_dir}}/versions/{{minecraft_version}}
- Симлинк: {{server_dir}}/current указывает на активную версию
- Первый деплой версии:
  - Скачивает Mojang server.jar нужной версии, пишет eula.txt, синкает моды, создаёт/обновляет сервис systemd, переводит симлинк current, запускает и делает healthcheck
- Повторный запуск с той же версией:
  - Только синкает моды в {{server_dir}}/current/mods и перезапускает сервис, если моды изменились; шаги EULA/сервиса/скачивания пропускаются
- Смена версии:
  - Останавливает сервис, готовит новую директорию версии, выполняет полный цикл деплоя, переключает симлинк, запускает и проверяет
- Healthcheck: ожидает открытие порта, затем ищет в latest.log строки Exception/ERROR/FATAL; при сбое падает плей и вытягивается лог в ./fetched_logs/latest.log

## Типовые операции
Смена версии Minecraft
- Измените minecraft_version в playbook.yml (предпочтительно) или передайте `-e minecraft_version=1.20.4`, либо экспортируйте `MC_VERSION=1.20.4`
- Запустите плейбук; роль остановит сервис, задеплоит новую версию и переключит симлинк

Добавление/обновление модов
- Положите .jar файлы в локальную папку модов (по умолчанию: ./mods или переопределите mods_src)
- Запустите плейбук; моды синхронизируются в {{server_dir}}/current/mods и сервис перезапустится только при изменениях

Изменение памяти
- Отредактируйте xms/xmx в playbook.yml или задайте `-e xms=2G -e xmx=4G`; перезапустите плейбук (сервис перезапустится)

Изменение путей установки
- Отредактируйте server_dir (и при необходимости service_name) в playbook.yml; запустите плейбук

Изменение порта сервера
- Отредактируйте server_port и откройте порт в файрволе; запустите плейбук

Старт/стоп/рестарт вручную на удалённом хосте
- `systemctl status minecraft`
- `systemctl stop minecraft`
- `systemctl start minecraft`
- `journalctl -u minecraft -e`

Логи
- Логи сервера: {{server_dir}}/current/logs/latest.log
- При неуспехе деплоя/рестарта роль забирает лог в ./fetched_logs/latest.log на ноду управления

Переопределение переменных на запуске
- `ansible-playbook -i inventory/hosts.ini playbook.yml -e minecraft_version=1.20.4 -e xmx=4G`

Использование env (минимальный приоритет)
- `export MC_VERSION=1.20.4` `export MC_MODS=/abs/путь/к/mods` затем запускайте плейбук

## Советы
- Держите локальную папку модов под Git, чтобы отслеживать изменения набора
- Тестируйте апгрейды версии на стейджинге; миры/моды могут быть несовместимы
- Делайте бэкап мира (директория world/ под current) перед крупными апгрейдами
- Если используете Forge/Fabric, убедитесь, что используете соответствующий loader; по умолчанию эта роль разворачивает ванильный Mojang server и не ставит модлоадеры

## Устранение проблем
- Плей упал и вытянул лог:
  - Посмотрите ./fetched_logs/latest.log и {{server_dir}}/current/logs/latest.log на удалённом
- Сервис не стартует:
  - `journalctl -u minecraft -e`; проверьте версию Java (17) и лимиты памяти
- Моды не синкнулись:
  - Убедитесь, что rsync установлен на ноде управления; проверьте корректность пути mods_src
- Порт недоступен извне:
  - Разрешите server_port в файрволе/SG

## Продвинутое
Несколько серверов
- Используйте несколько хостов в инвентаре; переопределяйте service_name, server_port, server_dir в host_vars

Разные моды по версиям
- Указывайте versioned-путь mods_src при смене версий, чтобы разделять наборы модов

## Команды
- Запуск: `ansible-playbook -i inventory/hosts.ini playbook.yml`
- Ограничить на один хост: `-l 178.72.129.136`
- Check mode: `--check`
- Подробно: `-vvv`

## Примечания
- Для современных версий (1.18+) требуется Java 17
- Screen используется под systemd для отстыковки консоли; имя сервиса по умолчанию — "minecraft"
