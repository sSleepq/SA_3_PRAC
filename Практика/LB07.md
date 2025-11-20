# Лабораторная работа №7  
**Тема:** Настройка веб‑сервера Nginx на Ubuntu Server  
**Цель:** Освоить установку, базовую конфигурацию и тестирование веб‑сервера Nginx для размещения статических сайтов.

## 1. Исходные данные

- **Сервер:** Ubuntu Server  
- **Сетевой интерфейс:** `ens34` (LAN‑сегмент)  
- **IP‑адрес сервера:** `10.10.13.1`  
- **Порт HTTP:** `80`  
- **Доменное имя (для теста):** `testsite.local` (будет использоваться локально)  
- **Корневая директория сайта:** `/var/www/testsite`

## 2. Подготовка сервера

### 2.1. Обновление системы
```bash
sudo apt update
sudo apt upgrade -y
```

### 2.2. Установка Nginx
```bash
sudo apt install nginx -y
```

## 3. Базовая конфигурация Nginx

### 3.1. Проверка запуска
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```
Ожидаемый статус: `active (running)`.

### 3.2. Проверка по умолчанию
Откройте в браузере:  
```
http://10.10.13.1
```
Должна появиться стартовая страница Nginx.

## 4. Создание тестового сайта

### 4.1. Создание директории и контента
```bash
sudo mkdir -p /var/www/testsite
sudo chown -R www-data:www-data /var/www/testsite
sudo chmod -R 755 /var/www/testsite
```

Создайте индексный файл:
```bash
sudo nano /var/www/testsite/index.html
```

Вставьте содержимое:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Test Site</title>
</head>
<body>
    <h1>Добро пожаловать на тестовый сайт!</h1>
    <p>Сервер Nginx работает корректно.</p>
</body>
</html>
```

### 4.2. Создание конфигурационного файла сайта
```bash
sudo nano /etc/nginx/sites-available/testsite
```

Вставьте конфигурацию:
```nginx
server {
    listen 80;
    server_name testsite.local;

    root /var/www/testsite;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    error_log /var/log/nginx/testsite_error.log;
    access_log /var/log/nginx/testsite_access.log;
}
```

### 4.3. Активация сайта
Создайте симлинк в `sites-enabled`:
```bash
sudo ln -s /etc/nginx/sites-available/testsite /etc/nginx/sites-enabled/
```

Проверьте конфигурацию:
```bash
sudo nginx -t
```
Ожидайте вывод: `syntax is ok` и `test is successful`.

Перезапустите Nginx:
```bash
sudo systemctl reload nginx
```

### 4.4 Удалить стандарнтый сайт

```bash
sudo rm /etc/nginx/sites-enabled/default
```

## 5. Тестирование сайта

### 5.1. Через браузер
Откройте:  
```
http://testsite.local
```
Должна отобразиться страница с заголовком «Добро пожаловать на тестовый сайт!».

###5.2. Через командную строку
На клиенте выполните:
```bash
curl http://testsite.local
```
В выводе должен быть HTML‑код страницы.

### 5.3. Проверка логов
На сервере просмотрите логи:
```bash
sudo tail -f /var/log/nginx/testsite_access.log
sudo tail -f /var/log/nginx/testsite_error.log
```
При успешном запросе в `access.log` появится запись.

## 6. Дополнительные настройки (опционально)

### 6.1. Настройка HTTPS (с самоподписанным сертификатом)

1. Создайте сертификат:
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/ssl/private/nginx-selfsigned.key \
     -out /etc/ssl/certs/nginx-selfsigned.crt
   ```

2. Обновите конфигурацию сайта (`/etc/nginx/sites-available/testsite`):
   ```nginx
   server {
       listen 443 ssl;
       server_name testsite.local;

       ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
       ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

       root /var/www/testsite;
       index index.html index.htm;

       location / {
           try_files $uri $uri/ =404;
       }

       error_log /var/log/nginx/testsite_https_error.log;
       access_log /var/log/nginx/testsite_https_access.log;
   }
   ```

3. Перезагрузите Nginx:
   ```bash
   sudo systemctl reload nginx
   ```

4. Проверьте HTTPS:  
   ```
   https://testsite.local
   ```
   > В браузере появится предупреждение о ненадёжном сертификате — это ожидаемо.

### 6.2. Ограничение доступа по IP
Добавьте в блок `location /`:
```nginx
allow 10.10.13.0/24;
deny all;
```

## 7. Устранение неполадок

### 7.1. Типичные проблемы
- **Сайт не открывается:** Проверьте статус Nginx и открытые порты:  
  ```bash
  sudo systemctl status nginx
  sudo ss -tulnp | grep :80
  ```
- **Ошибка 403 (Forbidden):** Проверьте права на директорию:  
  ```bash
  ls -ld /var/www/testsite
  ```
  Должны быть права `drwxr-xr-x` и владелец `www-data`.
- **Ошибка 404:** Убедитесь, что файл `index.html` существует и указан в `index`.

### 7.2. Просмотр логов
```bash
sudo journalctl -u nginx -f
```
Ищите строки с `error` или `failed`.

## 8. Результаты работы

1. Установлен и запущен веб‑сервер Nginx.  
2. Создан тестовый сайт с корневой директорией `/var/www/testsite`.  
3. Настроена конфигурация виртуального хоста для `testsite.local`.  
4. Доступ к сайту проверен через браузер и `curl`.  
5. Настроены логи доступа и ошибок.  
6. Опционально настроен HTTPS с самоподписанным сертификатом.  
7. Проверены права доступа и сетевые настройки.


## 9. Выводы


В ходе лабораторной работы:  
- Освоена установка Nginx на Ubuntu Server.  
- Настроен виртуальный хост для размещения статического сайта.  
- Протестирован доступ через HTTP и HTTPS.  
- Изучены методы диагностики и устранения неполадок.  
- Освоены базовые принципы конфигурации Nginx (блоки `server`, `location`, логирование).  

Работа выполнена успешно.
