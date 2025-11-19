## Лабораторная работа № 9  
**Тема:** Настройка контроллера домена Samba DC на сервере Ubuntu 24.04  

### Цель работы  
Настроить контроллер домена на базе Samba 4 в среде Ubuntu 24.04 с учётом существующей сетевой инфраструктуры (два интерфейса, DHCP, DNS). Обеспечить функционирование базовых служб Active Directory для управления пользователями и группами.

### Исходные данные
- **ОС:** Ubuntu 24.04 (виртуальная машина VMware).  
- **Имя хоста:** `cus`.  
- **Полное доменное имя (FQDN):** `cus.testdomen.local`.  
- **Сетевые интерфейсы:**  
  - NAT (динамический IP);  
  - LAN: `10.10.13.1/24`.  
- **Службы:**  
  - `isc-dhcp-server` (диапазон: `10.10.13.10–100`);  
  - `bind9` с зонами:  
    - прямая: `testdomen.local`;  
    - обратная: `13.10.10.in-addr.arpa`.

### Ход работы

#### 1. Предварительная настройка сети и имени хоста
1.1. Установите FQDN:  
```bash
sudo hostnamectl set-hostname cus.testdomen.local
```
1.2. Проверьте разрешение имени:  
```bash
hostname -f
```
Ожидаемый вывод: `cus.testdomen.local`.

#### 2. Подготовка DNS (bind9)
2.1. Убедитесь, что в зоне `testdomen.local` есть запись для контроллера домена:  
```zone
cus.testdomen.local. IN A 10.10.13.1
```
2.2. В обратной зоне `13.10.10.in-addr.arpa` добавьте PTR‑запись:  
```zone
1 IN PTR cus.testdomen.local.
```
2.3. Перезапустите bind9:  
```bash
sudo systemctl restart bind9
```
2.4. Проверьте разрешение имён:  
```bash
nslookup cus.testdomen.local
nslookup 10.10.13.1
```

#### 3. Установка пакетов Samba
3.1. Обновите пакеты:  
```bash
sudo apt update
```
3.2. Установите Samba и зависимости:  
```bash
sudo apt install samba winbind libpam-winbind libnss-winbind \
  libpam-krb5 krb5-config krb5-user krb5-kdc
```

#### 4. Очистка предыдущих настроек (если есть)
```bash
sudo systemctl stop winbind smbd nmbd krb5-kdc
sudo systemctl disable winbind smbd nmbd krb5-kdc
sudo rm -f /etc/samba/smb.conf /etc/krb5.conf
```

#### 5. Создание домена Samba AD DC
5.1. Запустите интерактивную настройку:  
```bash
sudo samba-tool domain provision \
  --realm=TESTDOMEN.LOCAL \
  --domain=testdomen \
  --server-role=dc \
  --dns-backend=BIND9_DLZ \
  --use-rfc2307 \
  --adminpass='StrongP@ssw0rd!'
```
**Пояснения параметров:**  
- `--realm`: область Kerberos (заглавными).  
- `--domain`: имя домена (без суффикса).  
- `--dns-backend`: использование bind9 вместо встроенного DNS.  
- `--adminpass`: пароль администратора домена.

5.2. После завершения проверьте создание файлов:  
- `/etc/samba/smb.conf`;  
- `/var/lib/samba/private/krb5.conf`.

#### 6. Настройка Kerberos
6.1. Скопируйте конфигурационный файл:  
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
6.2. Убедитесь, что в `/etc/krb5.conf` указаны:  
- `default_realm = TESTDOMEN.LOCAL`;  
- секции `[realms]` и `[domain_realm]` для `testdomen.local`.

#### 7. Интеграция Samba с bind9
7.1. Добавьте в `/etc/bind/named.conf.options` секцию для Samba:  
```conf
tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
```
7.2. Перезапустите bind9:  
```bash
sudo systemctl restart bind9
```

#### 8. Запуск служб Samba AD DC
8.1. Включите и запустите службы:  
```bash
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
```
8.2. Проверьте статус:  
```bash
sudo systemctl status samba-ad-dc
```

#### 9. Проверка работоспособности
9.1. Получите информацию о домене:  
```bash
sudo samba-tool domain info testdomen.local
```
9.2. Проверьте список пользователей:  
```bash
sudo samba-tool user list
```
9.3. Проверьте DNS‑записи через Samba:  
```bash
sudo samba-tool dns query cus.testdomen.local testdomen.local @ ALL
```

#### 10. Управление пользователями и группами
10.1. Создайте пользователя:  
```bash
sudo samba-tool user create alice \
  --given-name="Alice Smith" \
  --mail-address="alice@testdomen.local"
```
10.2. Установите пароль:  
```bash
sudo samba-tool user setpassword alice
```
10.3. Создайте группу:  
```bash
sudo samba-tool group add developers
```
10.4. Добавьте пользователя в группу:  
```bash
sudo samba-tool group addmembers developers alice
```

### Контрольные вопросы
1. Почему для контроллера домена рекомендуется использовать статический IP‑адрес?  
2. В чём отличие параметров `--realm` и `--domain` при настройке Samba AD?  
3. Зачем требуется интеграция Samba с bind9 (опция `--dns-backend=BIND9_DLZ`)?  
4. Как проверить, что Kerberos корректно настроен для работы с доменом?  
5. Какие команды используются для управления пользователями и группами в Samba AD?

### Отчёт по работе
В отчёте необходимо представить:  
1. Скриншоты этапов настройки (установка пакетов, вывод `samba-tool domain provision`, проверка DNS и Kerberos).  
2. Конфигурационные файлы:  
   - `/etc/samba/smb.conf` (ключевые секции);  
   - `/etc/krb5.conf`;  
   - фрагменты `/etc/bind/named.conf.options`.  
3. Вывод команд:  
   - `samba-tool domain info`;  
   - `samba-tool user list`;  
   - `nslookup cus.testdomen.local`.  
4. Ответы на контрольные вопросы.

### Возможные ошибки и их устранение
- **Ошибка «DNS resolution failed»**: проверьте `/etc/resolv.conf` и настройки bind9.  
- **Служба samba-ad-dc не запускается**: убедитесь, что порты 53 (DNS), 88 (Kerberos), 135–139 (SMB) не заблокированы.  
- **Не удаётся аутентифицироваться**: проверьте пароль администратора и настройки Kerberos в `/etc/krb5.conf`.  
- **Проблемы с DNS‑интеграцией**: убедитесь, что `tkey-gssapi-keytab` указан в `named.conf.options`.

### Вывод
В ходе лабораторной работы:  
- Настроен контроллер домена Samba AD DC на Ubuntu 24.04 с интеграцией в существующую инфраструктуру (DHCP, bind9).  
- Выполнена базовая конфигурация DNS и Kerberos