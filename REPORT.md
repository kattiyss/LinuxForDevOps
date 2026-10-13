  # Отчёт по домашней работе

## Проблема 1. Неверный ExecStart.

### Что было обнаружено

После `sudo ./scripts/setup.sh` сервис homework-app.service находился в состоянии activating (auto-restart). В логе:

```text
    Process: 8927 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)
    Main PID: 8927 (code=exited, status=203/EXEC)
```
status=203/EXEC - код, который systemd возвращает, когда не может выполнить ExecStart


### Как проводилась диагностика

```bash
systemctl status homework-app.service
```

Вывод показал status=203/EXEC (не удалось выполнить исполняемый файл).

```bash
sudo ls -la /opt/linux-devops-homework/
```

Содержимое каталога:

    drwxr-xr-x 2 root root 4096 ... .
    drwxr-xr-x 3 root root 4096 ... ..
    -rwxr-xr-x 1 root root 1304 ... server.py

Файла app.py в каталоге нет, есть только server.py. Значит, ExecStart ссылался на несуществующий путь

Дополнительно, чтобы понять целевое состояние:

```bash
grep -n "ExecStart\|python\|server.py" scripts/check.sh
```

Вывод:

```text
    12:execstart="$(systemctl show homework-app.service -p ExecStart --value 2>/dev/null || true)"
    13:if grep -q '/opt/linux-devops-homework/app/server.py' <<<"$execstart"; then
    74:if grep -q '^ExecStart=/opt/linux-devops-homework/app/server.py$' "$repo_root/systemd/homework-app.service"; then
```

То есть check.sh ожидает путь /opt/linux-devops-homework/app/server.py.


### В чём была причина

В unit-файле ExecStart указывал на несуществующий файл /opt/linux-devops-homework/app.py. Кроме того, setup.sh (строка 81) устанавливает файл приложения в /opt/linux-devops-homework/server.py, а целевое (эталонное) расположение — /opt/linux-devops-homework/app/server.py


### Что было изменено

Создан целевой подкаталог app/ и скопирован туда файл приложения:

```bash
sudo mkdir -p /opt/linux-devops-homework/app
sudo cp -a /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
sudo chown -R root:root /opt/linux-devops-homework/app
sudo chmod 0755 /opt/linux-devops-homework/app
sudo chmod 0755 /opt/linux-devops-homework/app/server.py
```

В файле systemd/homework-app.service строка ExecStart приведена к целевому виду:
    ExecStart=/opt/linux-devops-homework/app/server.py

Затем

```bash
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

check.sh — источник истины по целевому состоянию. Он проверяет две вещи: 
1. Через systemctl show -p ExecStart, что системный unit содержит подстроку /opt/linux-devops-homework/app/server.py.
2. Что в репозиторном unit есть строка ровно ^ExecStart=/opt/linux-devops-homework/app/server.py$. 

Значит, нужно привести раскладку файлов к /opt/linux-devops-homework/app/server.py, и указать этот путь в ExecStart. 


### Как проверялся результат

```bash
sudo ls -la /opt/linux-devops-homework/app/
systemctl cat homework-app.service
sudo ./scripts/check.sh
```

check.sh показывает [PASS] systemd ExecStart points to the application и [PASS] fixed systemd unit is saved in the repository


---

## Проблема 2. Права на /var/lib/linux-devops-homework.

### Что было обнаружено

После исправления ExecStart сервис всё ещё падал с status=1/FAILURE. В логе (journalctl -u homework-app.service) виден Python-traceback:

```text
    File "/opt/linux-devops-homework/server.py", line 18, in <module>
        with (STATE_DIR / 'startup.log').open('a', encoding='utf-8') as f:
    PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
```

### Как проводилась диагностика

```bash
journalctl -u homework-app.service -n 50 --no-pager
sudo ls -la /var/lib/linux-devops-homework/
sudo stat /var/lib/linux-devops-homework/
id homework
getent passwd homework
getent group homework
```
Фактическое состояние каталога:

```text
    Access: (0700/drwx------)  Uid: (    0/    root)   Gid: (    0/    root)

Пользователь homework существует и является системным:
    uid=997(homework) gid=973(homework) groups=973(homework)
    homework:x:997:973::/var/lib/linux-devops-homework:/usr/sbin/nologin
    homework:x:973:
```

### В чём была причина

Приложение работает от пользователя homework (в unit-файле User=homework), а каталог /var/lib/linux-devops-homework/ принадлежал root:root с правами 0700. У пользователя homework не было ни доступа к каталогу, ни права на запись в него, поэтому open('.../startup.log', 'a') завершался EACCES (Errno 13 Permission denied)


### Что было изменено

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
```


### Почему было выбрано это решение

3ий пункт из README требует именно homework:homework и 0750. Владелец-пользователь homework получает rwx и может создавать/писать startup.log, группа получает r-x, остальным доступ закрыт


### Как проверялся результат

```bash
sudo stat /var/lib/linux-devops-homework/
sudo ls -la /var/lib/linux-devops-homework/
systemctl status homework-app.service
```

Вывод:

```text
    Access: (0750/drwxr-x---)  Uid: (  997/homework)   Gid: (  973/homework)

    drwxr-x--- 2 homework homework ... .
    -rw-r--r-- 1 homework homework 27 ... startup.log
```

Сервис перешёл в состояние active (running).


---

## Проблема 3. Привязка к 127.0.0.1 вместо 0.0.0.0.

### Что было обнаружено

Сервис перешёл в active (running), но в логе:
    homework-app: listening on 127.0.0.1:8080

По пункту 6 README приложение должно слушать 0.0.0.0:8080.


### Как проводилась диагностика

