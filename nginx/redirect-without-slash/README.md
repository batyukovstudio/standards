Версия: 1.0

# Инструкция по настройке редиректов без конечного слеша в Nginx

## Назначение конфигурационных файлов

### 1. `nginx/snippets/rewrite-without-slash.conf`
Содержит правила, которые делают **редирект со страницы без конечного слеша** на версию с `/` (например, с `/catalog` на `/catalog/`), **если это не файл и нет query-параметров**.

### 2. `nginx/conf.d/redirect-map-without-slash.conf`
Содержит логические проверки, на основании которых определяется, требуется ли делать редирект. Использует директиву `map`, чтобы:
- Проверить, заканчивается ли URI на слэш
- Проверить, является ли URI ссылкой на статический файл
- Проверить наличие query-параметров
- Объединить условия в одну переменную `$do_without_slash_redirect`

Файл должен быть подключён в `nginx.conf` внутри блока `http`, чтобы карты стали глобально доступными:
```nginx
include /etc/nginx/conf.d/redirect-map-without-slash.conf;
```

## Что именно сделано

### ✅ `redirect-map-without-slash.conf`

Этот файл должен быть подключён в `nginx.conf`, внутри блока `http`:

```nginx
include /etc/nginx/conf.d/*.conf;
```

Содержит 4 карты:

```nginx
map $request_uri $needs_slash_redirect {
    default 0;
    ~^(.+[^/])$ 1; # если путь не заканчивается на /
}

map $request_uri $is_static_file {
    default 0;
    ~*\.((js|css|svg|webmanifest|png|jpe?g|gif|ico|woff2?|ttf|eot|otf|mp4|webp|json|xml|txt|html?))$ 1;
}

map $args $has_query_string {
    ""      0;
    default 1;
}

map "$needs_slash_redirect:$is_static_file:$has_query_string" $do_without_slash_redirect {
    default 0;
    "1:0:0" 1;
}
```

📌 **Назначение**:
- Сначала проверяется, что URL не заканчивается на `/`
- Потом — что это не файл (по расширению)
- Потом — что нет GET-параметров

Если всё совпадает, переменная `$do_without_slash_redirect` становится равной `1`, и это значит — **нужно сделать редирект на версию с `/`**.

---

### ✅ `rewrite-without-slash.conf`

Этот файл подключается внутрь `server {}` блока, например:

```nginx
include snippets/rewrite-without-slash.conf;
```

Внутри:

```nginx
if ($do_without_slash_redirect = 1) {
    return 301 $request_uri/;
}
```

📌 **Назначение**:
- Делает редирект (301 Permanent Redirect) на URL с `/`, **если переменная говорит, что это нужно**
- Безопасно использовать `if` в таком контексте (официально разрешено Nginx)

---

## Финальный пример `server`-блока

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    root /var/www/project/public;

    include snippets/redirect-map-without-slash.conf;
    include snippets/rewrite-without-slash.conf;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    ...
}
```

---

Эта структура конфигурации помогает:

- Избежать лишних `if`-ов
- Делать редиректы только при необходимости
- Сохранять читаемость и масштабируемость конфигурации
