# Инструкция по настройке базовой авторизации в Nginx

## Назначение файлов

### 1. `nginx/snippets/base-auth.conf`
Этот файл используется для настройки **базовой авторизации** в Nginx. Он включает в себя настройку аутентификации с использованием `.htpasswd` файла.

### 2. Настройки в `nginx.conf`
Для активации базовой авторизации нужно добавить следующие строки в соответствующий `server`-блок:

```nginx
auth_basic "Administrator’s Area";                    # Сообщение, которое будет показываться пользователю при запросе авторизации.
auth_basic_user_file /etc/nginx/.htpasswd;           # Путь к файлу, в котором хранится пара логин:пароль.
```

## Что именно сделано

### ✅ `base-auth.conf`

Этот файл содержит необходимые директивы для настройки базовой авторизации в Nginx:

```nginx
# Включение базовой аутентификации для всего сайта
auth_basic "Administrator’s Area";            # Всплывающее сообщение, требующее ввода логина и пароля.
auth_basic_user_file /etc/nginx/.htpasswd;   # Путь к файлу, где хранятся логины и пароли пользователей.
```

- **`auth_basic`**: Эта директива задает текст, который будет отображаться пользователю, когда ему нужно будет ввести логин и пароль для доступа к ресурсу.
- **`auth_basic_user_file`**: Эта директива указывает путь к файлу `.htpasswd`, в котором содержатся логины и пароли пользователей, которым разрешен доступ.

### ✅ Включение в конфигурацию

Чтобы включить авторизацию на сайте, нужно подключить файл `base-auth.conf` внутри блока `server` в конфигурации Nginx:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    root /var/www/project/public;

    # Включение базовой авторизации
    include snippets/base-auth.conf;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    ...
}
```

### ✅ Что необходимо сделать для работы:

1. Установите утилиту `apache2-utils` командой:
   ```bash
   sudo apt-get install apache2-utils
   ```
2. Создайте файл `.htpasswd` с помощью команды:
   ```bash
   sudo htpasswd -c /etc/nginx/.htpasswd username
   ```
   В процессе создания файла вам будет предложено ввести и подтвердить пароль для пользователя `username`. Для добавления других пользователей, используйте команду без флага `-c`:
   ```bash
   htpasswd /etc/nginx/.htpasswd anotheruser
   ```

3. Убедитесь, что Nginx может читать файл `.htpasswd`, для этого файл должен иметь права доступа, например:
   ```bash
   chmod 644 /etc/nginx/.htpasswd
   ```

4. Перезагрузите Nginx, чтобы изменения вступили в силу:
   ```bash
   sudo systemctl reload nginx
   ```

Теперь при обращении к защищенному ресурсу, Nginx будет запрашивать логин и пароль, используя данные из файла `.htpasswd`.

---

Эта настройка позволяет быстро реализовать базовую авторизацию для доступа к частным зонам на сайте, защищая административные или чувствительные разделы.