```bash
sudo ss -tlnp | grep 8080
sudo cat /etc/linux-devops-homework/app.conf
cat config/app.conf
```

Вывод:

```text
    LISTEN 0  5  127.0.0.1:8080  0.0.0.0:*  users:(("python3",pid=27727,fd=3))

    [server]
    host = 127.0.0.1
    port = 8080
```

И в системном, и в репозиторном конфиге прописан host = 127.0.0.1.


### В чём была причина

В конфиге стояло host = 127.0.0.1. Команда ss -tlnp подтвердила, что сокет привязан именно к этому адресу: LISTEN ... 127.0.0.1:8080. 127.0.0.1 — это loopback-интерфейс lo. Значит, сокет слушает только локальные подключения. README (6 пункт) требует привязки ко всем IPv4-интерфейсам, то есть на 0.0.0.0. Поэтому заменила 127.0.0.1 на 0.0.0.0.


### Что было изменено

В файле config/app.conf (в репозитории) и в его системной копии /etc/linux-devops-homework/app.conf строка

    host = 127.0.0.1 заменена на host = 0.0.0.0

Затем

```bash
sudo cp config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```
После правки репозиторного файла сервис всё ещё работал со старым значением, потому что конфиг читается процессом один раз при старте. Поэтому новый конфиг был скопирован в /etc/linux-devops-homework/app.conf, и сервис перезапущен, чтобы новый процесс прочитал уже host = 0.0.0.0.


### Почему было выбрано это решение

0.0.0.0 это wildcard-адрес: сокет принимает соединения на всех IPv4-интерфейсах. Это как раз то состояние, которого требует README. Правки внесены в репозиторий и скопированы в /etc, чтобы сервис их подхватил.


### Как проверялся результат

```bash
sudo ss -tlnp | grep 8080
curl -i http://127.0.0.1:8080/health
```

Вывод:

```text
    LISTEN 0  5  0.0.0.0:8080  0.0.0.0:*  users:(("python3",pid=28009,fd=3))

    HTTP/1.0 200 OK
    Server: BaseHTTP/0.6 Python/3.14.4
    Content-Type: application/json
    Content-Length: 16

    {"status": "ok"}
```

---

## Дополнительные наблюдения

### Процесс сервиса

Узнала PID главного процесса сервиса и посмотрела, от какого пользователя он работает:

```bash
MAIN_PID=$(systemctl show -p MainPID --value homework-app.service)
ps -o pid,ppid,user,group,cmd -p "$MAIN_PID"
grep -E '^(Name|Uid|Gid|PPid):' /proc/"$MAIN_PID"/status
tr '\0' ' ' < /proc/"$MAIN_PID"/cmdline; echo
```

Вывод:

```text
        PID    PPID USER     GROUP    CMD
      28614       1 homework homework python3 /opt/linux-devops-homework/app/s

    Name:	python3
    PPid:	1
    Uid:	997	997	997	997
    Gid:	973	973	973	973

    python3 /opt/linux-devops-homework/app/server.py
```

Процесс работает от пользователя homework (UID 997), не от root, как и нужно по требованиям. PPID = 1 (systemd) является прямым потомком PID 1, что ожидаемо для сервисов, запущенных через systemd.

### Открытые файлы процесса
Посмотрела, какие файловые дескрипторы держит процесс и каков его рабочий каталог:

```bash
sudo ls -l /proc/"$MAIN_PID"/fd/
sudo ls -l /proc/"$MAIN_PID"/cwd
```

Фактический вывод:

```text
    lr-x------ 1 homework homework 64 ... 0 -> /dev/null
    lrwx------+1 homework homework 64 ... 1 -> 'socket:[143267]'
    lrwx------+1 homework homework 64 ... 2 -> 'socket:[143267]'
    lrwx------+1 homework homework 64 ... 3 -> 'socket:[143269]'

    lrwxrwxrwx 1 homework homework 0 ... /proc/28614/cwd -> /
```
Среди файловых дескрипторов видны:
0 (/dev/null) - стандартный ввод
1, 2 (socket 143267) - стандартные потоки вывода и ошибок, направленные в журнал (journald), так как сервис запущен systemd
3 (socket 143269) - слушающий TCP-сокет 0.0.0.0:8080


## Итоговый ход диагностики

Кратко опишите весь путь от исходного нерабочего состояния до исправленного сервиса: в каком порядке проявлялись проблемы и как одна проверка приводила к следующей.

1. Первая команда systemctl status homework-app.service показала 203/EXEC, значит systemd не может выполнить ExecStart.
2. Проверка ls -la /opt/linux-devops-homework/ показала, что файла app.py нет, есть server.py. А grep по check.sh выявил целевой путь /opt/linux-devops-homework/app/server.py. Создала подкаталог app/, скопировала туда файл и привела ExecStart к целевому виду.
3. Второй запуск дал Python-traceback с PermissionError на /var/lib/linux-devops-homework/startup.log. Причиной были root:root 0700 на каталоге. Исправила на homework:homework 0750.
4. Сервис поднялся, но слушал 127.0.0.1:8080, когда в конфиге стоял host = 127.0.0.1. Заменила на 0.0.0.0.
5. По итогу sudo ./scripts/check.sh показал 10/10.

## Итоговая проверка

Вставьте полный вывод успешного запуска:

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

    [PASS] systemd ExecStart points to the application
    [PASS] service runs as homework:homework
    [PASS] state directory ownership and permissions are correct
    [PASS] service is enabled
    [PASS] service is active
    [PASS] running process has the expected UID
    [PASS] TCP/8080 listens on 0.0.0.0
    [PASS] GET /health returns the expected response
    [PASS] fixed systemd unit is saved in the repository
    [PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
