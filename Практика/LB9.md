# Лабораторная работа №9  
**Тема:** Настройка контроллера домена Samba DC  
**Цель:** Освоить установку и конфигурацию Samba в режиме контроллера домена (Active Directory Domain Controller, AD DC) для управления учётными записями в сети.

## 1. Исходные данные

- **Сервер:** Ubuntu Server  
- **Сетевой интерфейс:** `ens34` (LAN‑сегмент)  
- **IP‑адрес сервера:** `10.10.13.1`  
- **Имя хоста:** `dc1.prac-sa.stud`  
- **Домен:** `prac-sa.stud`  
- **Пароль администратора домена:** `AdminPass2025#`  
- **Рабочая группа:** `PRACSA`

## 2. Подготовка сервера

### 2.1. Обновление системы
```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2. Настройка имени хоста
```bash
sudo hostnamectl set-hostname dc1.prac-sa.stud
```

Проверьте:
```bash
hostname -f
```
Ожидаемый вывод: `dc1.prac-sa.stud`.

### 2.3. Редактирование `/etc/hosts`
Откройте файл:
```bash
sudo nano /etc/hosts
```
Добавьте строку:
```
10.10.13.1    dc1.prac-sa.stud dc1
```

### 2.4. Отключение systemd‑resolved (если используется)
```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
```
Создайте новый `/etc/resolv.conf`:
```bash
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

## 3. Установка Samba DC

### 3.1. Установка пакетов
```bash
sudo apt install samba samba-dsdb-modules samba-vfs-modules smbclient krb5-user -y
```

> **Примечание:** При установке `krb5-user` пропустите настройку Kerberos (выберите «Continue without configuration»).

### 3.2. Удаление стандартной конфигурации Samba
```bash
sudo rm /etc/samba/smb.conf
```

## 4. Развёртывание домена

### 4.1. Запуск `samba-tool`
```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --interactive
```

Последовательно введите:
- **Realm:** `PRAC-SA.STUD`  
- **Domain:** `prac-sa.stud`  
- **Server role:** `dc`  
- **DNS backend:** `SAMBA_INTERNAL`  
- **Administrator password:** `AdminPass2025#`  

### 4.2. Проверка конфигурации
После завершения появится путь к файлу `smb.conf`. Убедитесь, что он создан:
```bash
ls /etc/samba/smb.conf
```

## 5. Настройка Kerberos

### 5.1. Создание `/etc/krb5.conf`
```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 5.2. Тестирование аутентификации Kerberos
```bash
kinit administrator@PRAC-SA.STUD
```
Введите пароль `AdminPass2025#`.  
Проверьте билет:
```bash
klist
```

## 6. Запуск служб Samba

### 6.1. Остановка ненужных служб
```bash
sudo systemctl stop nmbd smbd
```

### 6.2. Запуск Samba AD DC
```bash
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
```

### 6.3. Проверка статуса
```bash
sudo systemctl status samba-ad-dc
```
Статус должен быть `active (running)`.

## 7. Тестирование контроллера домена

### 7.1. Проверка DNS
```bash
nslookup dc1.prac-sa.stud
```
Должен вернуться IP `10.10.13.1`.

### 7.2. Проверка LDAP
```bash
ldapsearch -H ldap://dc1.prac-sa.stud -x -D "administrator@prac-sa.stud" -W -b "dc=prac-sa,dc=stud"
```
Введите пароль `AdminPass2025#`. Должен выводиться список объектов домена.

### 7.3. Проверка Kerberos
```bash
kinit administrator@PRAC-SA.STUD
klist
```

## 8. Управление пользователями и группами

### 8.1. Создание пользователя
```bash
sudo samba-tool user create user1 --given-name="User" --surname="One" --mail-address="user1@prac-sa.stud"
```
Задайте пароль:
```bash
sudo samba-tool user setpassword user1
```

### 8.2. Добавление в группу
```bash
sudo samba-tool group addmembers "Domain Users" user1
```

### 8.3. Просмотр пользователей
```bash
sudo samba-tool user list
```

## 9. Настройка брандмауэра

### 9.1. Открытие портов
```bash
sudo ufw allow 53/tcp    # DNS
sudo ufw allow 53/udp    # DNS
sudo ufw allow 88/tcp    # Kerberos
sudo ufw allow 88/udp    # Kerberos
sudo ufw allow 135/tcp   # RPC
sudo ufw allow 137/udp   # NetBIOS
sudo ufw allow 138/udp   # NetBIOS
sudo ufw allow 139/tcp   # NetBIOS
sudo ufw allow 389/tcp   # LDAP
sudo ufw allow 445/tcp   # SMB
sudo ufw allow 464/tcp   # Kerberos password change
sudo ufw allow 464/udp   # Kerberos password change
sudo ufw allow 636/tcp   # LDAPS
sudo ufw allow 3268/tcp  # Global Catalog
```

### 9.2. Включение UFW
```bash
sudo ufw enable
```

## 10. Подключение клиента Windows к домену

1. На клиенте Windows откройте **Параметры** → **Система** → **О системе**.  
2. Нажмите **Изменить параметры домена**.  
3. Введите:  
   - **Домен:** `prac-sa.stud`  
   - **Логин:** `administrator@prac-sa.stud`  
   - **Пароль:** `AdminPass2025#`  
4. После перезагрузки войдите под учётной записью `user1`.

## 11. Устранение неполадок


### 11.1. Типичные проблемы
- **Не удаётся подключиться к DNS:** Проверьте `/etc/resolv.conf` и `nslookup`.  
- **Ошибка аутентификации Kerberos:** Убедитесь, что время синхронизировано (`sudo timedatectl set-ntp true`).  
- **Клиент не видит домен:** Проверьте порты брандмауэра и DNS‑настройки на клиенте.

### 11.2. Просмотр логов
```bash
sudo tail -f /var/log/samba/log.*
sudo journalctl -u samba-ad-dc -f
```

## 12. Результаты работы

1. Установлен и настроен Samba DC в роли контроллера домена `prac-sa.stud`.  
2. Настроены DNS, Kerberos и LDAP‑сервисы.  
3. Создан пользователь `user1` и добавлен в группу «Domain Users».  
4. Открыты необходимые порты брандмауэра.  
5. Проверено подключение клиента Windows к домену.  
6. Настроены логирование и диагностика.

## 13. Выводы


В ходе лабораторной работы:  
- Освоена установка Samba DC на Ubuntu Server.  
- Настроен домен `prac-sa.stud` с DNS и Kerberos.  
- Реализовано управление пользователями через Samba‑инструменты.  
- Протестировано подключение клиента Windows.  
- Изучены методы диагностики и устранения неполадок.  

Работа выполнена успешно.