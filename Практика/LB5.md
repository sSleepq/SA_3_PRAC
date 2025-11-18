# Лабораторная работа №5  
**Тема:** Настройка DNS‑сервера на Ubuntu Server  
**Цель:** Освоить установку и конфигурацию DNS‑сервера (BIND9) для разрешения доменных имён в локальной сети.

## 1. Исходные данные

- **Сервер:** Ubuntu Server  
- **Сетевой интерфейс:** `ens34` (LAN‑сегмент)  
- **IP‑адрес сервера:** `10.10.13.1`  
- **Доменное имя зоны:** `localnet`  
- **Имя хоста сервера:** `dns-server.localnet`  
- **DNS‑серверы для рекурсии:** `8.8.8.8`, `8.8.4.4`

## 2. Подготовка сервера

### 2.1. Обновление системы
```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2. Проверка сетевого интерфейса
Убедитесь, что интерфейс `ens34` имеет статический IP:
```bash
ip a show ens34
```
Ожидаемый результат:
```
inet 10.10.13.1/24 brd 10.10.13.255 scope global ens34
```

## 3. Установка BIND9

Установите пакет DNS‑сервера:
```bash
sudo apt install bind9 bind9utils bind9-doc -y
```

Запустите сервис:
```bash
sudo systemctl start bind9
sudo systemctl enable bind9
```

## 4. Базовая конфигурация BIND9

### 4.1. Редактирование основного конфига
Откройте файл:
```bash
sudo nano /etc/bind/named.conf.options
```

Вставьте конфигурацию (замените существующее содержимое):
```conf
options {
    directory "/var/cache/bind";

    // Разрешаем запросы только из локальной сети
    allow-query { 10.10.13.0/24; localhost; };

    // Отключаем рекурсивные запросы для внешних сетей
    recursion yes;
    allow-recursion { 10.10.13.0/24; localhost; };

    // DNS‑серверы для рекурсии
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // Отключаем передачу зон (по умолчанию)
    allow-transfer { none; };

    // Прочие настройки безопасности
    dnssec-validation auto;
    auth-nxdomain no;
    listen-on-v6 { none; };
};
```

### 4.2. Создание зоны `localnet`

1. Добавьте в `/etc/bind/named.conf.local`:
   ```bash
   sudo nano /etc/bind/named.conf.local
   ```
   Вставьте:
   ```conf
   zone "localnet" {
       type master;
       file "/etc/bind/db.localnet";
   };
   ```

2. Создайте файл зоны:
   ```bash
   sudo cp /etc/bind/db.local /etc/bind/db.localnet
   sudo nano /etc/bind/db.localnet
   ```

3. Отредактируйте файл (пример):
   ```conf
   $TTL    604800
   @       IN      SOA     dns-server.localnet. admin.localnet. (
                         2025111801 ; Serial
                         604800     ; Refresh
                  86400     ; Retry
                2419200     ; Expire
                 604800 )   ; Negative Cache TTL

   @       IN      NS      dns-server.localnet.
   dns-server      IN      A       10.10.13.1
   client1         IN      A       10.10.13.10
   client2         IN      A       10.10.13.11

   ; Обратная зона (опционально)
   1.13.10.10.in-addr.arpa. IN  PTR     dns-server.localnet.
   10.13.10.10.in-addr.arpa.  IN  PTR     client1.localnet.
   11.13.10.10.in-addr.arpa.  IN  PTR     client2.localnet.
   ```

> **Примечание:**  
> - `Serial` обновите при каждом изменении зоны (формат `ГГГГММДДНН`).  
> - Добавьте записи `A` и `PTR` для ваших клиентов.

### 4.3. Настройка обратной зоны (опционально)

1. Добавьте в `/etc/bind/named.conf.local`:
   ```conf
   zone "13.10.10.in-addr.arpa" {
       type master;
       file "/etc/bind/db.10.10.13";
   };
   ```

2. Создайте файл:
   ```bash
   sudo cp /etc/bind/db.127 /etc/bind/db.10.10.13
   sudo nano /etc/bind/db.10.10.13
   ```

3. Отредактируйте (пример):
   ```conf
   $TTL    604800
   @       IN      SOA     dns-server.localnet. admin.localnet. (
                         2025111801 ; Serial
                         604800     ; Refresh
                  86400     ; Retry
                2419200     ; Expire
                 604800 )   ; Negative Cache TTL

   @       IN      NS      dns-server.localnet.
   1       IN      PTR     dns-server.localnet.
   10      IN      PTR     client1.localnet.
   11      IN      PTR     client2.localnet.
   ```

## 5. Проверка конфигурации

### 5.1. Проверка синтаксиса
```bash
sudo named-checkconf
sudo named-checkzone localnet /etc/bind/db.localnet
sudo named-checkzone 13.10.10.in-addr.arpa /etc/bind/db.10.10.13
```
Ожидайте вывод `OK`.

### 5.2. Перезагрузка BIND9
```bash
sudo systemctl reload bind9
```

## 6. Тестирование DNS‑сервера

### 6.1. Локальная проверка

1. Проверьте разрешение имени сервера:
   ```bash
   dig dns-server.localnet @127.0.0.1
   ```

2. Проверьте обратную зону:
   ```bash
   dig -x 10.10.13.1 @127.0.0.1
   ```

### 6.2. Проверка с клиента (Windows 10)

1. Настройте DNS‑сервер на клиенте:  
   - IP: `10.10.13.10` (или другой из подсети)  
   - DNS: `10.10.13.1`  

2. Выполните:
   ```cmd
   ping dns-server.localnet
   nslookup client1.localnet
   ```

### 6.3. Проверка с Linux‑клиента
```bash
nslookup dns-server.localnet 10.10.13.1
dig client1.localnet @10.10.13.1
```

## 7. Устранение неполадок

### 7.1. Типичные проблемы
- **Сервер не отвечает:** Проверьте статус сервиса:  
  ```bash
  sudo systemctl status bind9
  ```
- **Ошибка синтаксиса:** Используйте `named-checkconf` и `named-checkzone`.  
- **Брандмауэр блокирует запросы:** Откройте порт `53/udp`

### 7.2. Просмотр логов
```bash
sudo journalctl -u bind9 -f
```
Ищите ошибки типа `client ... query (cache) ... refused`.