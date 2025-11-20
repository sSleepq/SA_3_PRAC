# Лабораторная работа №6  
**Тема:** Настройка службы каталога Samba на Ubuntu Server  
**Цель:** Освоить установку и конфигурацию Samba для организации общего доступа к файлам и принтерам в смешанной сети (Linux + Windows).

## 1. Исходные данные

- **Сервер:** Ubuntu Server  
- **Сетевой интерфейс:** `ens34` (LAN‑сегмент)  
- **IP‑адрес сервера:** `10.10.13.1`  
- **Имя сервера Samba:** `UBUNTU-SERVER`  
- **Рабочая группа:** `WORKGROUP` (по умолчанию)  
- **Пользователи:**  
  - `userx` (пароль: `Pass123!`)   
- **Общие папки:**  
  - `/srv/samba/public` (без аутентификации)  
  - `/srv/samba/private` (доступ только для `userx`)

## 2. Подготовка сервера

### 2.1. Обновление системы
```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2. Установка Samba
```bash
sudo apt install samba -y
```

## 3. Базовая конфигурация Samba

### 3.1. Создание общих папок

1. Создайте директории:
   ```bash
   sudo mkdir -p /srv/samba/{public,private}
   ```

2. Настройте права:
   ```bash
   sudo chmod -R 0775 /srv/samba/public
   sudo chmod -R 0770 /srv/samba/private
   sudo chown -R nobody:nogroup /srv/samba/public
   sudo chown -R root:sambashare /srv/samba/private
   ```

3. Создайте группу для приватного доступа:
   ```bash
   sudo groupadd sambashare
   ```

### 3.2. Добавление пользователей Samba

1. Создайте системных пользователей:
   ```bash
   sudo useradd -m -s /bin/bash userx
   ```

2. Задайте пароли:
   ```bash
   echo "userx:Pass123!" | sudo chpasswd
   ```

3. Добавьте в группу `sambashare`:
   ```bash
   sudo usermod -aG sambashare userx
   ```

4. Создайте пользователей Samba и задайте пароли:
   ```bash
   sudo smbpasswd -a userx
   ```
   > Введите пароли `Pass123!` соответственно.

## 4. Настройка общих ресурсов

Добавьте в конец файла `/etc/samba/smb.conf`:

```ini
[public]
   path = /srv/samba/public
   read only = no
   guest ok = yes
   force user = nobody
   force group = nogroup
   create mask = 0775
   directory mask = 0775

[private]
   path = /srv/samba/private
   read only = no
   valid users = user1, user2
   force user = root
   force group = sambashare
   create mask = 0770
   directory mask = 0770
```

## 5. Запуск и проверка сервиса

### 5.1. Проверка конфигурации
```bash
testparm
```
Ожидайте сообщение `Loaded services file OK`.

### 5.2. Перезагрузка Samba
```bash
sudo systemctl restart smbd
sudo systemctl enable smbd
```

### 5.3. Проверка статуса
```bash
sudo systemctl status smbd
```
Статус должен быть `active (running)`.

## 6. Тестирование доступа

### 6.1. С Windows‑клиента

1. Откройте проводник и введите:  
   ```
   \\10.10.13.1
   ```

2. Проверьте доступ:  
   - К папке `public` — без авторизации.  
   - К папке `private` — с логином `userx`/`Pass123!`.

3. Попробуйте создать файл в обеих папках.

### 6.2. С Linux‑клиента

1. Установите клиент Samba:  
   ```bash
   sudo apt install cifs-utils -y
   ```

2. Подключите общую папку:  
   ```bash
   # Публичная папка
   sudo mount -t cifs //10.10.13.1/public /mnt -o guest

   # Приватная папка
   sudo mount -t cifs //10.10.13.1/private /mnt -o username=user1,password=Pass123!
   ```

3. Проверьте содержимое:  
   ```bash
   ls /mnt
   ```

## 7. Устранение неполадок

### 7.1. Типичные проблемы
- **Нет доступа к папкам:** Проверьте права на директории и настройки `smb.conf`.  
- **Ошибка аутентификации:** Убедитесь, что пользователи добавлены через `smbpasswd`.  
- **Брандмауэр блокирует:** Откройте порты:  
  ```bash
  sudo ufw allow 137/udp
  sudo ufw allow 138/udp
  sudo ufw allow 139/tcp
  sudo ufw allow 445/tcp
  ```

### 7.2. Просмотр логов
```bash
sudo tail -f /var/log/samba/*.log
```
Ищите ошибки типа `permission denied` или `authentication failure`.

## 8. Результаты работы

1. Установлен и настроен Samba‑сервер.  
2. Созданы общие папки:  
   - `public` — доступен без авторизации.  
   - `private` — доступ только для `user1` и `user2`.  
3. Пользователи Samba добавлены с паролями.  
4. Доступ проверен с Windows и Linux‑клиентов.  
5. Порты брандмауэра открыты для Samba.  
6. Логирование включено для диагностики.

## 9. Выводы

В ходе лабораторной работы:  
- Освоена установка и базовая конфигурация Samba.  
- Настроены общие папки с разным уровнем доступа.  
- Добавлены пользователи и управлялись права.  
