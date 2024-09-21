---
title: Бакет как Volume в Doсker контейнере
draft: false
tags:
  - Docker
  - YandexCloud
---
#### Бакет как Volume
Монтированный бакет Yandex Cloud ObjectStorage можно использовать как Volume Docker контейнера. 
Бакет указывается как обычная директория в Docker compose:

```docker-compose
volumes:
  - ./logs:/app/logs
  - ./media/dev:/app/media
```

Для доступа к файлам бакета из контейнера необходимо в конфигурации 
[[Монтирование бакета на ВМ Compute Cloud#^f3a272 |демона, монтирующего бакет]] задать режим кэширования файлов `writes` для доступа к имеющимся в бакете файлам:
```bash
--vfs-cache-mode writes
```
 * In this mode all reads and writes are buffered to and from disk. When data is read from the remote this is buffered to disk as well.
 А также разрешить доступ root пользователю к монтированной директории.
```bash
--allow-root
```
* Необходимо для обхода [[Монтирование бакета на ВМ Compute Cloud#^384334|ошибки запуска]] Docker контейнера с монтированным бакетом в качестве volume.
#### Порядок запуска

Запущенный Docker контейнер может не иметь доступа к файлам бакета, если бакет был смонтирован после запуска контейнера. 

Для решения этой проблемы необходимо настроить корректный порядок запуска:
1. Монтирование бакета.
2. Запуск Docker контейнера.

Т.к. монтирование бакета выполняется с помощью демона пользователя, в конфигурации демона можно задать шаги для остановки `docker-service` и его запуска уже после монтирования директории.

В секцию `[Service]` демона добавьте команды для остановки `docker-service` перед запуском `rclone` и его запуска после завершения:

```bash
[Service]
ExecStartPre=-/usr/bin/sudo /bin/systemctl stop docker
ExecStartPost=/usr/bin/sudo /bin/systemctl start docker
```

- `ExecStartPre` остановит `docker` перед запуском `rclone`.
- `ExecStartPost` запустит `docker` после успешного старта `rclone`.

Обновите конфигурацию демонов пользователя:
```bash
systemctl --user daemon-reload
```

Разрешите пользователю управлять `docker` через `sudo` без пароля:

* Откройте файл sudoers:    
```bash
sudo visudo
```
- Добавьте строку:
```bash
<user_name> ALL=(ALL) NOPASSWD: /bin/systemctl stop docker, /bin/systemctl start docker
```
- Перезапустите `rclone<%i>.service`:
```bash
systemctl --user restart rclone@<%i>.service
```
----
📂 [[Yandex Cloud]]

Последнее изменение: 21.09.2024 20:08